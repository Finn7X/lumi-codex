# 学习笔记（Day 1-3 + Day 14 产出）

7 篇关于 codex 实现的笔记，目标：自己能把这一块讲清楚。

## 笔记列表

| # | 标题 | 状态 | 目标完成 |
|---|---|---|---|
| 01 | [codex-rs Workspace 总览](./01-codex-workspace-overview.md) | TODO | Day 1 (5/09) |
| 02 | [Submission/Event 总线设计](./02-submission-event-bus.md) | TODO | Day 2 (5/10) |
| 03 | [Context 管理与 Compaction](./03-context-and-compaction.md) | TODO | Day 3 (5/11) |
| 04 | [Rollout 与 Resume](./04-rollout-and-resume.md) | TODO | Day 3 (5/11) |
| 05 | [Tool 系统与 Sandbox](./05-tool-and-sandbox.md) | TODO | Day 3 (5/11) |
| 06 | [Multi-Agent (codex_delegate)](./06-multi-agent-delegate.md) | TODO | Day 3 (5/11) |
| 07 | [codex vs Claude Code](./07-codex-vs-claude-code.md) | TODO | Day 14 (5/22)，v1 完工后写 |

## 写作原则

- **代码引用要准确**：每个论断附 `codex-rs/path/to/file.rs:行号`
- **拒绝二手转述**：不要从博客 / wiki 抄，必须读源码
- **保留疑惑**：读不懂的地方记 `> ❓ 疑问：xxx`，回头解答
- **图说明流程**：每篇至少 1 个 ASCII / mermaid 图说明数据流

## 参考

完整设计文档：[docs/superpowers/specs/2026-05-09-lumi-codex-design.md](../superpowers/specs/2026-05-09-lumi-codex-design.md)
