# 项目管理指南

## 地址

- 项目看板：https://github.com/users/RaphaelNY/projects/2
- 里程碑：https://github.com/RaphaelNY/durable-agent-runtime/milestones
- Issues：https://github.com/RaphaelNY/durable-agent-runtime/issues

## 模型

| 东西 | 作用 | 不要拿它做什么 |
| --- | --- | --- |
| **Issue** | 唯一任务条目：功能、bug、ADR、spike | 不要用 Project 里的草稿代替 issue |
| **Milestone** | 阶段（P0–P5） | 不要用它表达优先级 |
| **Project** | Issue/PR 的视图和属性 | 不是第二个任务库 |
| **Label** | 搜索和过滤（`area:*`、`type:*`） | 不要重复 Project 字段 |

## Project 字段

| 字段 | 取值 | 用途 |
| --- | --- | --- |
| Status（内置） | Todo / In Progress / Done | 当前状态 |
| Phase | P0–P5 | 属于哪个阶段 |
| Area | runtime / memory / ann / harness / platform | 属于哪个子系统 |
| Kind | ADR / spike / feature / test / bug / docs | 任务类型 |
| Size | S / M / L | 粗略工作量 |

## 推荐视图（在 UI 里创建）

1. **Board**：按 `Status` 分组。日常看"现在做什么"。
2. **Table**：按 `Area` 分组。看每个子系统欠了多少。
3. **Roadmap**：按 `Phase` 分组。看阶段分布；不要为了时间线硬填日期。
4. **My work**：过滤器 `assignee:@me status:Todo,In Progress`。只看自己的下一步。

创建方式：打开 Project → 视图标签栏右侧 `+` → New view → 选择布局（Table / Board / Roadmap）→ 命名 → Save。

## 图表与 Insights

- **Project → Insights**：查看 burn up / current iteration 等内置图。item 的 Status 和关闭时间越准确，图越有意义。
- **Milestones 页面**：每个 milestone 自带完成百分比进度条。
- **仓库 Insights**：Pulse、Contributors、Commits、Code frequency，用来看代码活动，不代表设计进度。
- **Actions 页面**：CI 通过率、耗时、失败历史。
- **自定义统计**：内置图表维度有限。需要按 Area/Phase 统计时，用
  `gh project item-list 2 --owner RaphaelNY --format json`
  导出后自己算；不要为了图表引入一套重型工具。

## 自动化（在 UI 开启）

Project → 右上角 `...` → Workflows：

- **Auto-add to project**：本仓库新建的 issue/PR 自动加入。
- **Item closed** → Status = Done。
- **Pull request merged** → 关联 item 置为 Done。

## Issue 与 PR 流程

1. 新任务用模板：ADR / Spike / Bug / Task。
2. 一个 issue 只写一个可验收的结果；验收项用 checkbox。
3. 开始做：Status → In Progress。
4. 提 PR：描述里写 `Closes #N`；填 PR 模板里的崩溃/并发场景。
5. CI 绿了再合并；合并后自动化会把 item 置为 Done。
6. 决策结论写进 `docs/decisions/`，issue 只保留讨论和验收。

## 每周 10 分钟复盘

1. 看 Board：In Progress 是否超过 2 个？超过就说明并行太多。
2. 把本周完成的 item 确认到 Done。
3. 从 Backlog 挑 1–2 个下周要做的，设 Status = Todo。
4. 按 `docs/LEARNING.md` 的复盘模板写 5 行，记在提交信息或 `docs/journal/`。
5. 更新一次 milestone 进度，确认下一个 gate 是什么。

## 什么算"有进展"

不要用代码行数或提交数衡量。对这套项目，进展是：

- 一个 ADR 从"开放问题"变成"已决定"；
- 一个崩溃点从"推理"变成"自动化测试覆盖"；
- 一个不变量从"文档里写了"变成"CI 每次都在验证"；
- 一组数据从"没有"变成"有 p50/p99 或 recall@10"。
