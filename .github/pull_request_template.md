## 变更

<!-- 一句话说明改了什么 -->

## 为什么

<!-- 解决哪个问题？Closes #N -->

## 影响面

- [ ] 事件 schema / journal 格式
- [ ] 恢复语义 / 崩溃点
- [ ] 并发 / 锁
- [ ] 索引一致性
- [ ] 仅文档

## 崩溃 / 并发场景

<!-- 涉及持久化时：列出崩溃点、重启后的预期状态、是否可能重复副作用 -->

## 测试

- [ ] `cargo fmt --all -- --check`
- [ ] `cargo clippy --workspace --all-targets --all-features -- -D warnings`
- [ ] `cargo test --workspace --all-features`
- [ ] 新增/更新测试
- [ ] 需要 nightly 崩溃矩阵（说明覆盖点）

## 文档

- [ ] 更新 ADR / CONTRACTS / ARCHITECTURE
- [ ] 不需要

Closes #
