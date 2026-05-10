# lumi-codex 设计文档

| 字段 | 值 |
|---|---|
| 作者 | Finn (xujifeng) |
| 初版日期 | 2026-05-09 |
| 修订日期 | 2026-05-10（基于 codex 对照本地源码快照 `c37f743` 的评审） |
| 状态 | Revised v1.1（codex 评审后收敛范围） |
| 阶段目标 | Phase 1：14 天 Python v1 — Codex-core first，benchmark 收敛到 Terminal-Bench subset 硬目标 |
| 后续 | Phase 2（v2，v1 完成后可选）：用 Rust 重写 `core` |

> **重要**：本文档已根据 [`2026-05-10-codex-research-plan-review.md`](./2026-05-10-codex-research-plan-review.md) 中 codex 对照真实源码的评审做了修订。被修订的关键点（compaction 算法、approval/sandbox 拆分、apply_patch 不依赖 Lark、provider 优先级、14 天计划等）以本文档为准；评审文档解释"为什么改"。

---

## 0. 项目背景

一个学习 agent 架构的个人项目，参考主流终端 agent 实现（codex / claude code / openhands / hermes 等），覆盖 Agent Loop、Task 规划与分发、Context 管理、Session 管理、Multi-Agent 编排 5 大能力。

**本项目目标**：

1. 自己构建一个在终端中运行的 mini-codex（命名为 `lumi-codex`），覆盖上述 5 大能力
2. 在主流 benchmark 上跑出合理分数（不是 SOTA，而是"框架本身能跑通"的证明）
3. 通过实现，对 codex 架构形成第一手深入理解

---

## 1. 关键决策汇总（已确认）

| 决策项 | 选择 | 理由 |
|---|---|---|
| **主语言** | Python (v1) | 用户 Rust 0 基础；Python 上手快、benchmark 生态最好；学到的是**架构思想**而非语言细节 |
| **v2 Rust 重写** | 14 天后可选 | 不挤压 v1 完成度；v1 跑通后再用 Rust 重写 `core` crate |
| **Benchmark** | Terminal-Bench (主) + SWE-bench Lite 5-10 题 (展示) + HumanEval (冒烟) | Multi-Agent 用独立 demo 验证，不塞进 benchmark |
| **模型后端** | 多 provider 抽象（Claude / Kimi / GLM / OpenAI） | 学 codex 的 `ModelProvider` 设计精髓；多模型对照跑分 |
| **预算** | ¥300-1000 | 主跑分用便宜模型（Kimi/GLM），关键调试用 Claude |
| **时间分配** | 3 天学习 + 11 天构建 | 问题驱动的源码阅读比从头啃高效 |
| **学习产出** | 7 篇 Markdown 笔记，提交到 `docs/learning/` | v1 完成后可回顾 + 个人知识沉淀 |

---

## 2. Codex 源码深读发现

本项目最初的设计基于二手 research summary，与 codex 真实源码有显著偏差。下表列出**修正后的关键事实**：

| 我们最初以为的 | codex 真实情况 |
|---|---|
| Op / EventMsg 各几个类型 | **当前快照（`c37f743`）`Op` 已超过 30 个变体，`EventMsg` 已超过 70 个变体**。v1 不按数量复刻，按**生命周期**复刻：覆盖 user input → tool call → approval → turn complete → compact → rollback 闭环即可（v1 最小集见 §3.4） |
| Agent loop 是 ReAct 同步推进 | `submission_loop` 是**纯 channel-driven**：`rx_sub.recv()` 阻塞等下一个 `Submission`，handler 派发后台 task。整个引擎是事件总线驱动 |
| 多 provider 各自有 tool 调用格式 | **codex 把所有 tool 统一到 Responses API 格式**，没有 per-provider parser。`ResponsesApiTool { name, description, parameters: JsonSchema, output_schema }` |
| codex 用 tiktoken 计 token | **codex 完全不用 tiktoken**，依赖模型返回的 `TokenUsage` |
| Multi-agent = subagent dispatch | 是 **fork-join + 父级审批拦截**。`agent-graph-store` 只是 parent→child 边的元数据存储；`collaboration-mode-templates` 只是 system prompt 片段（PLAN/EXECUTE/PAIR_PROGRAMMING）；不是图、不是消息总线 |
| `code-mode` crate 是某种"代码模式" | 是 **V8 JavaScript 沙箱**，让 agent 写 JS 调子工具，跟代码编辑无关 |
| 有独立的 Planner 模块 | **没有真 planner**。`plan_tool` 是让模型自己维护 checklist 的 tool（`update_plan`）；`goals` 是外部元数据（SQLite state）；agent 自己决定怎么拆步 |
| Compaction = 简单丢工具输出 | 是 **丢工具输出 + 跑独立 LLM call 摘要**，summary 替换为 fake assistant message；保留结构 `[initial_context] + [user_messages] + [summary]`；`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000` |
| Resume 简单加载 JSONL | **反向扫 rollout 找最近 `Compacted.replacement_history` 作 checkpoint，正向 replay 后缀**；处理 `ThreadRolledBack` 跳过 N 个最新 turn |
| 沙箱是简单的 cwd 限制 | macOS 用 **Seatbelt**（`/usr/bin/sandbox-exec` + `.sbpl` 策略文件），Linux 用独立 **`codex-linux-sandbox` 二进制**（bubblewrap+seccomp+landlock），Windows 用 `windows-sandbox-rs` crate |

**深读结论**：codex 是单 agent + 临时子线程 + 静态元数据扩展（skills/plugins/hooks）的架构，复杂度集中在**审批路由、权限沙箱、工具集成**，不在 agent 协调。

---

## 3. 系统架构

### 3.1 关键架构决策

