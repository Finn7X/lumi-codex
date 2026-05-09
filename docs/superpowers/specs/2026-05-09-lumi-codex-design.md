# lumi-codex 设计文档

| 字段 | 值 |
|---|---|
| 作者 | Finn (xujifeng) |
| 日期 | 2026-05-09 |
| 状态 | Draft v1（待 review） |
| 阶段目标 | Phase 1：14 天内完成 Python v1，跑出 benchmark 分数 |
| 后续 | Phase 2（v2，14 天后）：用 Rust 重写 `core`，作为可选延续 |

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
| Op / EventMsg 各几个类型 | **Op 有 ~20 个变体**（UserInput / UserTurn / ExecApproval / Interrupt / Compact / RunUserShellCommand / Shutdown 等），**EventMsg 有 ~35 个变体**（TurnStarted / AgentMessageDelta / ExecApprovalRequest / PlanUpdate / CollabAgentSpawnBegin / TokenCount 等） |
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
| **Submission/Event 总线** | tokio mpsc channels (cap 1600)，`rx_sub.recv()` 驱动主 loop | Python `asyncio.Queue` + `dataclass` Op/Event | codex 的灵魂。1:1 复刻，但 Op/Event 变体精简到 ~8/~12 个 |
| **Streaming** | SSE (`eventsource_stream`) → mpsc channel → consumer | `httpx` SSE + `asyncio.Queue` | 必须从第一天支持 |
| **Tool 统一格式** | 所有 tool → `ResponsesApiTool` | 同样统一到 OpenAI function calling JSON Schema | 把 N×M 复杂度压成 N+M |
| **多 provider** | 单一 `ModelProvider` trait + 配置驱动；built-in 仅 4 个 | 单一 `ModelProvider` ABC + `[provider.X]` TOML 配置；built-in 给 Anthropic/OpenAI/Kimi/GLM | 学 codex 的"配置即扩展"模式 |
| **Approval** | 5 档：`UnlessTrusted` / `OnFailure` / `OnRequest`(默认) / `Granular` / `Never` | 简化到 3 档：`read_only` / `workspace_write` / `full_access` + `--strict` flag | Granular 太细，3 档够 v1 |
| **Sandbox** | macOS Seatbelt / Linux codex-linux-sandbox / Windows windows-sandbox-rs | **只做 ApprovalPolicy + cwd guard + 命令黑名单**，不做 OS 沙箱 | OS 沙箱是工程不是学习；v2 可加 |
| **Apply patch** | Lark grammar 解析 `*** Update File:` + `@@ context @@` | 同样格式，Python `lark` 库 | SWE-bench 必备工具，质量直接决定分数 |
| **Subagent** | `codex_delegate.rs`：parent `Codex::spawn()` 子实例 + `forward_events`/`forward_ops` 桥接，**审批由父拦截** | Python 同构：`SubagentDelegate` 类管理子 thread + 审批拦截 | 这是真实的多 agent 模式 |
| **Plan** | 不是 planner，是 `update_plan` tool；emit `PlanUpdate` event | 同样：`update_plan` tool + `Plan` 数据结构 | 学到 codex"用工具实现规划"的思想 |
| **Compaction** | 跑独立 LLM call 摘要历史；结构 `[initial] + [user_msgs] + [summary]`；摘要存 fake assistant msg | 完全 1:1 复刻，单独的 `compaction.py` | 是 context 管理的精华 |
| **Rollout JSONL** | RolloutItem 5 种：`SessionMeta`/`ResponseItem`/`Compacted{message, replacement_history}`/`TurnContext`/`EventMsg` | 同样 5 种，Python `dataclass` + JSON | session 管理的核心 |
| **Resume** | 反向扫找最近 `Compacted.replacement_history` 作 checkpoint，正向 replay 后缀 | 同样算法 | 这一招学到非常值 |
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

