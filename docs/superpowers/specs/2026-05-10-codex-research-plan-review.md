# Codex 真实实现调研与 lumi-codex 计划评审

| 字段 | 值 |
|---|---|
| 日期 | 2026-05-10 |
| 调研对象 | OpenAI Codex 本地源码快照：`/Users/xujifeng/dev/_research/codex` |
| 快照版本 | `c37f743`，2026-04-30，`Gate multi-agent v2 tools independently of collab (#20246)` |
| 评审对象 | `docs/superpowers/specs/2026-05-09-lumi-codex-design.md` |
| 结论 | 方向正确，但 14 天交付范围需要收敛；若照原计划全做，benchmark 和稳定性大概率会被挤掉 |

---

## 1. 总体判断

当前计划抓住了 Codex 的核心精神：**channel-driven agent loop、tool-call continuation、plan 作为工具、rollout JSONL、compaction、subagent fork-join**。这些正好对应 teamleader 提到的 Agent Loop、Task 规划与分发、Context 管理、Session 管理、Multi-Agent 编排。

但原计划有一个明显风险：它把“学习 Codex 核心架构”和“做一个能跑多 benchmark 的终端 agent”压进 11 天构建期，同时还包含 4 个 provider、Terminal-Bench 全量、SWE-bench Lite、HumanEval、multi-agent demo、apply_patch grammar、resume、compaction。对个人学习项目来说，这个目标很漂亮，但工程上偏满。

建议改成 **Codex-core first**：

1. 先跑通 `lumi exec` 的真实 agent loop 和工具闭环。
2. 再做 rollout/resume 与 apply_patch，这是 benchmark 能跑的基础。
3. 最后做 compaction 和 one-shot subagent，作为架构深度展示。
4. Benchmark 只把 Terminal-Bench subset 设为硬目标，Terminal-Bench 全量和 SWE-bench 作为冲刺项。

---

## 2. Codex 真实实现要点

### 2.1 Agent Loop 是 submission 驱动，不是同步 ReAct

Codex 的主循环在 `codex-rs/core/src/session/handlers.rs::submission_loop`。它从 `async_channel::Receiver<Submission>` 收消息，然后按 `Op` 分发。真正的 turn 执行在 `run_turn` 里，模型每次 sampling 后如果需要工具 follow-up，就继续下一轮 sampling。

需要复刻的是这两个层次：

| 层次 | Codex 做法 | lumi-codex v1 应复刻 |
|---|---|---|
| Submission loop | 阻塞收 `Submission`，按 `Op` 派发 | `asyncio.Queue[Submission]` + handler 分发 |
| Turn loop | 模型输出 tool call 后执行工具，再继续 sampling | `_run_turn` 内部循环，直到无 tool call |
| 事件输出 | 所有 UI/CLI 都消费 `EventMsg` | CLI 只渲染事件，不直接读内部状态 |

源码依据：
- `../_research/codex/codex-rs/core/src/session/handlers.rs:968`
- `../_research/codex/codex-rs/core/src/session/turn.rs:136`

### 2.2 当前 Codex 的协议比计划里更大

本地快照里 `Op` 已经超过 30 个变体，`EventMsg` 超过 70 个变体。原设计文档中“Op ~20、EventMsg ~35”的数量不是当前快照事实。

这不影响 v1 精简，但要调整表述：**不是照数量复刻，而是照生命周期复刻**。

v1 最小 `Op`：

| Op | 用途 |
|---|---|
| `UserInput` / `UserTurn` | 提交用户任务 |
| `ExecApproval` | 回复 shell 审批 |
| `PatchApproval` | 回复 patch 审批 |
| `Interrupt` | 中断当前 turn |
| `Compact` | 手动压缩上下文 |
| `ThreadRollback` | 丢弃最近 N 个用户 turn，可选 |
| `Shutdown` | 关闭 session |

v1 最小 `EventMsg`：