| 决策 | codex 真实做法 | lumi-codex v1 做法 | 裁剪理由 |
|---|---|---|---|
| **Submission/Event 总线** | tokio `async_channel` (当前快照 cap=512)，`rx_sub.recv()` 驱动主 loop | Python `asyncio.Queue` + `dataclass` Op/Event | codex 的灵魂。1:1 复刻**生命周期**（不是变体数量） |
| **Streaming** | SSE (`eventsource_stream`) → mpsc channel → consumer | `httpx` SSE + `asyncio.Queue` | 必须从第一天支持 |
| **FakeProvider 测试基础** | codex 大量用 mock provider 做单元测试 | v1 **第一步就写 `FakeProvider`**：返回预设 chunks 序列（delta / tool_call / finish_reason），所有 loop/router/rollout/approval 测试都不依赖真实模型 | 不写 fake 后续 debug 会非常痛 |
| **Tool 统一格式** | `ToolSpec` 含 `Function` / `Namespace` / `LocalShell` / `WebSearch` / `Freeform` 等多形态 | v1 **只做 function-style** `ToolSpec(name, description, input_schema, output_schema?)`，由 provider adapter 转 OpenAI / Anthropic 格式 | v1 学习重点是统一 schema + adapter 层，不是覆盖所有 tool 形态 |
| **多 provider** | 单一 `ModelProvider` trait（管 metadata/auth/capability）；实际 stream 走 `ModelClientSession`；built-in 仅 4 个（OpenAI / Bedrock / Ollama / LM Studio） | 单一 `ModelProvider` ABC（管 `stream_complete()`）+ `[provider.X]` TOML 配置 | v1 不做 codex 那么细的 provider/session 拆分 |
| **Provider 优先级** | OpenAI Responses API 是 first-class，第三方靠配置 | **v1 第一优先级：OpenAI-compatible**（覆盖 OpenAI / Kimi / GLM）；**第二优先级：Anthropic native**（tool_use/tool_result shape 增加 adapter 成本，先延后） | 学习重点是 loop + session，不是 provider 覆盖率 |
| **Approval（何时问用户）** | `AskForApproval`：`UnlessTrusted` / `OnFailure`(deprecated) / `OnRequest`(默认) / `Granular` / `Never` | v1 简化为 3 档：`on_request`（默认） / `never` / `unless_trusted` | Granular 太细，v1 先 3 档 |
| **Sandbox（工具实际能访问哪）** | `SandboxPolicy` / `PermissionProfile`：macOS Seatbelt / Linux codex-linux-sandbox / Windows windows-sandbox-rs | v1 简化为 3 档 `SandboxMode`：`read_only` / `workspace_write` / `danger_full_access`，**实现层只做 cwd guard + 命令黑名单 + subprocess 隔离**（不做 OS 沙箱） | OS 沙箱是工程不是学习；明确告诉用户：cwd guard 不是安全沙箱，跑 benchmark 用临时目录或 Docker |
| **Apply patch** | `apply-patch/src/parser.rs` 是**手写 lenient parser**（不是 Lark），兼容 ``` fences、heredoc 包裹等"模型常犯错"的输入 | v1 **不强依赖 Lark**：先实现可用 parser（add/delete/update/move 4 类），再补 lenient cases；parser 与 apply 行为分开测试 | Lark 解析严格但脆，模型输出常常需要宽松解析 |
| **Subagent** | `codex_delegate.rs`：parent `Codex::spawn()` + `forward_events`/`forward_ops`，**审批由父拦截**；当前快照还有 multi-agent v2/AgentControl | v1 **只做 one-shot**：`spawn_agent(task, role_prompt, allowed_tools)` + 父级审批拦截，子 agent `TurnComplete` 后返回 final text | 不做 agent graph store / wait/close 多生命周期 / 并行调度 |
| **Plan** | 不是 planner，是 `update_plan` tool；emit `PlanUpdate` event | 同样：`update_plan` tool + `Plan` 数据结构 | 学到 codex"用工具实现规划"的思想 |
| **Compaction** | 跑独立 LLM call 摘要历史；`build_compacted_history(initial_context, user_messages, summary_text)`；user msgs 从新到旧选取，预算 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`；mid-turn 把 initial context 插到最后一个 user msg 之前 | **v1 简化为：`selected_user_messages (≤20k approx tokens) + assistant_summary`**；先不做 initial_context injection，等 resume 跑通再补 | 第一版只要 `replacement_history` 写入 rollout，resume 就学到核心 |
| **Token 估算** | 真实 usage 用模型返回的 `TokenUsage`；预算/截断用 `approx_token_count`（chars/4 类启发式） | 同上：真实 usage 取 provider 返回；预算用 `approx_token_count`（不用 tiktoken） | 学 codex 的两层做法 |
| **Rollout JSONL** | RolloutItem 5 种：`SessionMeta`/`ResponseItem`/`Compacted{message, replacement_history}`/`TurnContext`/`EventMsg` | 同样 5 种，Python `dataclass` + JSON | session 管理的核心 |
| **Resume** | 反向扫；按 turn segment 跳过被 `ThreadRolledBack` 的；`Compacted.replacement_history` 必须属于"存活 segment"；恢复 `previous_turn_settings` + `reference_context_item`；遇到 legacy compaction 无 `replacement_history` 还要 fallback rebuild | v1 简化版 + **必须覆盖 4 类测试**：① 无 compaction、② 有 compaction、③ 有 rollback、④ compaction + rollback 组合 | 简化算法可以，测试不能省 |
| **Frontend** | InProcessAppServerClient + JSON-RPC 2.0 (app-server-protocol)，CLI/TUI/exec 都消费 ServerNotification | **v1 只做 1 个前端**：rich 渲染的 CLI（同时是 benchmark runner） | 多前端是过度工程；保留"engine + frontend 解耦"的设计原则即可 |

### 3.2 YAGNI 不做清单（v1 明确不做）

基于真实源码深读后确定：

- ❌ **OS 沙箱**（Seatbelt / Landlock / bubblewrap / windows-sandbox）— 用 approval + cwd guard 替代
- ❌ **MCP 客户端 + 服务端**（`rmcp-client` + `mcp-server` + `connectors`）— v1 完全跳过
- ❌ **Hooks 系统**（6 种 hook event）
- ❌ **Skills 系统**（SKILLS.md + 依赖声明）
- ❌ **Plugins**（目录式扩展）
- ❌ **Memories**（Phase1 提取 + Phase2 合并 + 7 天保留）
- ❌ **app-server JSON-RPC 协议** — v1 单前端
- ❌ **TUI**（Ratatui 级别，50+ 文件）— 用 `rich` + `prompt_toolkit`
- ❌ **Goals**（thread 级目标 + token budget）
- ❌ **Code mode**（V8 JS sandbox）
- ❌ **Cloud tasks** / **Realtime conversation** / **Reviews**
- ❌ **Connectors**（Slack/GitHub OAuth 集成）

### 3.3 5 大能力映射

| 能力 | lumi-codex 实现方式 | codex 对应代码 |
|---|---|---|
| **Agent Loop** | `submission_loop` + 通过 tool calls 实现 ReAct | `core/src/session/handlers.rs::submission_loop`、`core/src/session/turn.rs:136` |
| **Task 规划与分发** | `update_plan` tool（让模型自己维护 checklist） | `tools/src/plan_tool.rs:6`、`core/src/tools/handlers/plan.rs:80` |
| **Context 管理** | `MessageHistory` + 独立 LLM call 摘要 compaction（v1 简化版） | `core/src/compact.rs:116/382/462`、`core/src/context_manager/history.rs:262` |
| **Session 管理** | JSONL rollout + 反向扫 resume（含 4 类测试） | `core/src/session/rollout_reconstruction.rs:87/220` |
| **Multi-Agent 编排** | one-shot `SubagentDelegate` fork-join + 父级审批拦截 | `core/src/codex_delegate.rs:65/441` |

### 3.4 v1 最小 Op / EventMsg 生命周期

不按数量复刻，按生命周期复刻。下面是 v1 必须支持的最小集（其他变体直接不实现）。

**v1 最小 `Op`**：

| Op | 用途 |
|---|---|
| `UserInput` / `UserTurn` | 提交用户任务 |
| `ExecApproval` | 回复 shell 审批 |
| `PatchApproval` | 回复 patch 审批 |
| `Interrupt` | 中断当前 turn |
| `Compact` | 手动压缩上下文 |
| `ThreadRollback` | 丢弃最近 N 个用户 turn（可选，但 resume 测试需要） |
| `Shutdown` | 关闭 session |

**v1 最小 `EventMsg`**：

| Event | 用途 |
|---|---|
| `TurnStarted` / `TurnComplete` / `TurnAborted` | turn 生命周期 |
| `AgentMessageDelta` / `AgentMessage` | assistant 输出（流式 + 完整） |
| `UserMessage` | 标识写入模型的输入 |
| `ExecCommandBegin` / `ExecCommandOutputDelta` / `ExecCommandEnd` | shell 工具事件 |
| `ExecApprovalRequest` / `ApplyPatchApprovalRequest` | 审批请求 |
| `PlanUpdate` | 规划工具更新 |
| `TokenCount` | token usage 上报 |
| `ContextCompacted` / `ThreadRolledBack` | session/context 状态变化 |
| `Error` / `Warning` | 异常与提示 |

---

## 4. 仓库目录结构

