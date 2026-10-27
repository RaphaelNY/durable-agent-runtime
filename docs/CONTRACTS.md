# 契约（P0 首先要定的东西）

这份文档是设计工作纸，不是最终规范。**在开始写 engine 之前，把这里的开放问题逐个回答并记成 ADR**，否则事件 schema 会在实现中反复返工。

## 1. 事件信封

```rust
struct EventEnvelope {
    event_id: EventId,      // 全局唯一；建议 ULID 或 (run_id, seq) 的稳定哈希
    run_id: RunId,
    seq: u64,               // 每个 run 内单调递增，从 0 开始
    ts_ms: u64,             // 事件产生时间；测试用注入时钟
    schema_version: u16,    // 从 1 开始
    kind: EventKind,
}
```

`seq` 是重放顺序的唯一依据；`ts_ms` 只用于展示和定时器，不参与排序。

## 2. 事件类型（草案）

```rust
enum EventKind {
    RunStarted {
        workflow: String,
        input_digest: Digest,
    },
    ActivityScheduled {
        step_id: StepId,
        name: String,
        params_digest: Digest,
        idempotency_key: String,
        timeout_ms: u64,
        max_attempts: u32,
    },
    ActivityCompleted {
        step_id: StepId,
        attempt: u32,
        result_digest: Digest,
        result_ref: Option<BlobRef>,
    },
    ActivityFailed {
        step_id: StepId,
        attempt: u32,
        error_class: String,     // 稳定的分类，不是错误字符串
        retry_at_ms: Option<u64>,
    },
    TimerSet {
        timer_id: TimerId,
        fire_at_ms: u64,
    },
    TimerFired {
        timer_id: TimerId,
    },
    RunCompleted {
        result_digest: Digest,
    },
    RunFailed {
        error_class: String,
    },
}
```

约定：

- 日志里存 digest，不存明文参数/结果；大 payload 放 blob store，小 payload 可以内联（见开放问题 1）。
- `error_class` 是稳定枚举/短字符串，便于统计和重试策略；不要存完整错误栈。
- 新增字段必须向后兼容；删除/改语义必须升 `schema_version`。

## 3. Journal 记录分帧

候选方案：

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| JSONL + 每行 CRC32C | 可直接 `tail -f`，调试友好 | 需要自己保证一行一条，半行靠解析失败检测 |
| 二进制帧 `[len][crc32c][payload]` | 截断/撕裂检测直接，性能好 | 不可读，需要工具 |
| 长度前缀 + JSON + CRC | 兼顾 | 实现稍多 |

建议 MVP：**JSONL + 每行 `crc32c` 字段**。规则：

- 每行是一个完整 `EventEnvelope` 的 JSON，外加 `crc32c`（对规范化后的其余字段计算）。
- 读取时：解析失败或 CRC 不匹配 → 视为尾部损坏，截断到上一条完整记录并记录告警。
- 不允许空行；末尾必须有换行。
- 单条记录大小上限（例如 64 KiB），超过走 blob。

## 4. 状态机契约

```rust
trait Workflow {
    type State: Snapshot;

    /// 重放时按 seq 顺序调用；返回该事件之后应发出的命令。
    fn on_event(&mut self, event: &EventEnvelope) -> Result<Vec<Command>, WorkflowError>;

    fn state_hash(&self) -> Digest;
}

enum Command {
    ScheduleActivity { step_id: StepId, name: String, params: Vec<u8>, timeout_ms: u64, max_attempts: u32 },
    SetTimer { timer_id: TimerId, fire_at_ms: u64 },
    CompleteRun { result: Vec<u8> },
    FailRun { error_class: String },
}
```

确定性规则（写进 review checklist）：

- 工作流代码里不允许读时钟、随机数、环境变量、文件、网络。
- 定时器只能通过 `SetTimer` 命令和 `TimerFired` 事件推进。
- 活动结果只能来自 `ActivityCompleted` 事件，不能直接调用外部系统。
- 重放时，`on_event` 返回的命令序列必须与历史记录一致；不一致就是非确定性，必须报错而不是继续。

## 5. Activity 契约

```rust
trait Activity {
    type Input: DeserializeOwned;
    type Output: Serialize;

    fn execute(&self, ctx: &ActivityContext, input: Self::Input) -> Result<Self::Output>;
}
```

语义：

- **至少一次**：崩溃恢复后，未记录完成的活动会被重新调度，可能执行两次。
- **幂等键**：`idempotency_key = hash(run_id, step_id)`，重试复用同一个键；外部系统用它去重。
- **超时**：由执行器按 `timeout_ms` 强制；超时记为 `ActivityFailed`，是否重试由策略决定。
- **取消**：尽力而为；不保证副作用已停止。
- **结果**：完成后写 `ActivityCompleted`，结果存 digest + blob ref；重放时直接复用，不再执行。

