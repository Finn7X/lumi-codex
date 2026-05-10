# lumi-codex 架构

## 项目概述

lumi-codex 是用 Python 实现的终端 agent，参考 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构，覆盖 5 大能力：

- **Agent Loop** — channel-driven submission/event 总线 + tool-call continuation
- **Task 规划** — `update_plan` tool（让模型自维护 checklist）
- **Context 管理** — MessageHistory + 独立 LLM call 摘要 compaction
- **Session 管理** — JSONL rollout + 反向扫 resume
- **Multi-Agent 编排** — one-shot SubagentDelegate fork-join + 父级审批拦截

设计原则：**复刻 codex 的核心机制，不复刻其工程外延**（OS 沙箱、IDE 桥、MCP server、Hooks/Skills/Plugins 等不实现）。

## 5 大能力映射

| 能力 | lumi-codex 实现 | codex 对应代码 |
|---|---|---|
| Agent Loop | `submission_loop` + tool-call continuation | `core/src/session/handlers.rs::submission_loop`、`core/src/session/turn.rs:136` |
| Task 规划 | `update_plan` tool | `tools/src/plan_tool.rs:6`、`core/src/tools/handlers/plan.rs:80` |
| Context 管理 | `MessageHistory` + 独立 LLM call 摘要 | `core/src/compact.rs:116/382/462`、`core/src/context_manager/history.rs:262` |
| Session 管理 | JSONL rollout + 反向扫 resume | `core/src/session/rollout_reconstruction.rs:87/220` |
| Multi-Agent 编排 | one-shot `SubagentDelegate` + 审批拦截 | `core/src/codex_delegate.rs:65/441` |

## 关键架构决策

| 决策 | 选择 |
|---|---|
| **Submission/Event 总线** | Python `asyncio.Queue` + `dataclass` Op/Event |
| **Streaming** | `httpx` SSE + `asyncio.Queue`（流式 delta 第一天就支持）|
| **FakeProvider** | 测试基础。返回预设 chunks 序列。所有 loop/router/rollout/approval/compaction 测试都不依赖真实模型 |
| **Tool 统一格式** | function-style `ToolSpec(name, description, input_schema, output_schema?)`；provider 差异留在 adapter 层，每个 tool 不写多份 |
| **ModelProvider** | 单一 `ModelProvider` ABC（管 `stream_complete()`）+ TOML 配置 |
| **Provider 优先级** | OpenAI-compatible（一份代码覆盖 OpenAI / Kimi / GLM）；Anthropic native 通过 adapter 后续接入 |
| **ApprovalPolicy** | 3 档：`on_request` / `unless_trusted` / `never`（**何时问用户**）|
| **SandboxMode** | 3 档：`read_only` / `workspace_write` / `danger_full_access`（**工具能访问哪里**）|
| **沙箱实现** | cwd guard + 命令黑名单 + subprocess 隔离 |
| **Apply patch** | 手写 lenient parser，兼容 codex patch 格式（add/delete/update/move 4 类）|
| **Subagent** | one-shot fork-join + 父级审批拦截 |
| **Plan** | `update_plan` tool（让模型自维护 checklist），不是独立 planner |
| **Compaction** | 独立 LLM call 摘要：`selected_user_messages + assistant_summary` |
| **Token 估算** | 真实 usage 用 provider 返回；预算/截断用 `approx_token_count`（chars/4 启发式），不用 tiktoken |
| **Rollout JSONL** | 5 种 RolloutItem：`SessionMeta` / `ResponseItem` / `Compacted` / `TurnContext` / `EventMsg` |
| **Resume** | 反向扫找最近 `Compacted.replacement_history` 作 checkpoint，正向 replay 后缀 |
| **Frontend** | `rich` + `prompt_toolkit` 的 CLI（同时是 benchmark runner）|

## 不实现的功能（YAGNI）

明确不做以下能力（codex 有，但与"理解 agent 核心架构"无关）：

- **OS 沙箱**（Seatbelt / Landlock / bubblewrap / windows-sandbox）—— 用 ApprovalPolicy + cwd guard 替代
- **MCP 客户端 + 服务端**（`rmcp-client` + `mcp-server` + `connectors`）
- **Hooks 系统**（6 种 hook event）
- **Skills 系统**（SKILLS.md + 依赖声明）
- **Plugins**（目录式扩展）
- **Memories**（长期跨会话记忆，Phase1 提取 + Phase2 合并）
- **app-server JSON-RPC 协议** / IDE 桥
- **Ratatui 级别 TUI** —— 用 `rich` + `prompt_toolkit`
- **Goals**（thread 级目标 + token budget）
- **Code mode**（V8 JS sandbox）
- **Cloud tasks** / **Realtime conversation** / **Reviews**
- **Connectors**（Slack/GitHub OAuth 集成）

## Op / EventMsg 生命周期

不复刻 codex 的全部 30+/70+ 变体，按生命周期复刻最小集。

### Op

| Op | 用途 |
|---|---|
| `UserInput` / `UserTurn` | 提交用户任务 |
| `ExecApproval` | 回复 shell 审批 |
| `PatchApproval` | 回复 patch 审批 |
| `Interrupt` | 中断当前 turn |
| `Compact` | 手动压缩上下文 |
| `ThreadRollback` | 丢弃最近 N 个用户 turn |
| `Shutdown` | 关闭 session |

### EventMsg