| Event | 用途 |
|---|---|
| `TurnStarted` / `TurnComplete` / `TurnAborted` | turn 生命周期 |
| `AgentMessageDelta` / `AgentMessage` | assistant 输出 |
| `UserMessage` | 写入模型输入 |
| `ExecCommandBegin` / `ExecCommandOutputDelta` / `ExecCommandEnd` | shell 工具 |
| `ExecApprovalRequest` / `ApplyPatchApprovalRequest` | 审批 |
| `PlanUpdate` | 规划工具 |
| `TokenCount` | token usage |
| `ContextCompacted` / `ThreadRolledBack` | session/context 变化 |
| `Error` / `Warning` | 异常与提示 |

源码依据：
- `../_research/codex/codex-rs/protocol/src/protocol.rs:405`
- `../_research/codex/codex-rs/protocol/src/protocol.rs:1314`

### 2.3 Plan 不是 Planner，是 `update_plan` 工具

这一点原计划正确。Codex 没有独立 planner 模块，`update_plan` 是一个模型可调用工具，参数是 `plan` 和可选 `explanation`。handler 的核心行为是解析 JSON 参数并 emit `EventMsg::PlanUpdate`。

v1 不应该做单独 planner agent。要把计划能力放在 system prompt 和工具 schema 中，让模型自己维护 checklist。

源码依据：
- `../_research/codex/codex-rs/tools/src/plan_tool.rs:6`
- `../_research/codex/codex-rs/core/src/tools/handlers/plan.rs:80`

### 2.4 Tool 统一是 Responses API 形状，但不是只有 function tool

原计划“所有 provider 共用 `ResponsesApiTool` schema”方向对，但需要补一层现实情况：当前 Codex 的 `ToolSpec` 不只有 function，还包括：

- `Function(ResponsesApiTool)`
- `Namespace`
- `ToolSearch`
- `LocalShell`
- `ImageGeneration`
- `WebSearch`
- `Freeform`

v1 可以只实现 function-style tools，但文档里应明确这是裁剪，而不是 Codex 本身只有这一种。

建议 v1 统一内部工具为：

```text
ToolSpec(name, description, input_schema, output_schema?)
```

再由 provider adapter 转成 OpenAI compatible / Anthropic native tool 格式。这里要避免的是“每个工具都写一套 provider parser”，不是避免 provider adapter。

源码依据：
- `../_research/codex/codex-rs/tools/src/responses_api.rs:26`
- `../_research/codex/codex-rs/tools/src/tool_spec.rs:18`
- `../_research/codex/codex-rs/core/src/tools/router.rs:39`

### 2.5 ModelProvider 不是“每家 API 一个完整客户端”

当前 Codex 的 `ModelProvider` trait 更像 provider metadata/auth/capability 抽象，实际 stream 调用走 `ModelClientSession`、Responses API、SSE/WebSocket 等层。内置 provider 也不是 Claude/Kimi/GLM，而是 OpenAI、Amazon Bedrock、Ollama、LM Studio；第三方 provider 鼓励用户通过 config 添加。

v1 没必要照这个拆这么细，但计划要改成：

1. `ModelProvider` 负责统一 `stream_complete()` 抽象。
2. 第一优先级只做 **OpenAI-compatible provider**，覆盖 OpenAI/Kimi/GLM。
3. Anthropic native 放到第二阶段，因为它的 tool_use/tool_result shape 会增加不少 adapter 成本。

源码依据：
- `../_research/codex/codex-rs/model-provider/src/provider.rs:79`
- `../_research/codex/codex-rs/model-provider-info/src/lib.rs:402`

### 2.6 Token usage 依赖模型返回，但 Codex 也有近似 token 估算

“不要用 tiktoken”是对的。Codex 的真实 token usage 主要来自模型返回的 `TokenUsage`，并持久化到 `TokenCount` 事件。  