```
lumi-codex/
├── lumi/                                  # 主包
│   ├── core/                              # 引擎（对应 codex-core 最小集）
│   │   ├── thread.py                      # CodexThread - submission_loop 实现
│   │   ├── session.py                     # SessionState
│   │   ├── delegate.py                    # SubagentDelegate（one-shot + 审批拦截）
│   │   ├── compaction.py                  # v1：user_msgs + summary
│   │   ├── context.py                     # MessageHistory + approx_token_count
│   │   ├── tool_router.py                 # 统一 ToolSpec 格式 + 派发
│   │   ├── approval.py                    # ApprovalPolicy（何时问用户）
│   │   └── event_bus.py                   # asyncio Queue 包装
│   ├── protocol/                          # 类型化协议（精简版）
│   │   ├── ops.py                         # ~8 个 Op 变体
│   │   ├── events.py                      # ~12 个 EventMsg 变体
│   │   ├── items.py                       # ResponseItem / RolloutItem
│   │   └── tool_schema.py                 # ResponsesApiTool 等价物
│   ├── tools/
│   │   ├── shell.py                       # exec_command 等价
│   │   ├── file.py                        # read / write
│   │   ├── apply_patch.py                 # 兼容 codex patch 格式（手写 lenient parser）
│   │   ├── search.py                      # ripgrep 包装
│   │   ├── update_plan.py                 # codex 的 plan tool 复刻
│   │   ├── spawn_agent.py                 # one-shot 子 agent 调度 tool
│   │   └── registry.py                    # @register_tool 装饰器
│   ├── models/
│   │   ├── base.py                        # ModelProvider ABC + ResponseStream
│   │   ├── fake.py                        # FakeProvider — v1 第一步，所有测试基础
│   │   ├── openai_compat.py               # OpenAI / Kimi / GLM 共用（first priority）
│   │   ├── anthropic.py                   # native Messages API（second priority）
│   │   └── stream.py                      # SSE → AsyncIterator 包装
│   ├── rollout/                           # session 持久化
│   │   ├── writer.py                      # 写 JSONL
│   │   ├── reader.py                      # 反向扫 + replacement_history checkpoint（4 类测试）
│   │   └── schema.py                      # RolloutItem 5 种
│   ├── sandbox/                           # 工具实际能访问哪里
│   │   ├── mode.py                        # SandboxMode：read_only/workspace_write/danger_full_access
│   │   ├── cwd_guard.py                   # 写文件 cwd 限制
│   │   └── shell_filter.py                # 命令黑名单（注：不是安全沙箱）
│   └── cli/
│       ├── main.py                        # argparse 子命令
│       ├── interactive.py                 # rich + prompt_toolkit 交互模式
│       └── exec.py                        # headless（benchmark 用）
├── benchmarks/
│   ├── runner.py                          # 通用 runner
│   ├── humaneval/
│   ├── terminal_bench/
│   └── swebench_lite/
├── demos/
│   └── multi_agent_writer/                # Researcher + Writer + Reviewer
├── docs/
│   ├── learning/                          # 7 篇笔记
│   │   ├── 01-codex-workspace-overview.md
│   │   ├── 02-submission-event-bus.md
│   │   ├── 03-context-and-compaction.md
│   │   ├── 04-rollout-and-resume.md
│   │   ├── 05-tool-and-sandbox.md
│   │   ├── 06-multi-agent-delegate.md
│   │   └── 07-codex-vs-claude-code.md
│   └── superpowers/specs/
│       └── 2026-05-09-lumi-codex-design.md   # 本文档
├── tests/
├── scripts/
├── .env.example
├── pyproject.toml                         # uv
├── README.md
└── CLAUDE.md
```

---

## 5. 5 大组件详细设计

### 5.1 Agent Loop（`lumi/core/thread.py`）

复刻 codex 的 `submission_loop` + ReAct 通过 tool calls。

```python
class CodexThread:
    def __init__(self, sess: Session, model: ModelProvider, router: ToolRouter):
        self.sess = sess
        self.model = model
        self.router = router
        self.sub_queue: asyncio.Queue[Submission] = asyncio.Queue()
        self.event_queue: asyncio.Queue[EventMsg] = asyncio.Queue()
    
    async def submission_loop(self):
        """codex 的 submission_loop 等价物。channel-driven，不是 ReAct 循环。"""
        while True:
            sub = await self.sub_queue.get()           # 阻塞等下一个 Submission
            match sub.op:
                case Op.UserInput(items) | Op.UserTurn(items, ...):
                    asyncio.create_task(self._run_turn(sub.id, items))
                case Op.ExecApproval(call_id, decision):
                    await self._handle_approval(call_id, decision)
                case Op.Interrupt():
                    await self._abort_current_turn()
                case Op.Compact():
                    asyncio.create_task(self._run_compact())
                case Op.Shutdown():
                    break
    
    async def _run_turn(self, turn_id: str, items: list[UserInput]):
        """单个 turn：调模型 → 流式输出 → 处理 tool calls → 必要时 loop"""
        await self._emit(EventMsg.TurnStarted(turn_id))
        try:
            self.sess.history.add_user(items)
            
            while True:  # tool-call loop（不是 ReAct 显式 reasoning）
                stream = self.model.stream_complete(
                    messages=self.sess.history.materialize(),
                    tools=self.router.exposed_schemas(),
                )
                tool_calls = []
                async for chunk in stream:
                    if chunk.is_text_delta:
                        await self._emit(EventMsg.AgentMessageDelta(turn_id, chunk.text))
                    elif chunk.is_tool_call:
                        tool_calls.append(chunk.tool_call)
                
                if not tool_calls:
                    break  # 模型给最终答案，没调 tool，turn 结束
                
                for call in tool_calls:
                    decision = await self._approval_check(call)  # 可能 await user
                    if decision.is_reject:
                        result = ToolResult.rejected()
                    else:
                        await self._emit(EventMsg.ExecCommandBegin(call))
                        result = await self.router.execute(call, self.sess)
                        await self._emit(EventMsg.ExecCommandEnd(call, result))
                    self.sess.history.add_tool_result(call, result)
            
            # turn 结束，看是否需要 compact
            if self.sess.history.token_count() > AUTO_COMPACT_LIMIT:
                await self._run_compact()
        
        finally:
            await self._emit(EventMsg.TurnComplete(turn_id))
```

**关键设计点**：
- 主 loop（`submission_loop`）只管派发，不直接做 ReAct
- 每个 turn 在独立 task 里跑，可被 `Op.Interrupt` 中断
- Tool-call 内 loop 是显式的：模型每次返回若有 tool calls 就执行后再调一次模型
- 流式 delta 直接 emit 事件，不缓存

### 5.2 Task 规划（`lumi/tools/update_plan.py`）

复刻 codex 的 `plan_tool`。**让模型自己维护 checklist**，不是独立 LLM planner。

```python
@register_tool(
    name="update_plan",
    description="维护当前任务的 step-by-step checklist。在长任务开始时建议先列出步骤。",
    schema={
        "type": "object",
        "properties": {
            "plan": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "step": {"type": "string"},
                        "status": {"enum": ["pending", "in_progress", "completed"]},
                    },
                },
            },
            "explanation": {"type": "string"},
        },
    },
)
async def update_plan(args, ctx: ToolContext) -> ToolResult:
    ctx.session.plan = args["plan"]
    await ctx.event_bus.emit(EventMsg.PlanUpdate(args["plan"]))
    return ToolResult.success(f"Plan updated: {len(args['plan'])} steps")
```

**System prompt 中加引导**："当用户任务包含 3 个以上独立子任务时，先调 `update_plan` 列出步骤"。

### 5.3 Context 管理 + Compaction（`lumi/core/compaction.py`）

