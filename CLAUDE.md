# CLAUDE.md — lumi-codex 项目说明

本文件给 Claude Code / 其他 AI 助手提供项目上下文。

## 项目本质

`lumi-codex` 是一个用 Python 实现的终端 agent，复刻 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构，覆盖 Agent Loop / Task 规划 / Context 管理 / Session 管理 / Multi-Agent 编排 5 大能力。

## 设计原则

工作时请遵守：

1. **架构文档是权威**：[`docs/architecture.md`](./docs/architecture.md)（整体决策）+ [`docs/implementation.md`](./docs/implementation.md)（模块细节）。变动必须更新这两份
2. **YAGNI 严格执行**：MCP / Hooks / Skills / Plugins / OS 沙箱 / app-server JSON-RPC / Ratatui TUI / Memories 这些 codex 有的能力**明确不做**
3. **拆 ApprovalPolicy 与 SandboxMode**：两个独立概念，不要混淆
4. **Provider 优先级**：FakeProvider（测试基础）→ OpenAICompat（覆盖 Kimi/GLM/OpenAI）→ Anthropic native（adapter）
5. **Apply patch 不依赖 Lark**：手写 lenient parser，先 add/delete/update/move 4 类
6. **Token 不要用 tiktoken**：真实 usage 用 provider 返回；预算/截断用 `approx_token_count`（chars/4 启发式）
7. **Tool 格式统一到 function-style `ToolSpec`**：provider 差异留在 adapter 层，每个 tool 不写多份
8. **复刻 codex 的核心机制，不自创**：compaction 算法 / rollout schema / subagent 审批拦截 等都对照源码实现
9. **FakeProvider 是测试基础**：所有 loop/router/rollout/approval/compaction 测试都不依赖真实模型

## 关键源码定位

完整列表见 [`docs/codex-source-map.md`](./docs/codex-source-map.md)。最常用：

| 任务 | 看 codex 哪里 |
|---|---|
| Agent loop | `_research/codex/codex-rs/core/src/session/handlers.rs::submission_loop` |
| Op/EventMsg 全部变体 | `_research/codex/codex-rs/protocol/src/protocol.rs` |
| Compaction 策略 | `_research/codex/codex-rs/core/src/compact.rs:116/382/462` |
| Rollout 反向扫 resume | `_research/codex/codex-rs/core/src/session/rollout_reconstruction.rs:87/220` |
| Multi-agent delegate | `_research/codex/codex-rs/core/src/codex_delegate.rs:65/441` |
| Apply patch parser（手写，不是 Lark） | `_research/codex/codex-rs/apply-patch/src/parser.rs:1` |
| ModelProvider trait | `_research/codex/codex-rs/model-provider/src/provider.rs:79` |

`_research/codex/` **不在本仓库内**，是 sibling 目录 `/Users/xujifeng/dev/_research/codex/`（参考用）。本地快照版本：`c37f743`（2026-04-30）。

## 开发工具链

- **Python 包管理**：`uv`（不要用 pip / poetry）
- **测试**：`pytest`（带 `pytest-asyncio`）
- **Lint**：`ruff`
- **Type check**：`mypy`（可选）

## 提交风格

- 中文 commit message OK
- 遵循 conventional commits：`feat:` / `fix:` / `docs:` / `refactor:` / `test:`
- 每个组件独立 commit，便于回顾
- **不要添加任何 `Co-Authored-By:` 行**（包括 Claude / Claude Code / 其他 AI 助手）。这是项目作者的明确要求

## 构建顺序

按 [`docs/building.md`](./docs/building.md) 的阶段顺序执行。每个阶段完成后才能开始下一阶段。
