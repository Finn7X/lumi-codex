# CLAUDE.md — lumi-codex 项目说明

本文件给 Claude Code / 其他 AI 助手提供项目上下文。

## 项目本质

`lumi-codex` 是一个**学习项目**（不是生产软件），目的是在 14 天内（2026-05-09 到 2026-05-22）：
1. 用 Python 复刻 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构
2. 跑通主流 benchmark
3. 提供 hands-on 学习 agent 架构的机会

## 设计原则（重要）

工作时请遵守：

1. **架构决策已经基于 codex 真实源码做过深读**，不要凭印象建议改动。所有关键决策的依据见 `docs/superpowers/specs/2026-05-09-lumi-codex-design.md` 的 §2 和 Appendix A
2. **YAGNI 严格执行**：MCP / Hooks / Skills / Plugins / OS 沙箱 / app-server JSON-RPC / TUI Ratatui 这些 codex 有的东西 v1 **明确不做**
3. **优先复刻 codex 思路**而不是自创：compaction 算法 / rollout schema / subagent 审批拦截 等都对照 `_research/codex/codex-rs/` 源码 1:1 实现
4. **Token 不要用 tiktoken**：codex 也不用，依赖模型返回的 `TokenUsage`
5. **Tool 格式统一到 `ResponsesApiTool`**：所有 provider 共用一套 schema，不写 per-provider parser

## 关键源码定位

完整列表见设计文档 Appendix A。最常用：

| 任务 | 看 codex 哪里 |
|---|---|
| Agent loop | `_research/codex/codex-rs/core/src/session/handlers.rs::submission_loop` |
| Op/EventMsg 全部变体 | `_research/codex/codex-rs/protocol/src/protocol.rs` |
| Compaction 策略 | `_research/codex/codex-rs/core/src/compact.rs` |
| Rollout 反向扫 resume | `_research/codex/codex-rs/core/src/session/rollout_reconstruction.rs` |
| Multi-agent delegate | `_research/codex/codex-rs/core/src/codex_delegate.rs` |
| Apply patch grammar | `_research/codex/codex-rs/tools/src/tool_apply_patch.lark` |
| ModelProvider trait | `_research/codex/codex-rs/model-provider/src/provider.rs` |

注意 `_research/codex/` **不在本仓库内**，是 sibling 目录 `/Users/xujifeng/dev/_research/codex/`（用于参考）。

## 开发工具链

- **Python 包管理**：`uv`（不要用 pip / poetry）
- **测试**：`pytest`
- **Lint**：`ruff`
- **Type check**：`mypy`（可选）

## 提交风格

- 中文 commit message OK（项目作者母语）
- 遵循 conventional commits：`feat:` / `fix:` / `docs:` / `refactor:` / `test:`
- 每个组件独立 commit，便于回顾学习路径
- **不要添加任何 `Co-Authored-By:` 行**（包括 Claude / Claude Code / 其他 AI 助手）。这是项目作者的明确要求

## Phase 划分

- **Phase 1（5/09-5/22，14 天）**：Python v1，本仓库当前阶段
- **Phase 2（v1 完成后可选）**：Rust 重写 `core` crate

详见设计文档 §11（14 天时间表）+ §13（v2 Rust 重写计划）。