不要承诺"exactly-once 副作用"。正确表述是：**至多一次重放复用 + 至少一次执行 + 幂等键**。

## 6. 索引契约

```rust
struct Episode {
    episode_id: EpisodeId,   // 稳定 ID：hash(run_id, seq) 或 hash(内容)
    run_id: RunId,
    event_id: EventId,
    ts_ms: u64,
    text: String,
    metadata: BTreeMap<String, String>,
}

trait VectorIndex {
    fn upsert(&mut self, episode: &Episode, vector: &[f32]) -> Result<()>;
    fn search(&self, query: &[f32], k: usize, filter: &Filter) -> Result<Vec<Hit>>;
    fn snapshot(&self) -> Result<IndexSnapshot>;
    fn restore(&mut self, snapshot: IndexSnapshot) -> Result<()>;
}

struct Checkpoint {
    last_offset: u64,        // 已消费到的 journal 偏移（或事件序号）
    snapshot_id: u64,
    schema_version: u16,
    checksum: Digest,
}
```

索引规则：

- **幂等 upsert**：同一个 `episode_id` 重复写入不产生重复条目；内容变化则覆盖。
- **checkpoint 原子写**：临时文件 → fsync → rename；包含 offset、snapshot id、校验和。
- **watermark**：查询返回已消费到的位置，调用方据此判断新鲜度。
- **可重建**：删除索引目录后，从 journal（或 episode store）重新消费即可。
- **不阻塞写**：indexer 落后只影响查询新鲜度，不影响 engine。

## 7. 恢复语义（必须逐条测试）

| 场景 | 期望 |
| --- | --- |
| journal 尾部半条 | 截断到上一条完整记录；该步骤未完成 |
| 写 journal 前崩溃 | 事件不存在；run 从上一状态继续 |
| 写 journal 后、活动执行前崩溃 | 重放后重新调度活动 |
| 活动执行后、写完成事件前崩溃 | 重新调度；靠幂等键防止重复副作用 |
| 写完成事件后、fsync 前崩溃 | 视为未完成；同上 |
| 索引 upsert 后、checkpoint 前崩溃 | 从旧 checkpoint 重放；幂等 upsert |
| 索引快照损坏 | 从上一个有效快照 / journal 重建 |
| journal 已截断、索引 offset 过期 | 依赖稳定 episode id + episode store（见问题 3） |

## 8. 必须回答的开放问题

| # | 问题 | 建议默认 | 影响 |
| --- | --- | --- | --- |
| 1 | 参数/结果是内联还是 blob？ | 小 payload（≤64KiB）内联，大 payload 走内容寻址 blob | journal 大小、恢复速度 |
| 2 | 分帧用 JSONL+CRC 还是二进制？ | MVP 用 JSONL+CRC；保留切换二进制的接口 | 调试性、性能 |
| 3 | 日志是否截断？索引 offset 怎么对齐？ | MVP 不截断；索引存稳定 episode id + episode store，后续再加保留策略 | 最容易埋雷 |
| 4 | episode 的稳定 ID 是什么？ | `hash(run_id, seq)`；同一事件重复投影得到同一 ID | 幂等、去重 |
| 5 | embedding 从哪来？ | 测试用确定性 hash embedding（64 维）；生产可替换 provider | 可测性、外部依赖 |
| 6 | 查询一致性？ | 最终一致 + watermark；提供 `wait_for_watermark` | 读己之写 |
| 7 | 删除怎么处理？ | HNSW tombstone + 周期性重建 | 召回率、内存 |
| 8 | 快照如何原子？ | 临时文件 + fsync + rename + 校验和 | 崩溃恢复 |
| 9 | 并发模型？ | journal 单写者；indexer 单写者；查询多读者（快照/RwLock） | 锁粒度 |
| 10 | 时间怎么注入？ | `Clock` trait；测试用可控时钟 | 可复现性 |
| 11 | schema 升级策略？ | 只加字段；语义变更升版本并写迁移函数 | 重放兼容 |
| 12 | 故障注入怎么做？ | 子进程 + 在写入边界 `kill -9`；测试钩子在 fsync 前后 | 验证强度 |

## 9. 契约定稿的完成标准

- [ ] 上面 12 个问题都有答案，并各自记录成 ADR（或合并成少数几篇）
- [ ] 能画出一次正常运行的完整事件序列
- [ ] 能画出 8 个崩溃点分别发生什么
- [ ] 能说明"至少一次 + 幂等键"与"exactly-once"的区别
- [ ] 能说明索引落后时查询返回什么
- [ ] 能说明日志截断后索引如何重建

在这之前不写 engine 代码。
