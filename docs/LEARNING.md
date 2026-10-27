# 学习路径

目标是"每一行都能解释为什么"，而不是"功能跑起来"。按阶段推进，每阶段先设计、再写、再测。

## 一、工作方式

```
读契约 → 画事件序列/崩溃点 → 写最小实现 → 写测试 → 故障注入 → 复盘
```

- 一次只做一个组件：journal 没站住，不要碰 indexer。
- 先写测试/崩溃场景，再写实现；崩溃测试比 happy path 更有价值。
- 每个阶段结束写 5 行复盘，记到 `docs/journal/` 或提交信息。
- 卡住超过 30 分钟：查文档、查参考实现（Temporal/DBOS/Restate 的文档，不是抄代码），再把问题写清楚问 agent。

目前**没有**本项目的参考实现；`archive/` 里是旧方向，不要拿它套这个项目。

## 二、P0：设计练习（先不写代码）

任务：

1. 回答 [CONTRACTS.md](CONTRACTS.md) 的 12 个问题，每个写成半页 ADR。
2. 画一条完整事件序列：RunStarted → ActivityScheduled → ActivityCompleted → RunCompleted。
3. 画 8 个崩溃点，每个写"恢复后状态"。
4. 定义 episode id 和 checkpoint 的字段，说明为什么这样能幂等。
5. 写"至少一次 vs exactly-once"的一段解释，能说服自己。

自测：

- 如果 journal 最后一条只写了一半，恢复后这条算不算完成？
- 活动执行了但完成事件没写，重启后会怎样？
- 索引 upsert 成功但 checkpoint 没写，重启后会重复吗？
- 日志截断后，索引的 offset 还有意义吗？

## 三、P1：执行引擎

**先修概念**：WAL、fsync/fdatasync、CRC、torn write、状态机、重放、幂等、`tokio::task::spawn_blocking`。

**实现顺序**：

1. journal 记录分帧 + append + read + 尾部截断
2. 状态机 trait + 命令生成 + 重放 + 状态哈希
3. activity 执行器：超时、重试、幂等键
4. 定时器
5. 崩溃注入（子进程 + `kill -9`）

**自测**：

- fsync 返回成功代表什么？失败代表什么？
- 为什么 CRC 不够防恶意篡改，但够防 torn write？
- 重放时命令序列和历史不一致，应该怎么处理？
- 幂等键为什么不能包含 attempt？
- `spawn_blocking` 打满会怎样？

## 四、P2：记忆管线

**先修概念**：事件溯源、投影、物化视图、checkpoint、背压、最终一致、幂等消费。

**实现顺序**：

1. Episode 结构 + 稳定 ID
2. projector 规则 + 确定性 embedder
3. 暴力索引（精确 top-k）
4. indexer：批量、upsert、checkpoint
5. 查询 + watermark

**自测**：

- 重复消费同一批事件，为什么不能产生重复 episode？
- checkpoint 和 upsert 不在一个事务里，靠什么保证正确？
- indexer 落后于写入时，应该丢数据还是背压？
- 查询返回 watermark 有什么用？
- 为什么不把索引更新放进写路径？

## 五、P3：HNSW

**先修概念**：HNSW 分层图、插入、邻居选择、efSearch/efConstruction、召回率、内存布局、tombstone。

**实现顺序**：

1. 单层图 + 暴力搜索基线
2. 分层构建 + 搜索
3. 参数调优（M、efConstruction、efSearch）
4. 持久化（快照 + 校验和）
5. 删除（tombstone + 重建）
6. 并发读

**自测**：

- HNSW 为什么能比暴力快？代价是什么？
- M 和 efConstruction 增大，召回和内存怎么变？
- 删除为什么不能直接摘链？
- 快照恢复后，图的结构怎么验证一致？
- 你的 recall@10 是怎么测的？测试集多大？

## 六、P4：集成与故障注入

**先修概念**：故障注入、属性测试、fuzz、指标、崩溃矩阵。

**实现**：

- 在写入边界、checkpoint 边界注入崩溃
- 组合故障：engine 崩溃 + indexer 崩溃
- 属性测试：任意事件的 append/read 往返；任意截断位置恢复
- fuzz：journal 解析不 panic
- benchmark 表

**自测**：

- 你的崩溃矩阵覆盖了哪些边界？漏了哪个？
- 恢复后怎么证明没有重复副作用？
- 索引追平时间怎么测？
- 哪个数字最难看？为什么？

## 七、Rust 特性地图

| 阶段 | 特性 | 落点 |
| --- | --- | --- |
| P1 | 枚举 + 模式匹配 | 事件类型、命令 |
| P1 | 错误设计 / `thiserror` | journal、状态机、activity |
| P1 | 所有权与借用 | 从日志读事件、零拷贝解析 |
| P1 | `spawn_blocking` | fsync 与磁盘 IO |
| P2 | `Arc<RwLock>` / 快照 | 索引读写并发 |
| P2 | `serde` 边界 | 事件 schema、版本兼容 |
| P3 | 泛型 vs trait object | `VectorIndex`、`Embedder` |
| P3 | 内存布局 / SIMD | 距离计算、图存储 |
| P4 | `proptest` / `cargo-fuzz` | 解析、恢复 |
| P4 | `criterion` | 恢复/吞吐/召回/延迟 |

## 八、怎么向 agent 提问

好的提问：

- "我的 journal 分帧方案是 X，崩溃在写了一半时恢复的行为是 Y，帮我找反例。"
- "这是我的 checkpoint 代码，按 bug/并发/恢复语义分类 review，不要重写。"
- "HNSW 删除用 tombstone 还是重建？给我三种方案的权衡和数据需求。"
- "这个编译错误我看不懂，给我提示，不要直接改。"

坏的提问：

- "帮我把 journal 写完。"
- "把 HNSW 实现出来。"
- "直接改到测试通过。"

## 九、复盘模板

```
阶段：
我最初的方案：
实际方案：
卡住最久的地方：
哪个崩溃点最意外：
如果重做会改什么：
下一个最小验证：
```
