# 开发计划（长期、业余、手写）

时间不再是硬约束，但每个阶段都要有**可停可交付**的边界。任何阶段停下来，前面的成果都能独立演示。

## 总览

| 阶段 | 主题 | 预估（业余） | 可独立交付 |
| --- | --- | --- | --- |
| P0 | 契约定稿 | 2–3 周 | 事件 schema、崩溃矩阵、ADR |
| P1 | 执行引擎 | 8–10 周 | journal + 重放 + 崩溃恢复 |
| P2 | 记忆管线 | 3–4 周 | 投影 + 暴力索引 + checkpoint |
| P3 | HNSW | 4–6 周 | 从零实现 + 召回/延迟数据 |
| P4 | 集成 | 3–4 周 | 端到端 demo + 故障矩阵 |

总计约 20–27 周。P2 结束后即使不做 P3，项目也成立（暴力索引够用）；P3 卡住时可以随时降级。

## P0：契约定稿（不写业务代码）

**目标**：把 [CONTRACTS.md](CONTRACTS.md) 的 12 个问题全部回答，写成 ADR。

任务：

- [ ] 事件类型定稿：字段、digest、错误分类、schema_version
- [ ] journal 分帧定稿：JSONL+CRC 还是二进制；尾部损坏怎么处理
- [ ] 状态机契约：`on_event` 的确定性规则、命令与历史匹配
- [ ] Activity 契约：幂等键、重试、超时、取消
- [ ] 索引契约：episode id、checkpoint、watermark、重建
- [ ] 8 个崩溃点的期望行为
- [ ] 数据模型：payload 内联 vs blob，日志是否截断
- [ ] 写 2–3 篇 ADR

**验收**：不看代码，能画出一条完整事件序列和 8 个崩溃点的恢复结果。

## P1：执行引擎

**学习**：WAL、fsync、CRC、torn write、状态机、重放、幂等、tokio 阻塞边界。

**实现**：

1. journal：append、read、CRC、截断尾部、两种 fsync 模式（每条 / group commit）
2. 状态机：`Workflow` trait、命令生成、重放、非确定性检测
3. 活动执行器：超时、重试、退避、幂等键
4. 定时器：SetTimer / TimerFired
5. 崩溃注入：子进程 + 在写入边界 `kill -9`

**验收**：

- 20 个崩溃点 × 100 次运行，不变量违反 = 0
- 同一 journal 重放 100 次，状态哈希一致
- 尾部半条记录能截断，且不影响之前的记录
- `cargo clippy -D warnings` 干净

**数据**：

- 恢复时间 vs journal 大小（10KB / 1MB / 100MB）
- steps/s：每条 fsync vs group commit
- 崩溃点覆盖矩阵

**面试落点**：崩溃一致性、至少一次语义、幂等键、重放确定性。

## P2：记忆管线

**学习**：事件溯源、投影、checkpoint、背压、最终一致、幂等消费。

**实现**：

1. projector：从事件流抽取 episode（规则可配置）
2. 确定性 embedder：hash 到固定维向量，测试可复现
3. 暴力索引：精确 top-k，作为 HNSW 的对照基线
4. indexer：批量消费、upsert、checkpoint
5. 查询：top-k + watermark + filter

**验收**：

- 重复消费不产生重复 episode（重复数 = 0）
- 崩溃在 upsert 与 checkpoint 之间，重启后追平且不重复
- 删除索引目录后能从 journal 重建
- 查询返回 watermark，能说明数据新鲜度

**数据**：

- 索引落后时间 p50/p99
- 崩溃后追平时间
- 重建时间（1k / 10k / 100k 事件）

**面试落点**：派生视图、checkpoint 原子性、至少一次消费、背压。

## P3：HNSW

**学习**：HNSW 构建/搜索、召回率、内存布局、持久化、并发读、benchmark。

**实现**：

1. 构建：逐层插入、邻居选择、M / efConstruction
2. 搜索：efSearch、top-k、过滤
3. 持久化：图快照 + 校验和 + 恢复
4. 删除：tombstone + 周期性重建
5. 并发：单写多读（快照或 RwLock）

**验收**：

- 在测试集上 recall@10 ≥ 0.95（对比暴力索引）
- 查询 p99 延迟和内存占用有数据
- 快照恢复后图结构一致
- 删除后召回率不崩

**数据**：recall@10、QPS、p99、内存、构建时间、快照大小。

**面试落点**：HNSW vs IVF/flat、M/ef 参数、删除、并发、持久化。

## P4：集成

**目标**：把 engine、indexer、HNSW 串成端到端系统，并用故障注入验证。

**实现**：

1. 端到端 demo：一次 Agent run 写入事件 → 索引可查 → 崩溃 → 恢复 → 结果不重复
2. 索引崩溃恢复：checkpoint、追平、重建
3. 新鲜度：写入到可检索的延迟、watermark
4. 日志保留策略（如果 P0 决定做）
5. 故障矩阵：engine 崩溃 × indexer 崩溃 × 磁盘错误
6. README、benchmark 表、3 分钟演示

**验收**：

- 4 类故障场景可复现，且不变量成立
- 写入到可检索延迟、追平时间有数据
- 别人按 README 能在 10 分钟内跑起来

## 可选 P5（做完 P4 再决定）

- 本地 embedding（小模型或 API，不进关键路径）
- HTTP 查询接口
- 日志保留 / 分层存储
- 分布式 worker（不建议，会引入共识问题）

## 如果时间不够，按这个顺序砍

1. P5 全部
2. HNSW（保留暴力索引，项目仍成立）
3. 日志保留策略（MVP 不截断）
4. 本地 embedding（用 hash embedder）
5. 删除/tombstone（只做插入）

**不要砍**：P0 契约、P1 崩溃测试、P2 幂等/追平、README 的设计取舍。这四样是项目的骨架。

## 项目完成的定义

- [ ] 8 个崩溃点的行为与契约一致，有自动化测试
- [ ] 索引重复数 = 0，追平/重建可复现
- [ ] 至少 2 组真实数据（engine 恢复/吞吐、index recall/latency）
- [ ] 3 篇 ADR
- [ ] 3 分钟能讲完，能回答 15 个追问
- [ ] 能诚实列出没做的部分（分布式、exactly-once、多租户）