但源码中也使用 `approx_token_count` 做文本截断和预算估算，比如 compaction 选择保留哪些 user messages、工具输出截断、base instructions 估算。也就是说，v1 应该是：

- 真实 usage：用 provider 返回值。
- 截断/预算：用非常粗的 `approx_token_count`，例如 chars/4 或 words heuristic。
- 不引入 tiktoken。

源码依据：
- `../_research/codex/codex-rs/protocol/src/protocol.rs:2057`
- `../_research/codex/codex-rs/core/src/context_manager/history.rs:262`
- `../_research/codex/codex-rs/core/src/compact.rs:479`

### 2.7 Compaction 的原计划需要修正

原计划写了“保留最近 4 轮原样”，这不是当前 Codex 的核心逻辑。

Codex inline compaction 的关键是：

1. 把当前 history 加上压缩 prompt，作为一次独立模型调用。
2. 从压缩 turn 的最后 assistant message 提取 summary。
3. 从旧 history 收集 user messages。
4. 用 `build_compacted_history(initial_context, user_messages, summary_text)` 重建历史。
5. user messages 从新到旧选取，总预算 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`，必要时截断最老的一个。
6. summary 作为 fake assistant message 进入 replacement history。
7. mid-turn compaction 会把 initial context 插到最后一个真实 user message 之前；manual/pre-turn compaction 通常不注入，后续 turn 再注入。

v1 建议先实现一种稳定版本：

```text
new_history = [selected_user_messages <= 20k approx tokens] + [assistant summary]
```

等 resume 跑通后，再加 initial context injection。

源码依据：
- `../_research/codex/codex-rs/core/src/compact.rs:44`
- `../_research/codex/codex-rs/core/src/compact.rs:116`
- `../_research/codex/codex-rs/core/src/compact.rs:382`
- `../_research/codex/codex-rs/core/src/compact.rs:462`

### 2.8 Rollout resume 比“找 checkpoint + replay suffix”更复杂

原计划方向对，但算法写得过于简化。当前 Codex 反向扫时会按 turn segment 处理：

- `ThreadRolledBack` 表示“丢弃最新 N 个用户 turn”。
- 反向扫会跳过被 rollback 的用户 turn segment。
- `Compacted.replacement_history` 是 checkpoint，但要确认它属于“存活 segment”。
- 同时恢复 `previous_turn_settings` 和 `reference_context_item`，用于 resume/fork 后重建 turn context。
- 正向 replay suffix 时，遇到 legacy compaction 没有 `replacement_history` 还会 fallback rebuild。

v1 可以先实现简化版，但测试必须覆盖：

1. 无 compaction：replay 所有 `ResponseItem`。
2. 有 compaction：使用最近的 `replacement_history` + suffix。
3. 有 rollback：丢弃最近 N 个 user turn。
4. compaction 后 rollback：不能错误复活被 rollback 的历史。

源码依据：
- `../_research/codex/codex-rs/protocol/src/protocol.rs:2809`
- `../_research/codex/codex-rs/core/src/session/rollout_reconstruction.rs:87`
- `../_research/codex/codex-rs/core/src/session/rollout_reconstruction.rs:220`

### 2.9 Apply Patch 不要被 Lark 绑死

当前 Codex repo 里确实有 `tools/src/tool_apply_patch.lark` 作为 grammar spec，但实际 `apply-patch/src/parser.rs` 是手写 lenient parser，并且专门兼容模型把 heredoc 当参数传入等情况。

v1 可以用 Python `lark`，但目标应是“兼容 Codex patch 格式”，不是“必须用 Lark 证明正确”。更务实的实现顺序：

1. 先支持 `Add File` / `Delete File` / `Update File`。
2. 再支持 `*** Move to:`。
3. 再支持 lenient parsing：去掉 ``` fences、heredoc wrapper、前后空白。
4. 给 parser 和 apply behavior 分开写测试。