| EventMsg | 用途 |
|---|---|
| `TurnStarted` / `TurnComplete` / `TurnAborted` | turn 生命周期 |
| `AgentMessageDelta` / `AgentMessage` | assistant 流式输出 |
| `UserMessage` | 标识写入模型的输入 |
| `ExecCommandBegin` / `ExecCommandOutputDelta` / `ExecCommandEnd` | shell 工具事件 |
| `ExecApprovalRequest` / `ApplyPatchApprovalRequest` | 审批请求 |
| `PlanUpdate` | 规划工具更新 |
| `TokenCount` | token usage 上报 |
| `ContextCompacted` / `ThreadRolledBack` | session/context 状态变化 |
| `Error` / `Warning` | 异常与提示 |

## 仓库目录结构

```
lumi-codex/
├── lumi/                                  # 主包
│   ├── core/                              # 引擎
│   │   ├── thread.py                      # CodexThread（submission_loop）
│   │   ├── session.py                     # SessionState
│   │   ├── delegate.py                    # SubagentDelegate（one-shot + 审批拦截）
│   │   ├── compaction.py                  # 独立 LLM call 摘要
│   │   ├── context.py                     # MessageHistory + approx_token_count
│   │   ├── tool_router.py                 # ToolRouter（统一 ToolSpec + 派发）
│   │   ├── approval.py                    # ApprovalPolicy
│   │   └── event_bus.py                   # asyncio.Queue 包装
│   ├── protocol/                          # 类型化协议
│   │   ├── ops.py                         # Op 变体
│   │   ├── events.py                      # EventMsg 变体
│   │   ├── items.py                       # ResponseItem / RolloutItem
│   │   └── tool_schema.py                 # ToolSpec
│   ├── tools/                             # 工具实现
│   │   ├── shell.py                       # exec_command 等价
│   │   ├── file.py                        # read / write
│   │   ├── apply_patch.py                 # 手写 lenient parser
│   │   ├── search.py                      # ripgrep 包装
│   │   ├── update_plan.py                 # plan tool
│   │   ├── spawn_agent.py                 # one-shot 子 agent tool
│   │   └── registry.py                    # @register_tool 装饰器
│   ├── models/                            # ModelProvider 抽象 + 实现
│   │   ├── base.py                        # ModelProvider ABC + ResponseStream
│   │   ├── fake.py                        # FakeProvider（测试基础）
│   │   ├── openai_compat.py               # OpenAI / Kimi / GLM 共用
│   │   ├── anthropic.py                   # Anthropic native（adapter）
│   │   └── stream.py                      # SSE → AsyncIterator
│   ├── rollout/                           # session 持久化
│   │   ├── writer.py                      # 写 JSONL
│   │   ├── reader.py                      # 反向扫 + replacement_history checkpoint
│   │   └── schema.py                      # RolloutItem 5 种
│   ├── sandbox/                           # 工具能访问哪里
│   │   ├── mode.py                        # SandboxMode 枚举
│   │   ├── cwd_guard.py                   # 写文件 cwd 限制
│   │   └── shell_filter.py                # 命令黑名单
│   └── cli/                               # 终端入口
│       ├── main.py                        # argparse 子命令
│       ├── interactive.py                 # rich + prompt_toolkit 交互
│       └── exec.py                        # headless（benchmark 用）
├── benchmarks/                            # benchmark 适配
│   ├── runner.py                          # 通用 runner
│   ├── humaneval/
│   ├── terminal_bench/
│   └── swebench_lite/
├── demos/
│   └── multi_agent_writer/                # Researcher + Writer + Reviewer demo
├── docs/
│   ├── architecture.md                    # 本文档
│   ├── implementation.md                  # 模块实现细节
│   ├── benchmark.md                       # benchmark + multi-agent demo
│   ├── building.md                        # 构建顺序
│   ├── codex-source-map.md                # codex 源码定位
│   └── learning/                          # 学习产出
├── tests/
│   ├── unit/
│   └── integration/
├── scripts/
├── .env.example
├── pyproject.toml                         # uv
├── README.md
├── CLAUDE.md
└── AGENTS.md
```

## 配置示例

### `~/.lumi/config.toml`

```toml
default_provider = "kimi"

[provider.kimi]
wire_api = "openai_compatible"
base_url = "https://api.moonshot.cn/v1"
model = "kimi-k2-0905-preview"
api_key_env = "MOONSHOT_API_KEY"

[provider.glm]
wire_api = "openai_compatible"
base_url = "https://open.bigmodel.cn/api/paas/v4"
model = "glm-4.6"
api_key_env = "GLM_API_KEY"

[provider.openai]
wire_api = "openai_compatible"
base_url = "https://api.openai.com/v1"
model = "gpt-5"
api_key_env = "OPENAI_API_KEY"

[provider.anthropic]
wire_api = "anthropic_native"
model = "claude-sonnet-4-6"
api_key_env = "ANTHROPIC_API_KEY"
max_tokens = 8192

[provider.fake]
wire_api = "fake"

[approval]
policy = "on_request"   # on_request / unless_trusted / never

[sandbox]
mode = "workspace_write"   # read_only / workspace_write / danger_full_access
cwd_guard = true
shell_blacklist = [
    "rm -rf /",
    "sudo",
    "curl .* \\| sh",
    "wget .* \\| sh",
]

[context]
auto_compact_limit = 100000
compact_user_message_max_tokens = 20000

[session]
sessions_dir = "~/.lumi/sessions"
```

### `.env.example`

```bash
MOONSHOT_API_KEY=sk-...
GLM_API_KEY=...
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```
