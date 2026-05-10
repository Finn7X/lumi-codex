# Codex 源码定位（参考）

实现 lumi-codex 时对照 codex 真实源码的关键文件 + 行号。源码位置：`/Users/xujifeng/dev/_research/codex/codex-rs/`（不在本仓库内，sibling 目录）。

参考快照版本：`c37f743`（2026-04-30）。

---

## 主循环与协议

| 任务 | 文件 |
|---|---|
| `submission_loop` 主循环 | `core/src/session/handlers.rs:968` |
| `run_turn` 实现 | `core/src/session/turn.rs:136` |
| `auto_compact_limit` 定义 | `core/src/session/turn.rs:148` |
| 自动 compact 触发点 | `core/src/session/turn.rs:379` |
| `Op` enum 全部变体 | `protocol/src/protocol.rs:405` |
| `EventMsg` enum 全部变体 | `protocol/src/protocol.rs:1314` |
| `AskForApproval` enum | `protocol/src/protocol.rs:941` |
| `SandboxPolicy` / `PermissionProfile` | `protocol/src/protocol.rs:1035` |
| `TokenUsage` 类型 | `protocol/src/protocol.rs:2057` |
| `ThreadRolledBack` event | `protocol/src/protocol.rs:2809` |

## Tool 系统

| 任务 | 文件 |
|---|---|
| `ToolSpec` enum（多形态） | `tools/src/tool_spec.rs:18` |
| `ResponsesApiTool` 结构 | `tools/src/responses_api.rs:26` |
| `ToolRouter` | `core/src/tools/router.rs:39` |
| `update_plan` tool 定义 | `tools/src/plan_tool.rs:6` |
| `update_plan` handler | `core/src/tools/handlers/plan.rs:80` |
| Apply patch parser（手写 lenient，**不是 Lark**） | `apply-patch/src/parser.rs:1` |
| Apply patch grammar spec（Lark） | `tools/src/tool_apply_patch.lark` |

## Context & Compaction

| 任务 | 文件 |
|---|---|
| `COMPACT_USER_MESSAGE_MAX_TOKENS` | `core/src/compact.rs:44` |
| inline compaction 主流程 | `core/src/compact.rs:116` |
| `collect_user_messages` | `core/src/compact.rs:256` |
| compaction 中段细节 | `core/src/compact.rs:382` |
| `build_compacted_history` | `core/src/compact.rs:462` |
| remote compaction | `core/src/compact_remote.rs:113` |
| `approx_token_count`（截断/预算用） | `core/src/context_manager/history.rs:262` |

## Rollout & Resume

| 任务 | 文件 |
|---|---|
| `reconstruct_history_from_rollout` | `core/src/session/rollout_reconstruction.rs:87` |
| 反向扫 + segment 处理 | `core/src/session/rollout_reconstruction.rs:220` |

## Sandbox

| 任务 | 文件 |
|---|---|
| `SandboxType` enum | `sandboxing/src/manager.rs:23` |
| Seatbelt 调用（macOS） | `sandboxing/src/seatbelt.rs:26` |
| `sandbox-exec` 启动 | `sandboxing/src/manager.rs:196` |
| Linux 沙箱调用（landlock） | `core/src/landlock.rs:15` |

## Multi-agent

| 任务 | 文件 |
|---|---|
| `run_codex_thread_interactive` | `core/src/codex_delegate.rs:65` |
| `run_codex_thread_one_shot` | `core/src/codex_delegate.rs:163` |
| 审批拦截 `handle_exec_approval` | `core/src/codex_delegate.rs:284` |
| AgentControl（codex 多 agent 扩展能力，v1 不实现） | `core/src/codex_delegate.rs:441` |
| parent→child 边定义 | `agent-graph-store/src/types.rs` |
| 协作模式 prompt 模板 | `collaboration-mode-templates/src/lib.rs` |

## Model

| 任务 | 文件 |
|---|---|
| `ModelProvider` trait | `model-provider/src/provider.rs:79` |
| built-in providers（OpenAI / Bedrock / Ollama / LM Studio） | `model-provider-info/src/lib.rs:402` |
| `ResponseStream` 类型 | `core/src/client_common.rs:177` |
| `spawn_response_stream`（SSE） | `codex-api/src/sse/responses.rs:63` |

## Frontend

| 任务 | 文件 |
|---|---|
| CLI main 入口 | `cli/src/main.rs:733` |
| `exec` 子命令实现 | `exec/src/lib.rs:221` |
| TUI 主 event loop | `tui/src/app.rs:1020` |
