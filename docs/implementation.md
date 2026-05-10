# lumi-codex 模块实现细节

每个模块的实现说明（伪代码 + 关键设计点）。文件路径见 [`architecture.md`](./architecture.md#仓库目录结构)。

---

## 1. Agent Loop（`lumi/core/thread.py`）

参照 codex 的 `submission_loop` + `run_turn` 两层结构：

- **submission_loop**：阻塞收 `Submission`，按 `Op` 派发到 handler；handler 内启动后台 task
- **run_turn**：单个 turn 内的 tool-call continuation 循环（模型调 tool → 执行 → 回写 → 再调模型）

### 1.1 submission_loop

```python
class CodexThread:
    def __init__(self, sess: Session, model: ModelProvider, router: ToolRouter):
        self.sess = sess
        self.model = model
        self.router = router
        self.sub_queue: asyncio.Queue[Submission] = asyncio.Queue()
        self.event_queue: asyncio.Queue[EventMsg] = asyncio.Queue()

    async def submission_loop(self):
        """codex 的 submission_loop 等价物。channel-driven。"""
        while True:
            sub = await self.sub_queue.get()
            match sub.op:
                case Op.UserInput(items) | Op.UserTurn(items, ...):
                    asyncio.create_task(self._run_turn(sub.id, items))
                case Op.ExecApproval(call_id, decision):
                    await self._handle_approval(call_id, decision)
                case Op.PatchApproval(call_id, decision):
                    await self._handle_approval(call_id, decision)
                case Op.Interrupt():
                    await self._abort_current_turn()
                case Op.Compact():
                    asyncio.create_task(self._run_compact())
                case Op.ThreadRollback(turns):
                    await self._rollback(turns)
                case Op.Shutdown():
                    break
```

### 1.2 _run_turn

```python
async def _run_turn(self, turn_id: str, items: list[UserInput]):
    """单个 turn：调模型 → 流式输出 → 处理 tool calls → 必要时 loop。"""
    await self._emit(EventMsg.TurnStarted(turn_id))
    try:
        self.sess.history.add_user(items)

        while True:  # tool-call continuation loop
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
                if chunk.usage:
                    await self._emit(EventMsg.TokenCount(chunk.usage))

            if not tool_calls:
                break  # 模型给最终答案，无 tool call，turn 结束

            for call in tool_calls:
                decision = await self._approval_check(call)
                if decision.is_reject:
                    result = ToolResult.rejected(decision.reason)
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

### 1.3 关键设计点

- **主 loop（`submission_loop`）只管派发**，不直接做 ReAct
- **每个 turn 在独立 task 里跑**，可被 `Op.Interrupt` 中断
- **Tool-call 内 loop 是显式的**：模型每次返回若有 tool calls 就执行后再调一次模型
- **流式 delta 直接 emit 事件**，不缓存
- **审批检查在工具执行之前**，被拒绝时 result 写入 history，模型下一轮会看到

---

## 2. Task 规划：`update_plan` tool（`lumi/tools/update_plan.py`）

复刻 codex 的 `plan_tool`。**让模型自维护 checklist**，不是独立 LLM planner。

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

**System prompt 引导**："当用户任务包含 3 个以上独立子任务时，先调 `update_plan` 列出步骤"。

---

## 3. Context 管理与 Compaction（`lumi/core/compaction.py`）

### 3.1 MessageHistory（`lumi/core/context.py`）

维护当前会话的消息列表，提供：
- `add_user(items)` / `add_assistant(content)` / `add_tool_result(call, result)`
- `materialize() -> list[Message]` — 给模型的最终输入
- `token_count() -> int` — 用 `approx_token_count`（chars/4 启发式）累加
- `replace(new_history)` — compaction 后替换

### 3.2 Compaction 策略

```python
COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000
AUTO_COMPACT_LIMIT = 100_000  # 按模型 context window 调整

async def run_compact(sess: Session, model: ModelProvider) -> CompactedItem:
    """独立 LLM call 摘要历史，结构 [selected_user_messages] + [assistant_summary]。"""
    # 1. 从新到旧选取 user messages，总预算 ≤ 20k approx tokens
    user_msgs = sess.history.collect_user_messages_newest_first()
    selected = []
    budget = 0
    for msg in user_msgs:
        msg_tokens = approx_token_count(msg)
        if budget + msg_tokens > COMPACT_USER_MESSAGE_MAX_TOKENS:
            if not selected:
                selected.append(truncate_msg(msg, COMPACT_USER_MESSAGE_MAX_TOKENS))
            break
        selected.append(msg)
        budget += msg_tokens
    selected.reverse()  # 恢复时间顺序

    # 2. 调独立 LLM call 摘要（不带 tools）
    compact_prompt = load_compact_prompt()
    summary_stream = model.stream_complete(
        messages=[
            {"role": "system", "content": compact_prompt},
            *sess.history.materialize(),
            {"role": "user", "content": "请按上面的指令对历史做摘要。"},
        ],
        tools=[],
    )
    summary_text = ""
    async for chunk in summary_stream:
        summary_text += chunk.text or ""

    # 3. 构造 replacement_history
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

### 3.3 触发点

- `_run_turn` 结尾检查 token usage > `AUTO_COMPACT_LIMIT`
- 用户 `Op.Compact` 命令手动触发

### 3.4 测试要求

用 `FakeProvider` 模拟 summarizer 返回固定文本，断言：
- `replacement_history` 结构正确（selected user msgs + assistant summary）
- `CompactedItem` 写入 rollout
- `history.materialize()` 反映 replacement
- token 数显著降低

---

## 4. Session：Rollout & Resume

### 4.1 Schema（`lumi/rollout/schema.py`）—— 5 种 RolloutItem

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
    sandbox_mode: str
    model: str

@dataclass
class CompactedItem:
    message: str
    replacement_history: list[ResponseItem]  # 完整 history checkpoint

# 5 种 RolloutItem
RolloutItem = SessionMeta | ResponseItem | CompactedItem | TurnContextItem | EventMsg
```

### 4.2 Writer（`lumi/rollout/writer.py`）

每个 item 一行 JSON，写到 `~/.lumi/sessions/rollout-<ts>-<uuid>.jsonl`。每行格式：

```json
{"type": "response_item", "payload": {...}}
```

关键是保证 `ResponseItem` 能无损重建模型输入。

### 4.3 Reader（`lumi/rollout/reader.py`）—— 反向扫算法

```python
def reconstruct_history(rollout_path: Path) -> ReconstructedSession:
    items = read_jsonl(rollout_path)

    # 反向扫：找最近 Compacted.replacement_history 作 checkpoint
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

### 4.4 4 类必测场景（`tests/integration/test_resume.py`）

| # | 场景 | 期望行为 |
|---|---|---|
| 1 | 无 compaction | replay 所有 `ResponseItem`；history 与原 turn 一致 |
| 2 | 有 compaction | 使用最近的 `replacement_history` + suffix；token 显著小于场景 1 |
| 3 | 有 rollback（无 compaction） | 丢弃最近 N 个 user turn 及其 assistant 回应 |
| 4 | compaction + rollback 组合 | rollback 不能错误"复活"被 rollback 的历史；checkpoint 必须属于存活 segment |

### 4.5 CLI

- `lumi resume <session_id>` —— 续会
- `lumi replay <session_id>` —— 离线重放（debug）
- `lumi sessions list` —— 列出本地 sessions

---

## 5. Multi-Agent：SubagentDelegate（`lumi/core/delegate.py`）

复刻 codex 的 fork-join + 父级审批拦截。

### 5.1 spawn_interactive

```python
class SubagentDelegate:
    """父 agent 派生子 agent，桥接事件流，拦截子 agent 的审批请求。"""

    @staticmethod
    async def spawn_interactive(
        parent_sess: Session,
        config: SubagentConfig,
    ) -> tuple[asyncio.Queue, asyncio.Queue]:
        # 1. 创建子 Codex 实例（独立 session、独立 history、受限 tool 集）
        sub_sess = Session.create_subagent(
            parent=parent_sess,
            tools=config.allowed_tools,
            system_prompt=config.system_prompt,
        )
        sub_thread = CodexThread(sub_sess, parent_sess.model, parent_sess.router)
        asyncio.create_task(sub_thread.submission_loop())

        # 2. forward_events: 子 agent 的事件流，过滤审批请求
        events_out: asyncio.Queue = asyncio.Queue()
        async def forward_events():
            while True:
                event = await sub_thread.event_queue.get()
                if isinstance(event, EventMsg.ExecApprovalRequest | EventMsg.ApplyPatchApprovalRequest):
                    # 父级审批拦截！由 parent 的 policy 处理，不冒泡到上层 caller
                    decision = await parent_sess.handle_approval(event)
                    op = Op.ExecApproval(event.id, decision) \
                        if isinstance(event, EventMsg.ExecApprovalRequest) \
                        else Op.PatchApproval(event.id, decision)
                    await sub_thread.sub_queue.put(Submission(id=new_id(), op=op))
                else:
                    await events_out.put(event)
        asyncio.create_task(forward_events())

        # 3. forward_ops: caller 提交的 op 转发给子 agent
        ops_in: asyncio.Queue = asyncio.Queue()
        async def forward_ops():
            while True:
                op = await ops_in.get()
                await sub_thread.sub_queue.put(Submission(id=new_id(), op=op))
        asyncio.create_task(forward_ops())

        return events_out, ops_in
```

### 5.2 spawn_one_shot（v1 主用）

```python
@staticmethod
async def spawn_one_shot(
    parent_sess: Session,
    config: SubagentConfig,
    user_input: str,
) -> str:
    """提交单条输入，等 TurnComplete，关闭。"""
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

### 5.3 配套 tool（`lumi/tools/spawn_agent.py`）

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

## 6. Tool 系统

### 6.1 ToolSpec 统一格式（`lumi/protocol/tool_schema.py`）

```python
@dataclass
class ToolSpec:
    name: str
    description: str
    input_schema: dict           # JSON Schema
    output_schema: dict | None = None
```

由 provider adapter 转 OpenAI function calling / Anthropic tool_use 格式。

### 6.2 Registry + ToolRouter（`lumi/tools/registry.py` + `lumi/core/tool_router.py`）

```python
_REGISTRY: dict[str, ToolHandler] = {}

def register_tool(name: str, description: str, schema: dict):
    def decorator(fn):
        _REGISTRY[name] = ToolHandler(name, description, schema, fn)
        return fn
    return decorator

class ToolRouter:
    def exposed_schemas(self) -> list[ToolSpec]:
        return [t.to_spec() for t in _REGISTRY.values()]

    async def execute(self, call: ToolCall, sess: Session) -> ToolResult:
        handler = _REGISTRY[call.name]
        ctx = ToolContext(session=sess, ...)
        return await handler.fn(call.args, ctx)
```

### 6.3 内置 tool 列表

| Tool | 说明 |
|---|---|
| `shell` | 在 cwd 内执行 shell 命令 |
| `read_file` | 读文件 |
| `write_file` | 写文件（仅 cwd 内） |
| `apply_patch` | 应用 codex 格式的 patch |
| `search` | ripgrep / grep 包装 |
| `update_plan` | 维护任务 checklist |
| `spawn_agent` | 派生 one-shot 子 agent |

### 6.4 apply_patch（`lumi/tools/apply_patch.py`）

**手写 lenient parser**，兼容 codex patch 格式。

格式：

```
*** Begin Patch
*** Update File: path/to/file.py
@@ def some_function(): @@
-    old_line
+    new_line
*** End Patch
```

**实现顺序**：

1. **Add File**：`*** Add File: <path>` + `+` 前缀的全部内容行
2. **Delete File**：`*** Delete File: <path>`
3. **Update File**：`@@ context @@` + `-/+` 行；context 用于在文件中定位
4. **Move File**：`*** Update File: ...` 之后跟 `*** Move to: <new_path>`
5. **Lenient parsing**：去掉 ``` fences、heredoc wrapper、前后空白、混合 LF/CRLF（兼容模型常见错误输出）

**测试要求**：parser（解析正确性）和 apply（实际写文件后内容正确）**分开测试**。每类 hunk 至少 3 个 case：正常、含特殊字符、含 lenient 前缀。

---

## 7. ApprovalPolicy 与 SandboxMode

两个独立概念，**不要混淆**。

### 7.1 ApprovalPolicy（`lumi/core/approval.py`）—— 何时问用户

```python
class ApprovalPolicy(Enum):
    ON_REQUEST = "on_request"        # 默认：模型决定何时请求审批
    UNLESS_TRUSTED = "unless_trusted" # 只有"已知安全"的命令免审批
    NEVER = "never"                   # 不问用户，一律按 SandboxMode 决策

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
        return ApprovalDecision(True)
```

### 7.2 SandboxMode（`lumi/sandbox/mode.py`）—— 工具能访问哪里

```python
class SandboxMode(Enum):
    READ_ONLY = "read_only"                   # 不允许写、不允许 exec
    WORKSPACE_WRITE = "workspace_write"       # 默认：cwd 内可写，shell 命令过黑名单
    DANGER_FULL_ACCESS = "danger_full_access" # 全开（仅在 Docker/临时目录用）
```

实现层（`lumi/sandbox/`）：

- **cwd guard** (`cwd_guard.py`)：所有写文件操作必须落在 `--cwd` 子树内
- **shell 黑名单** (`shell_filter.py`)：正则匹配（`rm -rf`、`sudo`、绝对 path 出 cwd 等）触发审批
- **subprocess 隔离**：`asyncio.create_subprocess_exec`，不用 `shell=True`

### 7.3 重要警告

> ⚠️ cwd guard + 黑名单**不是安全沙箱**，只是防误操作的轻量机制。**跑 benchmark 或不可信任务时，必须在临时目录或 Docker workspace 里运行**。

---

## 8. ModelProvider

### 8.1 ABC（`lumi/models/base.py`）

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

### 8.2 Provider 实现

| Provider | 接口 | 备注 |
|---|---|---|
| `FakeProvider` | 返回预设 chunks 序列 | 测试基础。所有 loop/router/rollout/approval/compaction 测试都用它 |
| `OpenAICompatProvider` | OpenAI Chat Completions / Responses API | 一份代码覆盖 OpenAI / Kimi / GLM（差异在 `base_url` + `model`） |
| `AnthropicProvider` | native Messages API + tool_use blocks | 通过 adapter 把统一 `ToolSpec` 转 Anthropic native shape |

**关键点**：tool 调用差异留在 **provider adapter 层**，不要让每个 tool 写多个 provider 版本。

### 8.3 配置 schema

见 [`architecture.md` 配置示例](./architecture.md#-lumiconfigtoml)。

CLI 切换：`lumi --provider kimi exec "改 auth.py"`。

---

## 9. CLI（`lumi/cli/`）

### 9.1 子命令

| 命令 | 用途 |
|---|---|
| `lumi` | 启动交互模式 |
| `lumi exec <prompt>` | headless 单次执行（benchmark 用） |
| `lumi resume <session_id>` | 续会 |
| `lumi replay <session_id>` | 离线重放 rollout（debug） |
| `lumi sessions list` | 列出本地 sessions |
| `lumi config show / set` | 配置管理 |

### 9.2 交互模式（`lumi/cli/interactive.py`）

- `prompt_toolkit` 读用户输入（多行、history、autocomplete）
- `rich` 渲染 EventMsg 流（彩色输出、code block 高亮、stream cursor）
- 审批用 inline prompt：`>> 是否允许执行 'rm xxx'？(y/n/always)`

### 9.3 Headless exec（`lumi/cli/exec.py`）

- 单次输入（args 或 stdin），完成后退出
- 默认输出：最终答案到 stdout，事件流到 stderr
- `--json` 模式：所有事件序列化为 JSONL 输出 stdout（benchmark runner 解析用）
