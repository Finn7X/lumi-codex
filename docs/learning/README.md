# 学习笔记

阅读 codex 源码后的产出，每篇能让自己向他人讲清楚一块设计。

## 笔记列表

| # | 标题 | 内容 |
|---|---|---|
| 01 | codex-rs Workspace 总览 | 100+ crate 分类（核心 / 扩展 / cloud / 外围）；阅读路径推荐 |
| 02 | Submission/Event 总线设计 | submission_loop 主循环；Op/EventMsg 全部变体分类；为什么 channel-driven 而不是 request-response；async task 派发模式 |
| 03 | Context 管理与 Compaction | inline vs remote compaction；`build_compacted_history` 三段结构；`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`；fake assistant message 存 summary 的设计；mid-turn 注入 initial context 的细节 |
| 04 | Rollout 与 Resume | RolloutItem 5 种；反向扫找 checkpoint 的精妙；与 SQLite state 的分工；`ThreadRolledBack` 处理 |
| 05 | Tool 系统与 Sandbox | `ToolSpec` 多形态；`AskForApproval` vs `SandboxPolicy` 两层概念分离；Seatbelt vs Landlock vs Windows sandbox 实现差异；apply_patch 手写 lenient parser |
| 06 | Multi-Agent (codex_delegate) | fork-join 模式；父级审批拦截的实现细节；agent-graph-store 的真实作用；与 Claude Code subagent 对比 |
| 07 | codex vs Claude Code | Hooks/Skills/Plugins/MCP 设计差异；多 agent 模型差异 |

## 写作原则

- **代码引用要准确**：每个论断附 `codex-rs/path/to/file.rs:行号`
- **拒绝二手转述**：不要从博客 / wiki 抄，必须读源码
- **保留疑惑**：读不懂的地方记 `> ❓ 疑问：xxx`，回头解答
- **图说明流程**：每篇至少 1 个 ASCII / mermaid 图说明数据流

## 参考

- [架构文档](../architecture.md)
- [模块实现细节](../implementation.md)
- [codex 源码定位](../codex-source-map.md)