参考 codex `compact.rs` 但**v1 简化**：先做 `selected_user_messages + assistant_summary`，**不做** initial_context injection（v2 再补）。**关键**：只要 `replacement_history` 写入 rollout，resume 就能学到 codex 最有价值的机制。

```python
COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000
AUTO_COMPACT_LIMIT = 100_000  # 按模型 context window 调整

async def run_compact(sess: Session, model: ModelProvider) -> CompactedItem:
    """v1 简化版（codex compact.rs 的最小子集）。"""
    # 1. 从新到旧选取 user messages，总预算 ≤ 20k approx tokens（用 chars/4 启发式）
    user_msgs = sess.history.collect_user_messages_newest_first()
    selected = []
    budget = 0
    for msg in user_msgs:
        msg_tokens = approx_token_count(msg)  # chars/4
        if budget + msg_tokens > COMPACT_USER_MESSAGE_MAX_TOKENS:
            # 必要时截断最老的一个（codex 也这么做）
            if not selected:
                selected.append(truncate_msg(msg, COMPACT_USER_MESSAGE_MAX_TOKENS))
            break
        selected.append(msg)
        budget += msg_tokens
    selected.reverse()  # 恢复时间顺序

    # 2. 调独立 LLM call 摘要（不带 tools）
    compact_prompt = load_compact_prompt()  # 类比 codex /core/templates/compact/prompt.md
    summary_stream = model.stream_complete(
        messages=[
            {"role": "system", "content": compact_prompt},
            *sess.history.materialize(),  # 完整 history 给摘要器看
            {"role": "user", "content": "请按上面的指令对历史做摘要。"},
        ],
        tools=[],
    )
    summary_text = ""
    async for chunk in summary_stream:
        summary_text += chunk.text or ""

    # 3. 构造 replacement_history（v1：user msgs + summary as fake assistant）
    #    v2 补：在 mid-turn compaction 时把 initial context 插到最后一个 user msg 之前
    new_history = [
        *selected,
        AssistantMessage(content=summary_text),
    ]

    # 4. 写 RolloutItem.Compacted（resume 靠它）
    compacted = CompactedItem(message=summary_text, replacement_history=new_history)
    await sess.rollout_writer.write(RolloutItem.Compacted(compacted))
    await sess.event_bus.emit(EventMsg.ContextCompacted(...))

    sess.history.replace(new_history)
    return compacted
```

**触发点**：
- `_run_turn` 结尾检查 token usage > `AUTO_COMPACT_LIMIT`
- 用户 `Op.Compact` 命令手动触发

**v1 留白**（不做）：
- `initial_context` injection（codex 的 mid-turn 行为）
- `manual / pre-turn / mid-turn` 三种触发的差异化处理
- `compact_remote.rs` 走 model 端点的远程压缩

**测试要求**：用 `FakeProvider` 模拟 summarizer 返回固定文本，断言 `replacement_history` 结构正确、`CompactedItem` 写入 rollout、`history.materialize()` 反映 replacement。

### 5.4 Session 管理：Rollout + Resume（`lumi/rollout/`）

#### Schema（`lumi/rollout/schema.py`）

```python
@dataclass
class SessionMeta:
    id: str
    timestamp: str
    source: str
    model_provider: str
    git_info: dict | None

@dataclass
class TurnContextItem:
    cwd: str
    approval_policy: str
    sandbox_policy: str
    model: str

@dataclass
class CompactedItem:
    message: str
    replacement_history: list[ResponseItem]  # 完整 history checkpoint！

# 5 种 RolloutItem
RolloutItem = SessionMeta | ResponseItem | CompactedItem | TurnContextItem | EventMsg
```

#### Writer（`lumi/rollout/writer.py`）

每个 item 一行 JSON，写到 `~/.lumi/sessions/rollout-<ts>-<uuid>.jsonl`。

#### Reader（`lumi/rollout/reader.py`）—— 简化版反向扫 + 4 类必测场景

codex 的真实 reader 比下面这版复杂（要按 turn segment 处理 rollback、处理 legacy compaction 没 `replacement_history` 时的 fallback rebuild、恢复 `previous_turn_settings` 和 `reference_context_item`）。**v1 简化**到下面，但**测试必须覆盖 4 类场景**：

```python
def reconstruct_history(rollout_path: Path) -> ReconstructedSession:
    """v1 简化版。"""
    items = read_jsonl(rollout_path)

    # 反向扫
    checkpoint_idx = None
    rolled_back_count = 0
    for i in reversed(range(len(items))):
        item = items[i]
        if isinstance(item, EventMsg.ThreadRolledBack):
            rolled_back_count += item.turns
            continue
        if isinstance(item, CompactedItem) and item.replacement_history:
            checkpoint_idx = i
            break

    # 取 checkpoint history
    if checkpoint_idx is not None:
        history = items[checkpoint_idx].replacement_history.copy()
        suffix = items[checkpoint_idx + 1:]
    else:
        history = []
        suffix = items

    # 正向 replay 后缀（处理 ResponseItem，跳过 EventMsg 等）
    for item in suffix:
        if isinstance(item, ResponseItem):
            history.append(item)

    # 跳过最新 N 个 user turn（被 ThreadRolledBack 取消的）
    if rolled_back_count > 0:
        history = drop_latest_user_turns(history, rolled_back_count)

    return ReconstructedSession(history=history)
```

**4 类必测场景**（`tests/integration/test_resume.py`）：

| # | 场景 | 期望行为 |
|---|---|---|
| 1 | **无 compaction** | replay 所有 `ResponseItem`；history 与原 turn 一致 |
| 2 | **有 compaction** | 使用最近的 `replacement_history` + suffix；token 数应显著小于场景 1 |
| 3 | **有 rollback**（无 compaction） | 丢弃最近 N 个 user turn 及其 assistant 回应 |
| 4 | **compaction + rollback 组合** | rollback 不能错误"复活"被 rollback 的历史；checkpoint 必须属于存活 segment |

**v1 留白**：
- `previous_turn_settings` / `reference_context_item` 恢复（v2 加，做 fork 时需要）
- legacy compaction 无 `replacement_history` 时的 fallback rebuild

CLI：`lumi resume <session_id>` 续会；`lumi replay <session_id>` 重放（debug）。

### 5.5 Multi-Agent：SubagentDelegate（`lumi/core/delegate.py`）

复刻 codex 的 fork-join + 审批拦截。

```python
class SubagentDelegate:
    """父 agent 派生子 agent，桥接事件流，拦截子 agent 的审批请求。"""
    
    @staticmethod
    async def spawn_interactive(
        parent_sess: Session,
        config: SubagentConfig,
    ) -> tuple[asyncio.Queue, asyncio.Queue]:  # (events_rx, ops_tx)
        # 1. 创建子 Codex 实例（独立 session、独立 history、受限 tool 集）
        sub_sess = Session.create_subagent(
            parent=parent_sess,
            tools=config.allowed_tools,
            system_prompt=config.system_prompt,
        )
        sub_thread = CodexThread(sub_sess, parent_sess.model, parent_sess.router)
        
        # 2. 启动子 agent 的 submission_loop
        asyncio.create_task(sub_thread.submission_loop())
        
        # 3. forward_events: 子 agent 的事件流，过滤审批请求
        events_out: asyncio.Queue = asyncio.Queue()
        async def forward_events():
            while True:
                event = await sub_thread.event_queue.get()
                if isinstance(event, EventMsg.ExecApprovalRequest):
                    # 审批拦截！由父 agent 的 policy 处理，不冒泡到上层 caller
                    decision = await parent_sess.handle_approval(event)
                    await sub_thread.sub_queue.put(
                        Submission(id=new_id(), op=Op.ExecApproval(event.id, decision))
                    )
                else:
                    await events_out.put(event)
        asyncio.create_task(forward_events())
        
        # 4. forward_ops: caller 提交的 op 转发给子 agent
        ops_in: asyncio.Queue = asyncio.Queue()
        async def forward_ops():
            while True:
                op = await ops_in.get()
                await sub_thread.sub_queue.put(Submission(id=new_id(), op=op))
        asyncio.create_task(forward_ops())
        
        return events_out, ops_in
    
    @staticmethod
    async def spawn_one_shot(
        parent_sess: Session,
        config: SubagentConfig,
        user_input: str,
    ) -> str:
        """简化版：提交单条输入，等 TurnComplete，关闭。"""
        events_rx, ops_tx = await SubagentDelegate.spawn_interactive(parent_sess, config)
        await ops_tx.put(Op.UserInput([UserInput.text(user_input)]))
        
        final_text = ""
        while True:
            event = await events_rx.get()
            if isinstance(event, EventMsg.AgentMessage):
                final_text = event.content
            elif isinstance(event, EventMsg.TurnComplete):
                break
        return final_text
```

