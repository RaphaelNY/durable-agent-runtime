# 0004：路线图、工作流与项目管理

- 日期：2026-10-27
- 状态：接受

## 背景

项目最初只包含"持久化执行引擎 + 记忆索引"。讨论后确认可以扩展为一个完整平台：自研运行时、薄 harness、HNSW，以及可替换的执行后端。范围变大后，需要固定阶段顺序、工作流划分和任务管理方式，否则容易同时开工、每层都浅。

## 决定

### 1. 三个里程碑，按 A → B → C 推进

- **A 运行时底座**：P0 契约 → P1 运行时 → P2 记忆管线 → P3 HNSW
- **B 薄 harness**：P4，用真实 agent run 证明运行时
- **C 平台化**：P5，第二个适配器 + conformance suite + harness 广度

HNSW（P3）与 P4 可以并行，它只依赖 `VectorIndex` trait。

### 2. 五条工作流

| Area | 内容 |
| --- | --- |
| `runtime` | journal、重放、调度、activity、timer、快照 |
| `memory` | projector、embedder、indexer、checkpoint、watermark |
| `ann` | HNSW 构建/搜索/持久化/删除 |
| `harness` | agent loop、上下文、工具、模型适配、CLI |
| `platform` | execution API、适配器、conformance、CI/CD |

### 3. Execution API 在 P1 就位

不等到 P5 再抽接口。P1 同时交付：

- harness 需要的执行端口（start / signal / query / cancel / capabilities）；
- 内存适配器（测试替身）；
- conformance suite v0。

理由：先写 durable runtime 再抽接口，接口会泄漏 journal 的实现假设。

### 4. 任务管理

- **Issue 是唯一任务条目**；Project 只是视图。
- **Milestone 管阶段**（P0–P5），**Project 字段管属性**（Phase / Area / Kind / Size），**Labels 只做搜索**（`area:*`、`type:*`）。
- PR 描述写 `Closes #N`；合并后 Project item 自动关闭。
- ADR 的结论落在 `docs/decisions/`，issue 只保留讨论和验收。
- 每个 spike 有时间盒；超时就交当前结论，不无限延长。

### 5. CI 分层

- **PR**：fmt、clippy `-D warnings`、test（Ubuntu）、doc。
- **main / nightly**：加 macOS（fsync 语义不同）、崩溃矩阵、proptest、benchmark。
- **release**：tag `v*` 构建二进制并发布。

CI 在第一个 `Cargo.toml` 落地后启用；workflow 用路径过滤，避免纯文档提交变红。

## 后果

- 阶段边界清晰：每个 milestone 可停可交付。
- 项目管理有少量仪式感，但避免"写了半天不知道做到哪"。
- P1 多花 1–2 周做 execution API 和内存适配器，换来 P5 的可替换性。
- HNSW 可以独立推进，不被 harness 阻塞。
- 没有第二个真实后端时，"可替换"由内存适配器和 conformance suite 证明；Temporal 适配器可选。
