# AGENTS.md — lumi-codex 项目说明（codex / 其他终端 agent 用）

本文件给 Codex CLI / 其他 AI 助手提供项目上下文。Claude Code 会读 `CLAUDE.md`，两份内容保持同步。

## 项目本质

`lumi-codex` 是一个**学习项目**（不是生产软件），目的是在 14 天内（2026-05-09 到 2026-05-23）：
1. 用 Python 复刻 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构
2. 跑通主流 benchmark
3. 提供 hands-on 学习 agent 架构的机会

## 设计原则（重要）

工作时请遵守：

1. **设计文档是权威**：`docs/superpowers/specs/2026-05-09-lumi-codex-design.md`（v1.1 已根据 codex 评审修订）。修订原因见 `docs/superpowers/specs/2026-05-10-codex-research-plan-review.md`
2. **架构决策已经基于 codex 真实源码做过深读 + 评审**，不要凭印象建议改动
3. **Codex-core first 顺序**（不可颠倒）：
   - D2 protocol + **FakeProvider 必须先写**
   - D3 真实 OpenAI-compat provider（覆盖 Kimi/GLM/OpenAI）
   - D4-D6 tool 闭环（含 apply_patch 手写 lenient parser）
   - D7-D8 rollout/resume（含 4 类必测场景）
   - D9 compaction（v1 简化版：user_msgs + summary）
   - D10-D12 benchmark（Terminal-Bench subset 是硬目标）
   - D13 multi-agent demo + 模型对照
   - D14 收尾
4. **YAGNI 严格执行**：MCP / Hooks / Skills / Plugins / OS 沙箱 / app-server JSON-RPC / TUI Ratatui / Memories 这些 codex 有的东西 v1 **明确不做**
5. **拆 ApprovalPolicy 和 SandboxMode 两个概念**，不混淆
6. **Provider 优先级**：FakeProvider(0) > OpenAICompat(1, 覆盖 Kimi/GLM/OpenAI) > Anthropic native(2, 冲刺项)
7. **Apply patch 不依赖 Lark**：手写 lenient parser，先 add/delete/update/move 4 类
8. **Token 不要用 tiktoken**：真实 usage 用 provider 返回；预算/截断用 `approx_token_count`（chars/4 启发式）
9. **Tool 格式统一到 function-style `ToolSpec`**：provider 差异留在 adapter 层，每个 tool 不写多份

## 关键源码定位

完整列表见设计文档 Appendix A。最常用：

| 任务 | 看 codex 哪里 |
|---|---|
| Agent loop | `_research/codex/codex-rs/core/src/session/handlers.rs::submission_loop` |
| Turn 执行 | `_research/codex/codex-rs/core/src/session/turn.rs:136` |
| Op/EventMsg 全部变体 | `_research/codex/codex-rs/protocol/src/protocol.rs` |
| Compaction 策略 | `_research/codex/codex-rs/core/src/compact.rs:116/382/462` |
| Rollout 反向扫 resume | `_research/codex/codex-rs/core/src/session/rollout_reconstruction.rs:87/220` |
| Multi-agent delegate | `_research/codex/codex-rs/core/src/codex_delegate.rs:65/441` |
| Apply patch parser（手写，不是 Lark） | `_research/codex/codex-rs/apply-patch/src/parser.rs:1` |
| ModelProvider trait | `_research/codex/codex-rs/model-provider/src/provider.rs:79` |
| Approx token count | `_research/codex/codex-rs/core/src/context_manager/history.rs:262` |

注意 `_research/codex/` **不在本仓库内**，是 sibling 目录 `/Users/xujifeng/dev/_research/codex/`（用于参考）。本地快照版本：`c37f743`（2026-04-30），与本设计文档对应。

## 开发工具链

- **Python 包管理**：`uv`（不要用 pip / poetry）
- **测试**：`pytest`（带 `pytest-asyncio`）
- **Lint**：`ruff`
- **Type check**：`mypy`（可选）

## 提交风格

- 中文 commit message OK（项目作者母语）
- 遵循 conventional commits：`feat:` / `fix:` / `docs:` / `refactor:` / `test:`
- 每个组件独立 commit，便于回顾学习路径
- **不要添加任何 `Co-Authored-By:` 行**（包括 Claude / Codex / Claude Code / Codex CLI / 其他 AI 助手）。这是项目作者的明确要求

## Phase 划分

- **Phase 1（5/09-5/23，14 天）**：Python v1，本仓库当前阶段
- **Phase 2（v1 完成后可选）**：Rust 重写 `core` crate

详见设计文档 §11（修订版 14 天时间表）+ §13（v2 Rust 重写计划）。
