# Durable Agent Runtime

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

一个可崩溃恢复、可确定性重放的 Agent 运行时：**日志是事实来源，语义记忆是它的派生索引。**

> 状态：P0 设计阶段。代码尚未开始，先在 [docs/CONTRACTS.md](docs/CONTRACTS.md) 里把事件 schema、checkpoint 和恢复语义定下来，再动手写。

## 为什么做这个

Agent 的一次运行往往很长、很贵、带副作用：调用模型、查数据库、发请求、写文件。中途崩了以后，现有做法很难回答三个问题：

1. 已经执行到哪一步？
2. 恢复后会不会重复执行有副作用的工具（比如重复扣款）？
3. 怎么复现这次失败，而不是靠日志猜？

同时，Agent 还需要“记忆”：从历史运行里做语义检索。常见的做法是另起一套向量库存一份数据，结果日志和索引各说各话。

这个项目把两件事放在同一条事实来源上：

- **执行引擎**把每一步追加到 journal，崩溃后重放恢复，已完成的活动不重复执行。
- **记忆索引**异步消费 journal，把 episode 投影进 HNSW，作为派生视图；索引可以落后、可以重建，永远不阻塞写入。

## 拓扑

```
┌──────────────────────────────┐
│ Agent 工作流（执行引擎）      │
│  activity: memory.search ────┼───────────────┐
└──────────────┬───────────────┘               │ 查询 top-k
               │ 追加事件                       │
               ▼                               ▼
        ┌─────────────┐   异步、至少一次   ┌──────────────┐
        │  Journal    │ ─────────────────► │ Indexer      │
        │（事实来源）  │                    │ offset+快照  │
        └─────────────┘                    └──────┬───────┘
                                                  │ upsert/search
                                                  ▼
                                           ┌─────────────┐
                                           │ HNSW 索引    │
                                           │（派生视图）  │
                                           └─────────────┘
```

三条硬规则：

1. **写路径单向**：引擎只写 journal，不等索引。索引故障不能阻塞 Agent 运行。
2. **读路径显式**：`memory.search` 是一个普通 activity，有超时和重试；索引挂了只影响这一步。
3. **索引可重建**：journal 是真相，HNSW 是投影。索引丢了、落后了、写坏了，都能从 journal 重放恢复。

## 核心不变量

1. 已完成的 activity 在恢复后不会再次产生副作用：结果从 journal 复用，外部副作用靠幂等键收敛。
2. 任意崩溃点恢复后，run 状态与未崩溃执行一致；同一 journal 重放得到相同状态哈希。
3. journal 尾部半条记录可被检测并截断到最后一条完整记录。
4. 索引是派生视图：可以落后、可以重建，任何时刻都不能反向影响执行引擎。
5. 查询能返回自己的 watermark（索引已消费到哪），让调用方知道数据新鲜度。

## 非目标

- 分布式 worker、跨机调度、共识
- 任意副作用的 exactly-once（做不到，见 [docs/CONTRACTS.md](docs/CONTRACTS.md)）
- 多租户、配额、计费
- 通用工作流 DSL、可视化编排
- 本地大模型推理（embedding 先用确定性桩）

## 目录

```
durable-agent-runtime/
├── docs/
│   ├── ARCHITECTURE.md    # 组件、数据流、失败矩阵
│   ├── CONTRACTS.md       # 事件 schema、checkpoint、恢复语义、开放问题
│   ├── PLAN.md            # P0–P4 分期与验收
│   ├── FEASIBILITY.md     # 为什么这个组合自洽、风险在哪
│   ├── INTERVIEW.md       # 面试追问、叙事、数据
│   ├── LEARNING.md        # 每个阶段学什么、怎么练
│   └── decisions/         # ADR
├── archive/               # 旧方向（SecureAgent Runtime）存档，不参与新项目
├── AGENTS.md              # agent 协作规则：导师模式，不代写
└── rust-toolchain.toml
```

计划中的 crate（等契约定稿后再建）：

- `schema`：事件、ID、trait，无业务依赖
- `engine`：journal、replay、调度、活动、定时器、快照
- `memory`：投影、索引器、checkpoint、HNSW、查询
- `demo`：端到端示例与故障注入

## 先读什么

1. [docs/CONTRACTS.md](docs/CONTRACTS.md) —— 现在最该先定的是事件 schema 和 checkpoint 契约
2. [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) —— 组件与失败矩阵
3. [docs/PLAN.md](docs/PLAN.md) —— 分期与验收
4. [docs/INTERVIEW.md](docs/INTERVIEW.md) —— 这个项目在面试里怎么讲

## 进度与任务

- 当前阶段：**P0 契约**（详见 [PLAN.md](docs/PLAN.md)）
- [项目看板](https://github.com/users/RaphaelNY/projects/2)：Board / Table / Roadmap 三种视图
- [里程碑](https://github.com/RaphaelNY/durable-agent-runtime/milestones)：P0–P5
- [Issues](https://github.com/RaphaelNY/durable-agent-runtime/issues)：ADR、spike、功能、测试、bug

任务流转：Issue → PR（`Closes #N`）→ CI → 合并。决策结论写进 [docs/decisions/](docs/decisions/)，不只留在 issue 评论里。

## 协作方式

这是一个学习优先的个人项目，核心代码自己写。见 [AGENTS.md](AGENTS.md)：agent 默认只解释、给方案、review，不代写实现。

## License

[MIT](LICENSE) © 2026 RaphaelNY