**配套 tool**（`lumi/tools/spawn_agent.py`）：

```python
@register_tool(
    name="spawn_agent",
    description="派生子 agent 处理特定子任务。子 agent 独立 context，返回最终答案。",
)
async def spawn_agent(args, ctx: ToolContext) -> ToolResult:
    config = SubagentConfig(
        system_prompt=args["role_prompt"],
        allowed_tools=args.get("tools", ["read_file", "search"]),
    )
    result = await SubagentDelegate.spawn_one_shot(ctx.session, config, args["task"])
    return ToolResult.success(result)
```

---

## 6. Tool 系统与沙箱

### 6.1 Tool 注册与统一格式

```python
# lumi/protocol/tool_schema.py
@dataclass
class ResponsesApiTool:
    name: str
    description: str
    parameters: dict           # JSON Schema
    output_schema: dict | None = None
    strict: bool = False

# lumi/tools/registry.py
_REGISTRY: dict[str, ToolHandler] = {}

def register_tool(name: str, description: str, schema: dict):
    def decorator(fn):
        _REGISTRY[name] = ToolHandler(name, description, schema, fn)
        return fn
    return decorator

class ToolRouter:
    def exposed_schemas(self) -> list[ResponsesApiTool]:
        return [t.to_responses_api() for t in _REGISTRY.values()]
    
    async def execute(self, call: ToolCall, sess: Session) -> ToolResult:
        handler = _REGISTRY[call.name]
        ctx = ToolContext(session=sess, ...)
        return await handler.fn(call.args, ctx)
```

### 6.2 Apply Patch（`lumi/tools/apply_patch.py`）—— 兼容 codex patch 格式，**手写 lenient parser**

**重要纠正**：codex repo 里的 `tools/src/tool_apply_patch.lark` 是 grammar **spec**，但实际 `apply-patch/src/parser.rs` 是**手写 lenient parser**，专门兼容模型把 heredoc 当参数、外面包 ``` fences 等真实输入。**v1 不强依赖 Lark**：目标是兼容 codex patch format，不是用 Lark 证明正确。

格式：

```
*** Begin Patch
*** Update File: path/to/file.py
@@ def some_function(): @@
-    old_line
+    new_line
*** End Patch
```

**实现顺序**（D6 必交付前 4 项）：

1. **Add File**：解析 `*** Add File: <path>` + `+` 前缀的全部内容行
2. **Delete File**：`*** Delete File: <path>`
3. **Update File**：`@@ context @@` + `-/+` 行；context 用于在文件中定位
4. **Move File**：`*** Update File: ...` 之后跟 `*** Move to: <new_path>`
5. **Lenient parsing**（D6 时间充裕再加，否则进 stretch）：去掉 ``` fences、heredoc wrapper、前后空白、混合 LF/CRLF

**测试要求**：parser（解析正确性）和 apply（实际写文件后内容正确）**分开测试**。每类 hunk 至少 3 个 case：正常、含特殊字符、含 lenient 前缀。

源码定位：`_research/codex/codex-rs/apply-patch/src/parser.rs:1`

### 6.3 ApprovalPolicy（`lumi/core/approval.py`）—— **何时问用户**

codex 把"何时问用户"和"工具实际能访问哪里"是**两个独立概念**，原计划混淆了。v1 必须拆开：

```python
class ApprovalPolicy(Enum):
    ON_REQUEST = "on_request"      # 默认：模型决定何时请求审批
    UNLESS_TRUSTED = "unless_trusted"  # 只有"已知安全"的命令免审批
    NEVER = "never"                # 不问用户，一律按 SandboxMode 决策

@dataclass
class ApprovalDecision:
    approved: bool
    reason: str = ""

class ApprovalChecker:
    def __init__(self, policy: ApprovalPolicy):
        self.policy = policy

    async def check(self, call: ToolCall, ctx: ToolContext) -> ApprovalDecision:
        if self.policy == ApprovalPolicy.NEVER:
            return ApprovalDecision(True)
        if self.policy == ApprovalPolicy.UNLESS_TRUSTED:
            if not is_trusted_command(call):
                return await ctx.request_user_approval(call, "not in trust list")
        # ON_REQUEST: 由 SandboxMode 检查决定是否需审批，由 ToolRouter 调 emit
        return ApprovalDecision(True)
```

### 6.4 SandboxMode（`lumi/sandbox/mode.py`）—— **工具能访问哪里**

```python
class SandboxMode(Enum):
    READ_ONLY = "read_only"             # 不允许写、不允许 exec
    WORKSPACE_WRITE = "workspace_write" # 默认：cwd 内可写，shell 命令过黑名单
    DANGER_FULL_ACCESS = "danger_full_access"  # 全开（仅在 Docker/临时目录用）
```

实现层（v1 简化，**不是真沙箱**）：

- **cwd guard** (`sandbox/cwd_guard.py`)：所有写文件操作必须落在 `--cwd` 子树内
- **shell 黑名单** (`sandbox/shell_filter.py`)：正则匹配（`rm -rf`、`sudo`、绝对 path 出 cwd 等）触发 approval（如果 ApprovalPolicy 允许的话）
- **subprocess 隔离**：`asyncio.create_subprocess_exec`，不用 `shell=True`

> ⚠️ **明确告诉用户**：cwd guard + 黑名单**不是安全沙箱**，只是学习项目里的防误操作机制。**跑 benchmark 时要在临时目录或 Docker workspace 里运行**。
>
> codex 真实沙箱（v2 可加）：macOS Seatbelt / Linux codex-linux-sandbox / Windows windows-sandbox-rs。详见 learning note 05。

---

## 7. ModelProvider 多 provider 设计

### 7.1 ABC（`lumi/models/base.py`）

```python
@dataclass
class ChatChunk:
    text: str | None = None
    tool_call: ToolCall | None = None
    finish_reason: str | None = None
    usage: TokenUsage | None = None

class ModelProvider(ABC):
    @abstractmethod
    async def stream_complete(
        self,
        messages: list[Message],
        tools: list[ToolSpec],
        **kwargs,
    ) -> AsyncIterator[ChatChunk]:
        ...

    @abstractmethod
    def model_name(self) -> str: ...

    def supports_tools(self) -> bool:
        return True
```

### 7.2 实现优先级（**修订**：OpenAI-compatible first）

按 codex 评审建议，v1 学习重点是 loop + session，不是 provider 覆盖率。