| Leader 列出的能力 | lumi-codex 实现方式 | codex 对应代码 |
|---|---|---|
| **Agent Loop** | `submission_loop` + 通过 tool calls 实现 ReAct | `core/src/session/handlers.rs::submission_loop` |
| **Task 规划与分发** | `update_plan` tool（让模型自己维护 checklist） | `protocol/src/plan_tool.rs` + `tools/src/...` |
| **Context 管理** | `MessageHistory` + 独立 LLM call 摘要 compaction | `core/src/compact.rs` + `core/src/compact_remote.rs` |
| **Session 管理** | JSONL rollout + 反向扫 resume | `core/src/rollout.rs` + `core/src/session/rollout_reconstruction.rs` + `rollout/src/` |
| **Multi-Agent 编排** | `SubagentDelegate` fork-join + 父级审批拦截 | `core/src/codex_delegate.rs` + `agent-graph-store/src/` |

---

## 4. 仓库目录结构

```
lumi-codex/
├── lumi/                                  # 主包
│   ├── core/                              # 引擎（对应 codex-core 最小集）
│   │   ├── thread.py                      # CodexThread - submission_loop 实现
│   │   ├── session.py                     # SessionState
│   │   ├── delegate.py                    # SubagentDelegate（审批拦截 + 事件桥接）
│   │   ├── compaction.py                  # 独立 LLM call 摘要 + replacement history
│   │   ├── context.py                     # MessageHistory 管理
│   │   ├── tool_router.py                 # 统一 ResponsesApiTool 格式 + 派发
│   │   ├── approval.py                    # ApprovalPolicy 3 档 + 黑名单
│   │   └── event_bus.py                   # asyncio Queue 包装
│   ├── protocol/                          # 类型化协议（精简版）
│   │   ├── ops.py                         # ~8 个 Op 变体
│   │   ├── events.py                      # ~12 个 EventMsg 变体
│   │   ├── items.py                       # ResponseItem / RolloutItem
│   │   └── tool_schema.py                 # ResponsesApiTool 等价物
│   ├── tools/
│   │   ├── shell.py                       # exec_command 等价
│   │   ├── file.py                        # read / write
│   │   ├── apply_patch.py                 # codex 的 patch 格式解析（lark）
│   │   ├── search.py                      # ripgrep 包装
│   │   ├── update_plan.py                 # codex 的 plan tool 复刻
│   │   ├── spawn_agent.py                 # 子 agent 调度 tool
│   │   └── registry.py                    # @register_tool 装饰器
│   ├── models/
│   │   ├── base.py                        # ModelProvider ABC + ResponseStream
│   │   ├── anthropic.py                   # native Messages API
│   │   ├── openai.py                      # Responses API / chat completion
│   │   ├── kimi.py                        # OpenAI 兼容
│   │   ├── glm.py                         # zhipuai
│   │   └── stream.py                      # SSE → AsyncIterator 包装
│   ├── rollout/                           # session 持久化
│   │   ├── writer.py                      # 写 JSONL
│   │   ├── reader.py                      # 反向扫 + replacement_history checkpoint
│   │   └── schema.py                      # RolloutItem 5 种
│   ├── sandbox/
│   │   └── policy.py                      # ApprovalPolicy + cwd guard + 命令黑名单
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

复刻 codex 的 inline compaction。

```python
COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000
AUTO_COMPACT_LIMIT = 100_000  # 按模型 context window 计算

