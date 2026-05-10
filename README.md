# lumi-codex

> 一个用 Python 实现的终端 agent —— 复刻 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构

## 项目目标

一个学习 agent 架构的个人项目，覆盖：

- **Agent Loop** —— channel-driven submission/event 总线 + tool-call continuation
- **Task 规划** —— `update_plan` tool（让模型自维护 checklist）
- **Context 管理** —— MessageHistory + 独立 LLM call 摘要 compaction
- **Session 管理** —— JSONL rollout + 反向扫 resume
- **Multi-Agent 编排** —— one-shot SubagentDelegate fork-join + 父级审批拦截

设计原则：**复刻 codex 的核心机制，不复刻其工程外延**。

## 文档导航

| 文档 | 内容 |
|---|---|
| [`docs/architecture.md`](./docs/architecture.md) | 整体架构、关键决策、目录结构、配置 |
| [`docs/implementation.md`](./docs/implementation.md) | 每个模块的实现细节（伪代码 + 关键设计点） |
| [`docs/benchmark.md`](./docs/benchmark.md) | Benchmark 集成 + Multi-Agent demo |
| [`docs/building.md`](./docs/building.md) | 按依赖关系组织的构建顺序 |
| [`docs/codex-source-map.md`](./docs/codex-source-map.md) | codex 真实源码定位（参考） |
| [`docs/learning/`](./docs/learning/) | 学习 codex 源码的笔记 |

## 关键技术决策

| 项 | 选择 |
|---|---|
| 语言 | Python（`uv` + `ruff` + `pytest`） |
| 模型 Provider | OpenAI-compatible（一份代码覆盖 OpenAI / Kimi / GLM）+ Anthropic native |
| Benchmark | Terminal-Bench（主）+ SWE-bench Lite + HumanEval（smoke） |
| Tool 格式 | function-style `ToolSpec`，provider 差异留在 adapter 层 |
| Apply patch | 手写 lenient parser，兼容 codex patch 格式 |
| Sandbox | ApprovalPolicy + cwd guard + 命令黑名单（不是 OS 沙箱） |
| Context | 独立 LLM call 摘要 compaction，结构 `selected_user_messages + assistant_summary` |
| Rollout | JSONL，反向扫找 `Compacted.replacement_history` 作 checkpoint |

详见 [`docs/architecture.md`](./docs/architecture.md)。

## 快速开始

```bash
# 安装
uv sync

# 配置 API key
cp .env.example .env
# 编辑 .env 填入 MOONSHOT_API_KEY 或其他 provider 的 key

# 交互模式
lumi

# headless 模式（benchmark 用）
lumi exec "list files in current directory"

# 续会
lumi resume <session_id>

# 跑 benchmark
python -m benchmarks.runner --bench terminal_bench --provider kimi
```

## 仓库结构

```
lumi/         # 主包：core / protocol / tools / models / rollout / sandbox / cli
benchmarks/   # HumanEval / Terminal-Bench / SWE-bench Lite 接入
demos/        # Multi-Agent demo（Researcher + Writer + Reviewer）
docs/         # 架构 + 实现 + benchmark + 构建顺序 + codex 源码定位
tests/        # unit + integration
scripts/
```

详见 [`docs/architecture.md` 的目录结构](./docs/architecture.md#仓库目录结构)。

## License

MIT