| 优先级 | Provider | 接口 | 何时实现 |
|---|---|---|---|
| **0（首先）** | `FakeProvider` | 返回预设 chunks 序列 | **D2 第一步**。所有 loop/router/rollout/approval/compaction 测试都依赖它，**不能延后** |
| **1（first-class）** | `OpenAICompatProvider` | OpenAI Chat Completions / Responses API；**一份代码覆盖 OpenAI / Kimi / GLM**（差异在 base_url + model name） | **D3** |
| **2（second-class）** | `AnthropicProvider` | native Messages API + tool_use/tool_result blocks；需要 adapter 把统一 `ToolSpec` 转 native shape | **冲刺项**（Day 13 之后或 v1.5） |
| **延后** | Bedrock / Ollama / LM Studio | 不是 v1 范围 | v2 |

**关键点**：tool 调用差异留在 **provider adapter 层**，不要让每个 tool 写多个 provider 版本。

### 7.3 配置 schema（`~/.lumi/config.toml`）

```toml
# v1 默认主用 OpenAI-compatible（成本可控）
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

# 冲刺项，v1 后期或 v1.5 实现
# [provider.anthropic]
# wire_api = "anthropic_native"
# model = "claude-sonnet-4-6"
# api_key_env = "ANTHROPIC_API_KEY"
# max_tokens = 8192

[approval]
policy = "on_request"   # on_request / unless_trusted / never

[sandbox]
mode = "workspace_write"   # read_only / workspace_write / danger_full_access
cwd_guard = true
shell_blacklist = ["rm -rf /", "sudo", "curl .* | sh"]
```

CLI：`lumi --provider kimi exec "改 auth.py"`。

---

## 8. CLI 设计（`lumi/cli/`）

### 8.1 子命令

| 命令 | 用途 |
|---|---|
| `lumi` | 启动交互模式 |
| `lumi exec <prompt>` | headless 单次执行（benchmark 用） |
| `lumi resume <session_id>` | 续会 |
| `lumi replay <session_id>` | 离线重放 rollout（debug） |
| `lumi sessions list` | 列出本地 sessions |
| `lumi config show / set` | 配置管理 |

### 8.2 交互模式（`lumi/cli/interactive.py`）

- `prompt_toolkit` 读用户输入（多行、history、autocomplete）
- `rich` 渲染 EventMsg 流（彩色输出、code block 高亮、stream cursor）
- 审批用 inline prompt：`>> 是否允许执行 'rm xxx'？(y/n/always)`

### 8.3 Headless exec（`lumi/cli/exec.py`）

- 单次输入（args 或 stdin），完成后退出
- 默认输出：最终答案到 stdout，事件流到 stderr
- `--json` 模式：所有事件序列化为 JSONL 输出 stdout（benchmark runner 解析用）

---

## 9. Benchmark 集成方案

### 9.1 通用 Runner（`benchmarks/runner.py`）

```python
class BenchmarkRunner:
    """通用框架：读 task → drive lumi → grade → 汇总。"""
    
    def __init__(self, name: str, task_loader, grader):
        self.name = name
        self.load_tasks = task_loader
        self.grade = grader
    
    async def run(self, provider: str, n_tasks: int | None = None, parallel: int = 4):
        tasks = self.load_tasks()[:n_tasks]
        sem = asyncio.Semaphore(parallel)
        
        async def run_one(task):
            async with sem:
                result = await run_lumi_exec(
                    prompt=task.prompt,
                    cwd=task.workspace,
                    provider=provider,
                )
                score = self.grade(task, result)
                return TaskResult(task.id, score, result.usage)
        
        results = await asyncio.gather(*(run_one(t) for t in tasks))
        return self.aggregate(results)
```

### 9.2 HumanEval smoke（**只跑 20 题**）

按 codex 评审：HumanEval 主要测**模型能力**，不太能证明 agent 架构。**只跑 20 题作 smoke test**，目的是验证：

- headless runner 能批量驱动
- workspace 创建 / 切换正常
- 测试执行（`pytest`）和结果汇总链路通

不追求高分。

### 9.3 Terminal-Bench subset（**v1 主线，硬目标**）

**v1 必须交付**：固定 **10 题 subset**，每题记录：

- task id
- provider/model
- pass/fail
- 总 token / shell 次数 / apply_patch 次数
- 失败原因分类：模型能力 / 工具不够用 / 上下文管理差 / provider 问题 / 环境问题

**交付重点不是高分**，而是能在 README 里说清楚：

> "哪些任务失败是模型能力问题，哪些是工具/上下文/session 问题。"

### 9.4 Terminal-Bench 全量（冲刺项）

D12 后若时间允许跑全 ~89 题；否则保持 10 题 subset 即可。

### 9.5 SWE-bench Lite（冲刺项，**只承诺 1-3 题**）

**修订**：原计划 5-10 题过于乐观。SWE-bench 环境成本高（Docker、镜像下载、测试套件）+ 强依赖 apply_patch / repo search / test selection 多组件协作。v1 **只承诺 1-3 道非常小的题作为 demo**，证明 lumi 能处理真实 patch 工作流即可。

### 9.6 模型对照跑分（应该交付）

D13 用同一套 Terminal-Bench subset 跑 Kimi / GLM / OpenAI（任两个），对照表展示：

| 模型 | Terminal-Bench subset 通过率 | 平均 token | 总成本 |
|---|---|---|---|
| kimi-k2 | ?% | ? | ¥? |
| glm-4.6 | ?% | ? | ¥? |
| gpt-5（如果 budget 允许） | ?% | ? | ¥? |
| claude-sonnet-4-6（冲刺：实现 Anthropic provider 后） | ?% | ? | ¥? |

---

## 10. Multi-Agent Demo（`demos/multi_agent_writer/`）

**任务**：用户输入"写一篇关于 X 的技术博客"，生成完整 Markdown。

**Pipeline**：

```
User input
   ↓
[Main Agent]
   ↓
   ├─ spawn_agent(role="researcher", task="搜索 X 相关资料，输出结构化 outline")
   │   └─ Researcher 用 web search + read_file tools，独立 context
   │   ←  返回 outline JSON
   ↓
   ├─ spawn_agent(role="writer", task="基于 outline 写完整 blog")
   │   └─ Writer 用 outline + 自己的语言能力写
   │   ←  返回 markdown
   ↓
   ├─ spawn_agent(role="reviewer", task="审稿，给改进建议")
   │   └─ Reviewer 读 markdown，给 list of issues
   │   ←  返回建议
   ↓
   └─ Main Agent 整合：让 writer 应用建议，输出最终 blog
```

**录 asciinema gif**，README 里展示 multi-agent 协作过程。**不参与 benchmark 跑分**。

---

## 11. 14 天时间表（修订版，按 codex 评审）

**核心改动**：从原"3 天学 + 11 天建"改为 **Codex-core first**：D1 调研评审 → D2 protocol/FakeProvider → D3 真实 provider → D4-D6 tool 闭环 → D7-D8 rollout/resume → D9 compaction → D10-D12 benchmark → D13 multi-agent → D14 收尾。**学习笔记 7 篇分散在构建过程中产出**（每个组件做完时配套写）。

**起始**：2026-05-09（D0 设计文档）
**评审**：2026-05-10（D1 codex 评审 + 计划修订）
**结束**：2026-05-23（D14 收尾，预留 buffer）

### Phase 1：交付范围（codex 评审给出的 3 档）

#### 4.1 必须交付（v1 硬目标）