async def run_compact(sess: Session, model: ModelProvider) -> CompactedItem:
    """codex compact.rs 的 Python 等价。"""
    # 1. 选取要压缩的历史（除最近 N 轮和 initial system prompt）
    initial_context = sess.history.initial_messages()  # system + 任务描述
    recent_messages = sess.history.recent(n=4)         # 保留最近 4 轮原样
    middle_messages = sess.history.middle()             # 这部分被压缩
    
    # 2. 抽取所有 user messages（drop 工具结果）
    user_msgs = [m for m in middle_messages if m.role == "user"]
    truncated = truncate_to_token_limit(user_msgs, COMPACT_USER_MESSAGE_MAX_TOKENS)
    
    # 3. 调独立 LLM call 摘要
    compact_prompt = load_compact_prompt()  # /core/templates/compact/prompt.md 等价
    summary_stream = model.stream_complete(
        messages=[
            {"role": "system", "content": compact_prompt},
            *initial_context,
            *truncated,
            {"role": "user", "content": "请按照上面的指令对历史做摘要。"},
        ],
        tools=[],  # 摘要不需要 tools
    )
    summary_text = ""
    async for chunk in summary_stream:
        summary_text += chunk.text or ""
    
    # 4. 替换 history 为 [initial_context] + [user_msgs] + [fake assistant: summary]
    new_history = [
        *initial_context,
        *user_msgs,
        AssistantMessage(content=summary_text),
        *recent_messages,
    ]
    
    # 5. 写 RolloutItem.Compacted 到 rollout（含 replacement_history 用于 resume）
    compacted = CompactedItem(message=summary_text, replacement_history=new_history)
    await sess.rollout_writer.write(RolloutItem.Compacted(compacted))
    
    sess.history.replace(new_history)
    return compacted
```

**触发点**：
- `_run_turn` 结尾检查 token usage > `AUTO_COMPACT_LIMIT`
- 用户 `Op.Compact` 命令手动触发

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

#### Reader（`lumi/rollout/reader.py`）—— 复刻 codex 的反向扫算法

```python
def reconstruct_history(rollout_path: Path) -> ReconstructedSession:
    """codex 的 rollout_reconstruction 等价。
    
    算法：反向扫，找最近 Compacted.replacement_history 作 checkpoint，
    然后正向 replay 后缀。
    """
    items = read_jsonl(rollout_path)  # 全部加载
    
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
    
    return ReconstructedSession(history=history, ...)
```

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

### 6.2 Apply Patch（`lumi/tools/apply_patch.py`）—— 复刻 codex Lark grammar

格式：

```
*** Begin Patch
*** Update File: path/to/file.py
@@ def some_function(): @@
-    old_line
+    new_line
*** End Patch
```

实现用 `lark` 库写 grammar，与 codex 的 `tools/src/tool_apply_patch.lark` 一致。Hunk 类型：`AddFile` / `DeleteFile` / `UpdateFile`。`UpdateFileChunk` 包含 `change_context`（@@ 上下文）+ `old_lines` + `new_lines`。

### 6.3 Approval Policy（`lumi/core/approval.py`）

```python
class ApprovalMode(Enum):
    READ_ONLY = "read_only"           # 只读，所有写/exec 拒绝
    WORKSPACE_WRITE = "workspace_write"  # 默认，cwd 内可写，exec 走黑名单审批
    FULL_ACCESS = "full_access"        # 全开，仅 strict mode 下才会要审批

@dataclass
class ApprovalDecision:
    approved: bool
    reason: str = ""

class ApprovalPolicy:
    def __init__(self, mode: ApprovalMode, strict: bool = False):
        self.mode = mode
        self.strict = strict
    
    async def check(self, call: ToolCall, ctx: ToolContext) -> ApprovalDecision:
        if self.mode == ApprovalMode.READ_ONLY:
            if call.is_write_or_exec():
                return ApprovalDecision(False, "read_only mode")
        
        if call.name == "shell":
            cmd = call.args["cmd"]
            if matches_blacklist(cmd):  # rm -rf /, sudo, 出 cwd 的 path 等
                return await ctx.request_user_approval(call, "blacklist match")
            if self.strict:
                return await ctx.request_user_approval(call, "strict mode")
        
        return ApprovalDecision(True)