源码依据：
- `../_research/codex/codex-rs/apply-patch/src/parser.rs:1`
- `../_research/codex/codex-rs/tools/src/tool_apply_patch.lark`

### 2.10 Approval 和 Sandbox 要分开建模

原计划把 `read_only / workspace_write / full_access` 放进 `ApprovalMode`，这会混淆 Codex 的两个概念：

- `AskForApproval`：什么时候问用户。
- `SandboxPolicy` / `PermissionProfile`：工具实际能访问哪里。

v1 建议分开：

```text
ApprovalPolicy = on_request | never | unless_trusted
SandboxMode = read_only | workspace_write | danger_full_access
```

不做 OS sandbox 是合理裁剪，但文档要说清楚：cwd guard + blacklist 不是安全沙箱，只是学习项目里的防误操作机制。跑 benchmark 时应在临时目录或 Docker workspace 里运行。

源码依据：
- `../_research/codex/codex-rs/protocol/src/protocol.rs:941`
- `../_research/codex/codex-rs/protocol/src/protocol.rs:1035`
- `../_research/codex/codex-rs/sandboxing/src/seatbelt.rs:26`
- `../_research/codex/codex-rs/core/src/landlock.rs:15`

### 2.11 Multi-Agent 原方向对，但 v1 只能做 one-shot

Codex 的 subagent 重点仍是 fork-join 和父级审批拦截。当前快照还有 multi-agent v2/AgentControl 相关能力，但 v1 不应追这个扩展面。

v1 只做：

- `spawn_agent(task, role_prompt, allowed_tools, fork_context=false)`
- 子 agent 独立 session/history。
- 子 agent 的审批请求不直接暴露给外层 CLI，而是路由给 parent policy。
- 父 agent 等待子 agent `TurnComplete` 后拿最终文本。

不做：

- agent graph store。
- resume/close/wait 多 agent 生命周期。
- role install/model override。
- 并行多 agent 调度器。

源码依据：
- `../_research/codex/codex-rs/core/src/codex_delegate.rs:65`
- `../_research/codex/codex-rs/core/src/codex_delegate.rs:441`

---

## 3. 对当前计划的具体评审

| 模块 | 当前计划判断 | 需要调整 |
|---|---|---|
| Agent loop | 方向正确 | 加一个 `FakeProvider` 和 deterministic tests，否则后面 debug 会很痛 |
| Protocol | 方向正确 | 不要强调 Op/Event 数量，强调生命周期 |
| Tool schema | 基本正确 | 明确 v1 只做 function-style tool，provider adapter 单独处理 |
| Provider | 过宽 | 先 OpenAI-compatible，Anthropic 第二优先级 |
| Context/Compaction | 核心正确，细节有误 | 去掉“最近 4 轮原样保留”，改为 user messages + summary |
| Rollout/Resume | 核心正确，算法过简 | 增加 rollback + compaction 组合测试 |
| Apply patch | 重要但风险高 | 不强依赖 Lark；先实现可用 parser，再补 lenient cases |
| Approval/Sandbox | 需要拆概念 | 分成 `ApprovalPolicy` 和 `SandboxMode` |
| Multi-agent | 方向正确，范围过宽 | v1 只做 one-shot subagent |
| Benchmark | 目标过满 | Terminal-Bench subset 硬目标，全量和 SWE-bench 改冲刺 |

---

## 4. 修订后的 v1 交付范围

### 4.1 必须交付

1. `lumi exec "<task>"` 可运行真实模型。
2. `lumi` 交互模式可连续多轮输入。
3. tool-call loop 支持 shell/read/write/search/update_plan/apply_patch。
4. JSONL rollout 持久化，`lumi resume <session_id>` 可续会。
5. apply_patch 能处理常见新增、删除、更新、移动文件。
6. approval 和 cwd guard 防止明显危险操作。
7. Terminal-Bench subset 可以跑，并输出报告。
8. README 能解释 5 大能力分别如何实现。

