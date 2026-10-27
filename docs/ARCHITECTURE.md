# 架构

## 1. 目标与约束

目标：单机、嵌入式、可崩溃恢复、可确定性重放的 Agent 工作流运行时，并在这条事实来源上构建可重建的语义记忆索引。

约束：

- 单进程、单机；不做分布式共识。
- 写入是单写者顺序追加；读取可以并发。
- 副作用只能做到"至少一次 + 幂等键"，不承诺任意副作用的 exactly-once。
- 索引是派生视图，允许最终一致，允许重建。

## 2. 组件

```
crates/schema   ← 事件、ID、trait（无业务依赖）
     ▲     ▲
     │     │
crates/engine  crates/memory
     ▲             ▲
     └──── crates/demo ────┘
```

### schema

- `EventEnvelope`、`RunId`、`EventId`、`StepId`、`EpisodeId`
- `Activity` / `ActivityExecutor` trait
- `VectorIndex` trait（upsert / search / snapshot / restore）
- `Embedder` trait

schema 不依赖 engine 或 memory；两个 crate 只依赖 schema，互不依赖。

### engine

- **Journal**：追加写、记录分帧、校验和、fsync 策略、尾部修复。
- **Run state machine**：从 journal 重放得到 run 状态；命令与历史必须匹配（防不确定性）。
- **Scheduler**：按 run 串行推进；activity 进入执行池；定时器到期触发。
- **Activity executor**：超时、重试、退避；幂等键透传。
- **Snapshot / compaction**：run 状态快照 + journal 截断（策略见开放问题）。
- **Crash recovery**：扫描 journal、截断半条记录、重建状态、恢复 pending activity/timer。

### memory

- **Projector**：从 journal 事件流里抽取 episode（可配置规则：只索引工具结果、只索引摘要等）。
- **Indexer**：批量消费、调用 embedder、写入 `VectorIndex`，推进 checkpoint。
- **Checkpoint**：`last_offset + snapshot_id + schema_version + checksum`。
- **HNSW**：P3 从零实现；P2 先用暴力索引验证管线。
- **Query**：`search(vector, k, filter, as_of)`，返回 `watermark`。

## 3. 数据流

### 写路径

```
workflow step
  → engine 构造事件
  → journal.append(event)        （按 durability 配置 fsync）
  → 返回 step 已持久化
  ↓
indexer 异步 tail journal
  → projector 抽取 episode
  → embedder
  → index.upsert(episode_id, vector)
  → checkpoint(offset)
```

关键：indexer 的进度**不在**写路径上。写入成功只依赖 journal 落盘。

### 读路径

```
workflow activity: memory.search
  → engine 像调度普通 activity 一样调度它
  → query: index.search(vector, k, filter)
  → 返回 episode_id + watermark
  → engine 用 episode_id 从 episode store / journal 取回原文
```

读路径的失败是 activity 失败，走重试/超时；不影响 run 的持久状态。

### 崩溃恢复

| 崩溃点 | 恢复行为 |
| --- | --- |
| journal 追加中（尾部半条） | 校验失败 → 截断到上一条完整记录；该步骤视为未完成 |
| journal 已 fsync，activity 未执行 | 重放后重新调度 activity |
| activity 副作用已发生，journal 未记录 | 无法自动消除；依赖幂等键，重启后再次执行是安全的 |
| activity 完成，journal 已记录 | 重放时直接复用结果，不重新执行 |
| indexer 写完索引，checkpoint 未落盘 | 重启后从旧 checkpoint 重放；upsert 幂等，不产生重复 |
| index 快照损坏 | 从 journal / episode store 重建索引 |
| journal 已 compaction，索引 offset 失效 | 依赖稳定 episode ID + episode store；见开放问题 |

## 4. 一致性模型

- **Journal**：单写者、顺序追加；`durable` 模式下每次 append 后 fsync，`group-commit` 模式下批量 fsync。恢复只保证"最后一条完整记录之前"的状态。
- **Engine state**：从 journal 确定性重放；同一 journal 的状态哈希必须一致。
- **Memory index**：最终一致。`watermark` 表示索引已消费到的位置；查询结果可能落后于最新 journal。
- **读己之写**：默认不保证。工作流如果在同一个 run 里刚写入记忆就查询，可能查不到；需要显式等待 watermark 或使用 `as_of` 语义。

## 5. 失败矩阵

| 失败 | 影响 | 处理 |
| --- | --- | --- |
| 进程崩溃 | 未完成的 run 暂停 | 重启恢复，重放 journal |
| 磁盘满 / fsync 失败 | append 失败 | 拒绝推进 run；向上返回可重试错误 |
| journal 尾部损坏 | 最后一条不完整 | 截断；记录告警 |
| activity 超时 | 步骤失败 | 按策略重试；超过上限则 run 失败 |
| activity 重复执行 | 可能重复副作用 | 幂等键；文档明确"至少一次" |
| indexer 落后 | 查询结果旧 | watermark + lag 指标；背压/批处理 |
| index 写坏 | 查询错误或缺失 | 从 checkpoint 重放；必要时全量重建 |
| embedder 不可用 | 索引停止推进 | 重试；不阻塞 engine |

## 6. 为什么 HNSW 是派生视图

HNSW 不是第二个事实来源：

- 它可以从 journal + episode store 重建。
- 它允许落后，允许丢失，允许换实现（暴力索引、HNSW、未来的其他 ANN）。
- 它不能反向写 journal，也不能让 engine 等它。

因此 engine 和 memory 可以独立测试：engine 不需要向量；memory 用合成事件流就能测。集成测试只验证边界行为：幂等、追平、watermark、崩溃恢复。

## 7. 与 CONTRACTS 的关系

本文件描述"系统怎么动"；具体的字段、分帧、trait 签名、开放问题在 [CONTRACTS.md](CONTRACTS.md)。在契约定稿前不要开始写 engine，否则事件 schema 会反复返工。

## 8. 主要设计风险

1. **事件 schema 过早定型**：后续加字段、改语义会破坏重放兼容性。需要 `schema_version` 和升级策略。
2. **日志截断与索引 offset**：最容易埋雷的地方。先决定 episode 的稳定标识和 payload 存放位置。
3. **把索引放进写路径**：会让持久性依赖检索延迟，必须避免。
4. **把"至少一次"说成"exactly-once"**：面试和文档里都要如实。
5. **HNSW 范围膨胀**：P3 只做单机、内存、可持久化、可删除的基本实现，不做分布式/GPU。