1. `lumi exec "<task>"` 可运行真实模型
2. `lumi` 交互模式可连续多轮输入
3. tool-call loop 支持 shell/read/write/search/update_plan/apply_patch
4. JSONL rollout 持久化，`lumi resume <session_id>` 可续会
5. apply_patch 能处理常见新增、删除、更新、移动文件
6. ApprovalPolicy + cwd guard 防止明显危险操作
7. Terminal-Bench 10 题 subset 可以跑，并输出报告
8. README 能解释 5 大能力分别如何实现

#### 4.2 应该交付（应该有，否则缺一块拼图）

1. compaction：手动 `/compact` + token 超限前自动 compact
2. `spawn_agent` one-shot 子 agent demo（multi-agent writer）
3. Kimi/GLM/OpenAI 任一 OpenAI-compatible provider 可通过配置切换（v1 计划全做）
4. HumanEval smoke 20 题，验证 headless runner 稳定

#### 4.3 冲刺交付（时间充裕再做）

1. Terminal-Bench 全量 ~89 题
2. SWE-bench Lite 1-3 题
3. Anthropic native provider
4. asciinema demo
5. 更完整的 multi-agent lifecycle：wait/resume/close

### Phase 2：Day-by-day 计划

| Day | 日期 | 任务 | 验收标准 |
|---|---|---|---|
| **D0** | 5/09 | 设计文档 v1 + codex-rs 深度调研（已完成） | 设计文档 push GitHub |
| **D1** | 5/10 | codex 对照源码评审，本设计文档修订 v1.1（今天完成） | 评审文档 + 设计文档修订 push |
| **D2** | 5/11 | protocol（最小 Op/Event）+ `EventBus` + `FakeProvider` + 基础 tests | FakeProvider 能驱动一个 `UserInput → AgentMessage → TurnComplete`；事件流测试通过 |
| **D3** | 5/12 | `OpenAICompatProvider` + headless `lumi exec` | `lumi exec "say hi"` 调通真实 Kimi/GLM；流式 delta 能渲染 |
| **D4** | 5/13 | `ToolRouter` + `shell`/`read_file`/`write_file`/`search`/`update_plan` tools | 模型能列目录、读文件、写文件、更新 plan |
| **D5** | 5/14 | `_run_turn` tool-call continuation + `ApprovalPolicy` + `SandboxMode` 拆分 | 工具调用后能继续向模型回传 result；危险 shell 会被拦截 |
| **D6** | 5/15 | `apply_patch` parser/apply（手写 lenient）+ 单元测试 | add/delete/update/move 4 类 patch 测试通过 |
| **D7** | 5/16 | `RolloutWriter` + session list/replay | 每个 turn 写 JSONL；能离线重放事件和 response items |
| **D8** | 5/17 | `RolloutReader` + resume + 4 类测试 | 无 compact / 有 compact / 有 rollback / compact+rollback 4 种场景都能正确 resume |
| **D9** | 5/18 | `compaction.py` v1（FakeProvider 测试 + 真实 manual `/compact`） | fake summarizer 测试通过；真实模型手动 compact 可用 |
| **D10** | 5/19 | HumanEval smoke 20 题 + benchmark runner 基础设施 | 20 题可批量跑，失败能落日志、能聚合报告 |
| **D11** | 5/20 | Terminal-Bench subset 10 题接入 + 第一次跑 | 10 题 subset 跑完，至少产出 pass/fail 报告 |
| **D12** | 5/21 | 分析 D11 失败 case + 修暴露的问题 + 二次跑 | 失败按模型/工具/上下文/provider/prompt/环境分类 |
| **D13** | 5/22 | `spawn_agent` one-shot + multi-agent demo + 模型对照跑分 | Researcher/Writer/Reviewer demo 能完成一次；2 个 provider 跑同一 subset |
| **D14** | 5/23 | README、笔记 07、结果报告、收尾 | README 展示架构、用法、benchmark 表、后续计划；7 篇笔记齐 |

**学习笔记节奏**（与开发并行，不再集中前 3 天）：

| Note | 写作时机 |
|---|---|
| 01 codex-rs Workspace 总览 | D1 评审完成时（已具备） |
| 02 Submission/Event 总线 | D2 完成 protocol + FakeProvider 后 |
| 03 Context 与 Compaction | D9 完成 compaction 后 |
| 04 Rollout 与 Resume | D8 完成 resume 后 |
| 05 Tool 与 Sandbox | D5-D6 完成后 |
| 06 Multi-Agent (codex_delegate) | D13 完成 spawn_agent 后 |
| 07 codex vs Claude Code | D14 收尾时 |

---

## 12. 学习笔记 7 篇大纲

每篇目标：自己能把这一块讲清楚。

| # | 标题 | 核心内容 |
|---|---|---|
| 01 | **codex-rs Workspace 总览** | 100+ crate 分类（核心/扩展/cloud/外围）；阅读路径推荐；为什么 Rust 而不是 TS |
| 02 | **Submission/Event 总线设计** | submission_loop 主循环；Op/EventMsg 全部变体分类；为什么 channel-driven 而不是 request-response；async task 派发模式 |
| 03 | **Context 管理与 Compaction** | inline vs remote compaction；`build_compacted_history` 三段结构（initial / user_msgs / summary）；`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`；为什么用 fake assistant message 存 summary；mid-turn 注入 initial context 的细节 |
| 04 | **Rollout 与 Resume** | RolloutItem 5 种；反向扫找 checkpoint 的精妙；与 SQLite state 的分工；ThreadRolledBack 处理 |
| 05 | **Tool 系统与 Sandbox** | `ToolSpec` 多形态；`AskForApproval` vs `SandboxPolicy` 两层概念分离；Seatbelt vs Landlock vs Windows sandbox 实现差异；apply_patch 手写 lenient parser（不是 Lark）|
| 06 | **Multi-Agent (codex_delegate)** | fork-join 模式；父级审批拦截的实现细节；agent-graph-store 的真实作用；与 Claude Code subagent 对比 |
| 07 | **codex vs Claude Code** | Hooks/Skills/Plugins/MCP 设计差异；多 agent 模型差异；给后续做技术选型的参考 |

---

## 13. v2 Rust 重写计划（14 天后，可选）

**范围**：仅重写 `core` crate（约 8 个模块），其他保持 Python。

**学习目标**：
1. 实战 Rust async（tokio mpsc、Arc/Mutex、async traits）
2. 复刻 codex 的关键代码片段
3. 体验 Rust 单进程 binary 部署的优势

**阶段**：
- Week 1：Rust 基础 + tokio + serde；移植 `protocol/` 到 Rust
- Week 2：移植 `core/thread.rs` + `event_bus.rs`
- Week 3：移植 `compaction.rs` + `rollout/`
- Week 4：性能对比 + 写笔记 08（Python vs Rust 实现感受）

**不重写**：tools / models / cli / benchmarks（Python 已足够）。

---

## 14. 风险与应对（修订）