```

### 6.4 Sandbox（`lumi/sandbox/policy.py`）

仅做：
- **cwd guard**：所有写文件操作必须在 `--cwd` 子树内
- **命令黑名单**：正则匹配（`rm -rf`、`sudo`、绝对 path 出 cwd 等）
- **subprocess 隔离**：用 `asyncio.create_subprocess_exec`，不用 `shell=True`

明确**不做** Seatbelt/Landlock。learning note 05 写清原因 + codex 怎么做的。

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
        tools: list[ResponsesApiTool],
        **kwargs,
    ) -> AsyncIterator[ChatChunk]:
        ...
    
    @abstractmethod
    def model_name(self) -> str: ...
    
    def supports_tools(self) -> bool:
        return True
```

### 7.2 各 provider 实现

| Provider | 接口 | 备注 |
|---|---|---|
| `AnthropicProvider` | native Messages API + tool_use blocks | claude-sonnet-4-6, claude-opus-4-7 |
| `OpenAIProvider` | Responses API（codex 用的） | 备用；用户暂不主用 |
| `KimiProvider` | OpenAI 兼容（base_url=https://api.moonshot.cn/v1） | kimi-k2-0905-preview |
| `GLMProvider` | zhipuai SDK 或 OpenAI 兼容 | glm-4.6 |

**关键点**：`AnthropicProvider` 需要做"native tool_use → 统一 ToolCall"的转换；其他 OpenAI 兼容的直接 map function calling。

### 7.3 配置 schema（`~/.lumi/config.toml`）