### 4.2 应该交付

1. compaction：手动 `/compact` + token 超限前自动 compact。
2. `spawn_agent` one-shot 子 agent demo。
3. Kimi/GLM/OpenAI 任一 OpenAI-compatible provider 可通过配置切换。
4. HumanEval smoke 20 题，用来验证 headless runner 稳定。

### 4.3 冲刺交付

1. Terminal-Bench 全量。
2. SWE-bench Lite 1-3 题。
3. Anthropic native provider。
4. asciinema demo。
5. 更完整的 multi-agent lifecycle：wait/resume/close。

---

## 5. 修订后的 14 天计划

> 当前日期是 2026-05-10。下面按 2026-05-10 到 2026-05-23 排，保留一天 buffer。若从 2026-05-09 算 Day 1，则整体顺延理解即可。

| Day | 日期 | 任务 | 验收标准 |
|---|---|---|---|
| D1 | 5/10 | 完成 Codex 调研评审，锁定 v1 范围 | 本文档完成；现有设计文档待修订点明确 |
| D2 | 5/11 | 建 protocol + FakeProvider + EventBus + 基础 tests | fake model 能驱动一个 `UserInput -> AgentMessage -> TurnComplete` |
| D3 | 5/12 | OpenAI-compatible provider + headless `lumi exec` | `lumi exec "say hi"` 调通一个真实 provider |
| D4 | 5/13 | ToolRouter + shell/read/write/search/update_plan | 模型能列目录、读文件、写文件、更新 plan |
| D5 | 5/14 | run_turn tool-call continuation + approval/sandbox 拆分 | 工具调用后能继续向模型回传 result；危险 shell 会被拦截 |
| D6 | 5/15 | apply_patch parser/apply + 单元测试 | add/delete/update/move 四类 patch 测试通过 |
| D7 | 5/16 | RolloutWriter + session list/replay | 每个 turn 写 JSONL；能离线重放事件和 response items |
| D8 | 5/17 | RolloutReader + resume | 有/无 compaction 的 session 都能续会 |
| D9 | 5/18 | Compaction v1 | fake summarizer 测试通过；真实模型手动 `/compact` 可用 |
| D10 | 5/19 | HumanEval smoke + runner 基础设施 | 20 题可批量跑，失败能落日志 |
| D11 | 5/20 | Terminal-Bench subset 接入 | 10 题 subset 跑完，至少产出 pass/fail 报告 |
| D12 | 5/21 | 修 benchmark 暴露的问题 | 对失败分类：工具、上下文、provider、prompt、环境 |
| D13 | 5/22 | `spawn_agent` one-shot + multi-agent demo | Researcher/Writer/Reviewer demo 能完成一次 |
| D14 | 5/23 | README、学习笔记、结果报告、收尾 | README 展示架构、用法、benchmark 表、后续计划 |

---

## 6. 关键实现顺序

### 6.1 先写 FakeProvider

不要一开始就绑真实模型。FakeProvider 可以返回预设 chunks：

```text
assistant_delta("我先看文件")
tool_call("shell", {"cmd": "ls"})
assistant_delta("完成")
```

这样可以稳定测试：

- streaming delta
- tool call parse
- tool result 回填
- continuation loop
- rollout 写入顺序
- approval request/response

### 6.2 Provider 先只做 OpenAI-compatible

优先支持：

```toml
[provider.openai_compatible]
base_url = "..."
api_key_env = "..."
model = "..."
wire_api = "chat_completions"
```

OpenAI Responses API、Anthropic native 都可以后置。v1 的学习重点不是 provider 覆盖率，而是 loop 和 session。

### 6.3 Tool 最小集

| Tool | 必要性 | 说明 |
|---|---|---|
| `shell` | 必须 | Terminal-Bench 主力 |
| `read_file` | 必须 | 降低 shell cat 依赖 |
| `write_file` | 必须 | 简单写文件 |
| `search` | 必须 | rg 包装 |
| `apply_patch` | 必须 | SWE/代码修改核心 |
| `update_plan` | 必须 | 规划能力展示 |
| `spawn_agent` | 应该 | multi-agent demo |