| 风险 | 概率 | 应对 |
|---|---|---|
| **不写 FakeProvider，直接绑真实模型** | 高 | D2 第一步就是 FakeProvider；后续所有 loop/router/rollout/approval/compaction 测试都用 fake，不烧 token、不依赖网络 |
| Apply patch parser 复杂，Day 6 完不成 | 中 | 拆 5 步：先 add/delete/update/move 4 类（必须），lenient parsing 进 stretch；parser 与 apply 行为分开测试 |
| Resume 算法漏 case（rollback + compaction 组合） | 中 | 4 类必测场景 + FakeProvider 构造测试数据，不靠真实 session 验证 |
| Compaction 一步到位失败（initial_context injection 复杂） | 中 | v1 只做 `user_msgs + summary`；只要 `replacement_history` 写入 rollout，resume 就拿到核心 |
| Terminal-Bench 通过率低（< 20%） | 中 | 不追求高分；按模型/工具/上下文/provider/prompt/环境分类失败原因，**失败分析本身就是产出** |
| 模型成本超预算 | 低 | 默认用 Kimi/GLM；只在调试用真实模型时才用更贵的 |
| SWE-bench Docker 配置卡住 | 高 | SWE-bench 已经从必做下调到冲刺项；卡住直接跳过，不影响 v1 验收 |
| Multi-agent demo 不稳定 | 中 | demo 失败可接受，关键是 `SubagentDelegate` 单元测试覆盖 fork-join + 审批拦截路径 |
| Anthropic provider adapter 复杂（tool_use shape 不同） | 中 | 已经从 D-day 必做改为冲刺项；优先 OpenAI-compatible（一份代码覆盖 OpenAI/Kimi/GLM） |
| 时间不够 | 中 | 优先级（codex 评审后修订）：Agent loop + tool 闭环 > rollout/resume > compaction > Terminal-Bench subset > multi-agent demo > 全量 benchmark > Anthropic provider |

---

## Appendix A: Codex 关键源码定位（深读时引用）

### 主循环与协议
- `codex-rs/core/src/session/handlers.rs:968` — `submission_loop` 主循环
- `codex-rs/core/src/session/turn.rs:136` — `run_turn` 实现
- `codex-rs/core/src/session/turn.rs:148` — `auto_compact_limit` 定义
- `codex-rs/core/src/session/turn.rs:379` — 自动 compact 触发点
- `codex-rs/protocol/src/protocol.rs:405` — `Op` enum 全部变体
- `codex-rs/protocol/src/protocol.rs:1314` — `EventMsg` enum 全部变体
- `codex-rs/protocol/src/protocol.rs:941` — `AskForApproval` enum

### Tool 系统
- `codex-rs/tools/src/tool_spec.rs:22` — `ToolSpec` enum
- `codex-rs/tools/src/responses_api.rs:25` — `ResponsesApiTool` 结构
- `codex-rs/core/src/tools/router.rs:39` — `ToolRouter` 结构
- `codex-rs/apply-patch/src/parser.rs:126` — `parse_patch` 入口
- `codex-rs/tools/src/tool_apply_patch.lark` — Lark grammar
- `codex-rs/protocol/src/plan_tool.rs` — `update_plan` tool 定义

### Context & Rollout
- `codex-rs/core/src/compact.rs:44` — `COMPACT_USER_MESSAGE_MAX_TOKENS`
- `codex-rs/core/src/compact.rs:116` — inline compaction 主流程
- `codex-rs/core/src/compact.rs:256` — `collect_user_messages`
- `codex-rs/core/src/compact_remote.rs:113` — remote compaction
- `codex-rs/core/src/session/rollout_reconstruction.rs:87` — `reconstruct_history_from_rollout`

### Sandbox
- `codex-rs/sandboxing/src/manager.rs:23` — `SandboxType` enum
- `codex-rs/sandboxing/src/manager.rs:196` — Seatbelt 调用
- `codex-rs/core/src/landlock.rs:22` — Linux 沙箱调用

### Multi-agent
- `codex-rs/core/src/codex_delegate.rs:65` — `run_codex_thread_interactive`
- `codex-rs/core/src/codex_delegate.rs:163` — `run_codex_thread_one_shot`
- `codex-rs/core/src/codex_delegate.rs:284` — 审批拦截 `handle_exec_approval`
- `codex-rs/agent-graph-store/src/types.rs` — parent→child 边定义
- `codex-rs/collaboration-mode-templates/src/lib.rs` — 协作模式 prompt 模板

### Model
- `codex-rs/model-provider/src/provider.rs:79` — `ModelProvider` trait
- `codex-rs/model-provider-info/src/lib.rs:402` — `built_in_model_providers`
- `codex-rs/core/src/client_common.rs:177` — `ResponseStream` 类型
- `codex-rs/codex-api/src/sse/responses.rs:63` — `spawn_response_stream`

### Frontend
- `codex-rs/cli/src/main.rs:733` — CLI main 入口
- `codex-rs/exec/src/lib.rs:221` — exec 子命令实现
- `codex-rs/tui/src/app.rs:1020` — TUI 主 event loop（select! over 4 channels）

---

## Appendix B: 配置示例

### `~/.lumi/config.toml`

```toml
# v1 默认主用 OpenAI-compatible（Kimi/GLM 成本可控）
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

# 冲刺项（v1 后期或 v1.5 实现）
# [provider.anthropic]
# wire_api = "anthropic_native"
# model = "claude-sonnet-4-6"
# api_key_env = "ANTHROPIC_API_KEY"
# max_tokens = 8192

# v1 测试基础（不需要 key）
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
auto_compact_limit = 100000          # 触发 compaction 的 token 阈值
compact_user_message_max_tokens = 20000

[session]
sessions_dir = "~/.lumi/sessions"
```

### `.env.example`

```bash
# v1 必填（任一即可启动）
MOONSHOT_API_KEY=sk-...
GLM_API_KEY=...

# v1 选填
OPENAI_API_KEY=sk-...

# 冲刺项（实现 anthropic_native provider 后用）
# ANTHROPIC_API_KEY=sk-ant-...
```

---

## 附：本设计文档的状态

**v1.0**（2026-05-09）：基于 codex-rs 源码 + 6 个并行 research agent 的初版设计
**v1.1**（2026-05-10）：基于 codex 对照本地源码快照 `c37f743` 的评审做了 10 项修订（详见评审文档 §8）

| 节 | 状态 | 评审修订点 |
|---|---|---|
| §0 背景 | 稳定 | — |
| §1 关键决策 | 稳定 | — |
| §2 Codex 源码发现 | 已修订 v1.1 | Op/Event 计数 → 生命周期复刻 |
| §3 系统架构 | 已修订 v1.1 | channel cap=512；FakeProvider；ToolSpec function-only；Provider 优先级；Approval/Sandbox 拆分；Apply patch 不依赖 Lark；Subagent one-shot；Compaction 简化；Resume 4 类测试 |
| §3.4（新增） | 新增 v1.1 | v1 最小 Op / EventMsg 生命周期 |
| §4 目录结构 | 已修订 v1.1 | 加 `models/fake.py`；sandbox/ 拆 mode + cwd_guard + shell_filter |
| §5.3 Compaction | 已修订 v1.1 | 移除"recent n=4"；改为 user_msgs + summary 简化版 |
| §5.4 Resume | 已修订 v1.1 | 加 4 类必测场景说明 |
| §6.2 Apply Patch | 已修订 v1.1 | 不强依赖 Lark；改为手写 lenient parser；4+1 实现顺序 |
| §6.3 / §6.4 | 已修订 v1.1 | ApprovalPolicy 与 SandboxMode 概念拆分 |
| §7 ModelProvider | 已修订 v1.1 | OpenAI-compat first；Anthropic native second；FakeProvider 优先级 0 |
| §9 Benchmark | 已修订 v1.1 | TB subset 硬目标；TB 全量 + SWE-bench 1-3 题改冲刺 |
| §11 14 天计划 | 已修订 v1.1 | 完全替换为 Codex-core first 的 D0-D14 |
| §12 学习笔记 | 已修订 v1.1 | 节奏改为与开发并行；note 03/05 措辞修正 |
| §14 风险 | 已修订 v1.1 | 加 FakeProvider 缺位风险；Apply patch / Anthropic provider 风险细化 |
| Appendix B 配置 | 已修订 v1.1 | default_provider 改为 kimi；anthropic 注释掉 |

**下一步**：用户 review 后用 `superpowers:writing-plans` 把 D2-D14 拆成 step-by-step 实施 plan。
