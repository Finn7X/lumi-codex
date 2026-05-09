# lumi-codex

> 一个用于学习的 mini-codex —— 在终端中运行的 Python 实现的 agent，复刻 [OpenAI Codex CLI](https://github.com/openai/codex) 的核心架构

## 项目背景

一个学习 agent 架构的个人项目。目标：

1. 自己构建一个终端 agent，覆盖 **Agent Loop / Task 规划与分发 / Context 管理 / Session 管理 / Multi-Agent 编排** 5 大能力
2. 在主流 benchmark（Terminal-Bench / SWE-bench Lite / HumanEval）上跑出合理分数
3. 通过实现，对 codex 架构形成第一手深入理解
4. 完工后可选用 Rust 重写 `core` 作为 v2

## 当前状态

🚧 **Phase 1 · Day 0 · 设计阶段**

- 时间线：2026-05-09 → 2026-05-22（14 天）
- 当前阶段：设计文档定稿，准备进入学习 + 构建

## 文档导航

- **[设计文档（spec）](./docs/superpowers/specs/2026-05-09-lumi-codex-design.md)** —— 完整的架构、决策、组件设计、时间表
- **[学习笔记（7 篇）](./docs/learning/)** —— Day 1-3 学习 codex 源码后的产出（待写）

## 关键决策

| 决策项 | 选择 |
|---|---|
| 主语言 | Python (v1) → Rust (v2，14 天后可选) |
| Benchmark | Terminal-Bench (主) + SWE-bench Lite 5-10 题 + HumanEval (冒烟) |
| 模型后端 | 多 provider：Claude / Kimi / GLM / OpenAI |
| 时间分配 | 3 天学 codex + 11 天构建 |

## 架构核心

完全基于 codex 真实源码的设计原则：

- **Submission/Event 总线**（asyncio.Queue 复刻 codex 的 tokio mpsc）
- **统一 Tool 格式**（`ResponsesApiTool`，避免 N×M 复杂度）
- **独立 LLM call 摘要 compaction**（保留 `[initial] + [user_msgs] + [summary]` 结构）
- **JSONL Rollout + 反向扫 resume**（找最近 `Compacted.replacement_history` 作 checkpoint）
- **SubagentDelegate fork-join + 父级审批拦截**（codex 的真实多 agent 模式）

## 仓库结构

```
lumi/         # 主包（core / protocol / tools / models / rollout / sandbox / cli）
benchmarks/   # HumanEval / Terminal-Bench / SWE-bench Lite 接入
demos/        # Multi-Agent demo（Researcher + Writer + Reviewer）
docs/         # 设计文档 + 学习笔记
tests/
scripts/
```

详见 [设计文档第 4 节](./docs/superpowers/specs/2026-05-09-lumi-codex-design.md#4-仓库目录结构)。

## 快速开始（v1 完成后）

```bash
# 安装
uv sync

# 配置 API key
cp .env.example .env
# 编辑 .env 填入你的 key

# 交互模式
lumi

# headless 模式
lumi exec "list files in current directory"

# 续会
lumi resume <session_id>

# 跑 benchmark
python -m benchmarks.runner --bench terminal_bench --provider kimi
```

## License

MIT