```toml
default_provider = "anthropic"

[provider.anthropic]
model = "claude-sonnet-4-6"
api_key_env = "ANTHROPIC_API_KEY"
max_tokens = 8192

[provider.kimi]
model = "kimi-k2-0905-preview"
api_key_env = "MOONSHOT_API_KEY"
base_url = "https://api.moonshot.cn/v1"
wire_api = "openai_compatible"

[provider.glm]
model = "glm-4.6"
api_key_env = "GLM_API_KEY"

[approval]
mode = "workspace_write"
strict = false

[sandbox]
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

### 9.2 HumanEval 接入（冒烟测试）

- 数据：`openai/human-eval` HuggingFace dataset，164 题
- Workspace：每题一个临时目录，含 `prompt.py` + `test.py`
- Grade：跑 `pytest test.py` exit code

**目标**：仅验证 lumi 跑得通；不期望高分（没意义）。

### 9.3 Terminal-Bench 接入（主战场）

- 数据：`tbench` 的 ~89 个 core tasks
- Workspace：每题一个 Docker 镜像或本地目录
- Grade：tbench 官方 grader

**目标**：30-50% 通过率。失败 case 写分析报告（哪个组件不行）。

### 9.4 SWE-bench Lite 接入（难度展示）

- 数据：300 题中选 5-10 题（手挑相对简单的 Django/Flask 类）
- Workspace：Docker 镜像（仓库 + 测试环境）
- Grade：跑指定测试，看是否 pass

**目标**：能跑通即可；分数高低不重要，关键是 demo 模型可以处理多文件 patch。

### 9.5 模型对照跑分

最后一天用同一套 Terminal-Bench subset 跑 Claude / Kimi / GLM，对照表展示：

| 模型 | Terminal-Bench 通过率 | 平均 token 消耗 | 总成本 |
|---|---|---|---|
| claude-sonnet-4-6 | ?% | ? | ¥? |
| kimi-k2 | ?% | ? | ¥? |
| glm-4.6 | ?% | ? | ¥? |

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

## 11. 14 天时间表

**起始**：2026-05-09（今天）
**结束**：2026-05-22（预留 5/23 缓冲）

### Phase 1：Codex 学习（Day 1-3，5/09-5/11）

| Day | 任务 | 产出 |
|---|---|---|
| **D1** (5/09) | clone codex-rs；通读 README + workspace `Cargo.toml` + `docs/`；定位 100+ crate 的核心 vs 外围 | `01-codex-workspace-overview.md` + 设计文档 finalize |
| **D2** (5/10) | 精读 `core/src/codex_thread.rs` + `protocol/src/protocol.rs` + `core/src/session/handlers.rs::submission_loop`；对照 Op/Event 全部变体 | `02-submission-event-bus.md` |
| **D3** (5/11) | 精读 `compact.rs` + `rollout/` + `apply-patch/` + `codex_delegate.rs`；笔记 03/04/05/06 各写到能解释清楚的程度（笔记 07 留到 v1 完工后） | `03-context-and-compaction.md`、`04-rollout-and-resume.md`、`05-tool-and-sandbox.md`、`06-multi-agent-delegate.md` |

### Phase 2：构建（Day 4-14，5/12-5/22）

| Day | 任务 | 验收标准 |
|---|---|---|
| **D4** (5/12) | skeleton：`uv init`，目录结构，`Op`/`EventMsg` dataclass，`EventBus`，`ModelProvider` ABC，`AnthropicProvider`，最小 `lumi exec "hello"` 能调通 Claude | `lumi exec "say hi"` 输出 Claude 响应 |
| **D5** (5/13) | `CodexThread.submission_loop` + `_run_turn` tool-call loop；`ToolRouter`；`update_plan` tool；`shell`/`read_file`/`write_file` tools | `lumi exec "list files in cwd and create a hello.txt"` 能跑通 |
| **D6** (5/14) | `apply_patch` tool（lark grammar，写测试覆盖 add/delete/update）；`ApprovalPolicy` 3 档；`cwd guard` + 命令黑名单 | `apply_patch` 能成功改一个真实 Python 文件 + 测试通过 |
| **D7** (5/15) | `MessageHistory` + `compaction.py`（独立 LLM 摘要 + replacement_history）；token 计数（用模型 usage 而非 tiktoken） | 模拟 200k token 历史能 compact 成 < 50k 且后续 turn 正常 |
| **D8** (5/16) | `RolloutWriter`（JSONL）+ `RolloutReader`（反向扫 resume）；`KimiProvider` + `GLMProvider`；`lumi resume <id>` | 中断一个 session，`resume` 后能继续；3 个 provider 都能跑 |
| **D9** (5/17) | `SubagentDelegate`（fork-join + 审批拦截）；`spawn_agent` tool；`demos/multi_agent_writer/` 跑通 | Demo 能产出一篇博客 |
| **D10** (5/18) | HumanEval runner + smoke test；定位早期 bug | HumanEval pass rate ≥ 70%（基本是模型分数，验证 lumi 没炸） |
| **D11** (5/19) | Terminal-Bench 接入：tbench 框架对接 + 跑 10 题 subset | 10 题里至少 3 题通过 |
| **D12** (5/20) | Terminal-Bench 全跑 + 分析失败 case + 调 prompt/工具 | 全 89 题跑完，pass rate 报告 |
| **D13** (5/21) | SWE-bench Lite Docker 接入 + 跑 5-10 题；模型对照跑分（Terminal-Bench subset × 3 provider） | SWE-bench 至少 1 题通过；模型对照表完成 |
| **D14** (5/22) | README + asciinema demo + 笔记 07（codex vs claude code）+ 整理 + push final | 所有内容上 GitHub |

**Buffer**：5/23 留作 buffer day。

---

## 12. 学习笔记 7 篇大纲

每篇目标：自己能把这一块讲清楚。

| # | 标题 | 核心内容 |
|---|---|---|
| 01 | **codex-rs Workspace 总览** | 100+ crate 分类（核心/扩展/cloud/外围）；阅读路径推荐；为什么 Rust 而不是 TS |
| 02 | **Submission/Event 总线设计** | submission_loop 主循环；Op/EventMsg 全部变体分类；为什么 channel-driven 而不是 request-response；async task 派发模式 |
| 03 | **Context 管理与 Compaction** | inline vs remote compaction；replacement_history 结构；`COMPACT_USER_MESSAGE_MAX_TOKENS`；为什么用 fake assistant message 存 summary |
| 04 | **Rollout 与 Resume** | RolloutItem 5 种；反向扫找 checkpoint 的精妙；与 SQLite state 的分工；ThreadRolledBack 处理 |
| 05 | **Tool 系统与 Sandbox** | `ResponsesApiTool` 统一格式；AskForApproval 5 档；Seatbelt vs Landlock vs Windows sandbox 实现差异；apply_patch lark grammar |
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

## 14. 风险与应对

| 风险 | 概率 | 应对 |
|---|---|---|
| Apply patch lark grammar 复杂，Day 6 完不成 | 中 | 备选：v1 用 search-replace 简化版，v1.5 升级到 lark grammar |
| Terminal-Bench 通过率低（< 20%） | 中 | 写清楚分析报告，分类失败原因（tool 不够用 / context 管理差 / planner 弱）。失败分析本身就是产出 |
| 模型成本超预算 | 低 | 全跑分用 Kimi/GLM，Claude 仅用于关键调试；strict mode 拦危险命令 |
| Compaction 策略不熟，写错 | 低 | 直接对照 codex `compact.rs` 1:1 复刻，不自创 |
| SWE-bench Docker 配置卡住 | 中 | 跳过 SWE-bench，只跑 Terminal-Bench；deliverable 中标注 |
| Multi-agent demo 不稳定 | 中 | demo 失败可接受，关键是 `SubagentDelegate` 代码本身正确，写测试覆盖 |
| 时间不够 | 中 | 优先级：Agent Loop > Context/Session > Apply Patch > Multi-Agent > Benchmark；最后两天必须留给 benchmark 跑分 + 收尾 |

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
default_provider = "anthropic"

[provider.anthropic]
model = "claude-sonnet-4-6"
api_key_env = "ANTHROPIC_API_KEY"
max_tokens = 8192

[provider.openai]
model = "gpt-5"
api_key_env = "OPENAI_API_KEY"
wire_api = "responses"

[provider.kimi]
model = "kimi-k2-0905-preview"
api_key_env = "MOONSHOT_API_KEY"
base_url = "https://api.moonshot.cn/v1"
wire_api = "openai_compatible"

[provider.glm]
model = "glm-4.6"
api_key_env = "GLM_API_KEY"
wire_api = "openai_compatible"
base_url = "https://open.bigmodel.cn/api/paas/v4"

[approval]
mode = "workspace_write"   # read_only / workspace_write / full_access
strict = false              # benchmark 时可置 true

[sandbox]
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
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
MOONSHOT_API_KEY=sk-...
GLM_API_KEY=...
```

---

## 附：本设计文档的状态

- [x] §0 背景
- [x] §1 关键决策（已与用户确认）
- [x] §2 Codex 源码深读发现
- [x] §3 系统架构（用户已确认 §1.4 目录结构 + 7 篇笔记）
- [x] §4 仓库目录结构
- [ ] §5 5 大组件详细设计 — **待用户 review**
- [ ] §6 Tool 系统与沙箱 — **待用户 review**
- [ ] §7 ModelProvider — **待用户 review**
- [ ] §8 CLI 设计 — **待用户 review**
- [ ] §9 Benchmark 集成方案 — **待用户 review**
- [ ] §10 Multi-Agent Demo — **待用户 review**
- [ ] §11 14 天时间表 — **待用户 review**
- [ ] §12 学习笔记 7 篇大纲 — **待用户 review**
- [ ] §13 v2 Rust 重写计划 — **待用户 review**
- [ ] §14 风险与应对 — **待用户 review**

下一步：用户 review 后用 `superpowers:writing-plans` 写 Phase 2 实施 plan（Day 4-14 day-by-day breakdown）。