### 6.4 Rollout schema 不要过度抽象

v1 schema 就用 5 类：

```text
SessionMeta
TurnContext
ResponseItem
EventMsg
Compacted
```

每行 JSON:

```json
{"type":"response_item","payload":{...}}
```

关键是保证 `ResponseItem` 能无损重建模型输入。

### 6.5 Compaction 不追求一步到位

第一版：

```text
selected_user_messages + assistant_summary
```

第二版再补：

```text
initial_context injection
manual/pre-turn/mid-turn 区分
```

只要 `replacement_history` 写入 rollout，resume 就已经学到 Codex 最有价值的机制。

---

## 7. Benchmark 策略

### 7.1 不要把 HumanEval 当核心指标

HumanEval 更像 provider 能力测试，不太能证明 agent 架构。建议只跑 20 题 smoke：

- 验证 headless runner。
- 验证 workspace 创建。
- 验证测试执行。
- 验证结果汇总。

### 7.2 Terminal-Bench 是主线

先固定 10 题 subset。每题记录：

- task id
- provider/model
- pass/fail
- 总 token
- shell 次数
- apply_patch 次数
- 失败原因分类

交付时重点不是高分，而是能说清：

```text
哪些任务失败是模型能力问题，哪些是工具/上下文/session 问题。
```

### 7.3 SWE-bench Lite 作为冲刺

SWE-bench Lite 环境成本高，且和 apply_patch、repo search、test selection 强相关。14 天内可以只选 1-3 道非常小的题作为 demo，不建议承诺 5-10 题。

---

## 8. 建议修改原设计文档的点

1. 把 §2 的 Op/Event 数量改成“当前快照超过 30/70，v1 只复刻生命周期核心”。
2. 把 §3.1 的 submission channel cap 从具体数字删除，或标注当前快照是 `512`。
3. 把 §5.3 compaction 的“recent n=4”删掉，改成 Codex 的 `collect_user_messages + summary`。
4. 把 §5.4 resume 算法补充 rollback segment 复杂性，v1 标成简化实现。
5. 把 §6.2 apply_patch 改成“兼容 Codex patch format”，不要写死必须 Lark。
6. 把 §6.3 Approval 拆成 `ApprovalPolicy` 与 `SandboxMode`。
7. 把 §7 provider 优先级改成 OpenAI-compatible first，Anthropic native second。
8. 把 §9 benchmark 目标下调：Terminal-Bench subset 是硬目标，全量/SWE-bench 是冲刺项。
9. 把 §11 时间表按本文 D1-D14 更新。
10. 增加 FakeProvider 和 deterministic tests，这是原计划缺口。

---

## 9. 最终推荐

这个项目最适合被包装成：

> “我用 Python 复刻 Codex core 架构，重点实现了 event-driven agent loop、tool routing、rollout/resume、compaction 和 one-shot subagent，并用 Terminal-Bench subset 验证。”

这比“我做了一个大而全的终端 agent，但每块都不稳定”更有说服力。面向即将入职的 agent 开发工作，最有价值的不是 benchmark 数字本身，而是你能从源码到实现讲清楚这些问题：

1. 为什么 Codex 的主循环是 submission/event，而不是一个同步 while ReAct。
2. 为什么 planning 被设计成 tool，而不是单独 planner。
3. 为什么 session 要写 rollout JSONL，而不是只存最终 history。
4. 为什么 compaction 要写 `replacement_history`，以及 resume 如何利用它。
5. 为什么 subagent 的审批必须由 parent 拦截。
6. 为什么 tool schema 要统一，provider 差异要留在 adapter 层。

把这些做扎实，lumi-codex 就足够成为一个很好的入职前学习项目。
