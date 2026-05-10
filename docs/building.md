# lumi-codex 构建顺序

按依赖关系组织的构建步骤。每个阶段完成后才能开始下一阶段。

---

## 阶段 1：基础设施

- `protocol/` — Op / EventMsg / RolloutItem / ToolSpec dataclass
- `core/event_bus.py` — `asyncio.Queue` 包装 Submission / EventMsg
- `models/fake.py` — **FakeProvider**，返回预设 chunks 序列
- `tests/unit/` 基础测试框架 + FakeProvider 单测

**验收**：FakeProvider 能驱动一个最小 turn：`UserInput → AgentMessage → TurnComplete`

---

## 阶段 2：真实 Provider

- `models/base.py` — ModelProvider ABC + ResponseStream
- `models/openai_compat.py` — 一份代码覆盖 OpenAI / Kimi / GLM
- `models/stream.py` — SSE → AsyncIterator
- `cli/exec.py` 最小 headless 入口

**验收**：`lumi exec "say hi"` 调通 Kimi 或 GLM；流式 delta 能正确渲染

---

## 阶段 3：Tool 闭环

- `core/tool_router.py` — ToolRouter（统一 ToolSpec 派发）
- `tools/registry.py` — `@register_tool` 装饰器
- `tools/shell.py` / `file.py` / `search.py` / `update_plan.py`
- `core/thread.py::_run_turn` — tool-call continuation loop
- `core/approval.py` — ApprovalPolicy（3 档）
- `sandbox/mode.py` + `cwd_guard.py` + `shell_filter.py` — SandboxMode

**验收**：模型能列目录、读文件、写文件、更新 plan；危险 shell 会被拦截

---

## 阶段 4：Apply Patch

- `tools/apply_patch.py` — 手写 lenient parser
- 实现顺序：Add File → Delete File → Update File → Move File → lenient parsing
- parser 与 apply 行为分开测试，每类 hunk 至少 3 个 case

**验收**：4 类 patch（add / delete / update / move）单元测试全通过

---

## 阶段 5：Rollout

- `rollout/schema.py` — 5 种 RolloutItem dataclass
- `rollout/writer.py` — 写 JSONL（每个事件 / response item / compacted 一行）
- `rollout/reader.py` — 反向扫 + replacement_history checkpoint
- `core/session.py` — SessionState 管理 + 整合 writer/reader
- CLI：`lumi resume` / `lumi replay` / `lumi sessions list`

**验收**：4 类必测场景全通过：
1. 无 compaction
2. 有 compaction
3. 有 rollback
4. compaction + rollback 组合

---

## 阶段 6：Compaction

- `core/compaction.py` — 独立 LLM call 摘要
- `core/context.py` — `approx_token_count`（chars/4）
- 与 RolloutWriter 集成，写入 `RolloutItem.Compacted`

**验收**：FakeProvider 模拟 summarizer 测试通过；真实模型手动 `Op.Compact` 能工作

---

## 阶段 7：Benchmark

- `benchmarks/runner.py` — 通用 Runner
- `benchmarks/humaneval/` — 20 题 smoke test
- `benchmarks/terminal_bench/` — 10 题 subset 接入
- 失败 case 分析报告

**验收**：HumanEval 20 题能跑完；Terminal-Bench 10 题 subset 能跑完，输出 pass/fail 报告

---

## 阶段 8：Multi-Agent

- `core/delegate.py` — SubagentDelegate（spawn_interactive + spawn_one_shot）
- `tools/spawn_agent.py` — spawn_agent tool
- `demos/multi_agent_writer/` — Researcher + Writer + Reviewer demo

**验收**：multi_agent_writer demo 能完整跑一次产出博客

---

## 阶段 9：扩展（按需）

- `models/anthropic.py` — Anthropic native provider（adapter）
- Terminal-Bench 全量 89 题
- SWE-bench Lite 1-3 题
- 模型对照跑分（同 subset × 多个 provider）

---

## 阶段 10：收尾

- README 完善：架构图、用法、benchmark 报告表
- 学习笔记 7 篇产出（见 `docs/learning/`）
- asciinema demo 录制
