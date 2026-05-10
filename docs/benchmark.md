# Benchmark 与 Multi-Agent Demo

lumi-codex 通过 3 个 benchmark 验证 agent 框架，加 1 个独立 demo 展示 multi-agent 编排。

---

## 1. 通用 Runner（`benchmarks/runner.py`）

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

每题记录：

- task id
- provider/model
- pass/fail
- 总 token / shell 次数 / apply_patch 次数
- 失败原因分类（模型能力 / 工具不够用 / 上下文管理差 / provider 问题 / 环境问题）

---

## 2. HumanEval（`benchmarks/humaneval/`）

**用途**：smoke test，验证 headless runner 链路通。

- 数据：`openai/human-eval` HuggingFace dataset，164 题
- Workspace：每题一个临时目录，含 `prompt.py` + `test.py`
- Grade：跑 `pytest test.py` exit code
- 跑 20 题足够，用于验证：headless runner 能批量驱动 / workspace 创建正常 / 测试执行链路通 / 结果汇总正确

---

## 3. Terminal-Bench（`benchmarks/terminal_bench/`）

**用途**：lumi-codex 的主验证目标。

- 数据：`tbench` 的 ~89 个 core tasks
- Workspace：每题一个 Docker 镜像或本地目录
- Grade：tbench 官方 grader

**报告内容**：分类失败原因，能在 README 里说清楚：

> "哪些任务失败是模型能力问题，哪些是工具/上下文/session 问题。"

固定 10 题 subset 作为日常回归。完整 89 题用于最终展示。

---

## 4. SWE-bench Lite（`benchmarks/swebench_lite/`）

**用途**：展示 lumi-codex 能处理真实 patch 工作流。

- 数据：300 题中选 1-3 题（手挑相对简单的 Django/Flask 类）
- Workspace：Docker 镜像（仓库 + 测试环境）
- Grade：跑指定测试，看是否 pass

不追求高分，能跑通即可。重点是验证 apply_patch / repo search / test selection 多组件协作。

---

## 5. 模型对照跑分

用同一套 Terminal-Bench 10 题 subset 跑多个 provider，对照表展示：

| 模型 | Terminal-Bench subset 通过率 | 平均 token | 总成本 |
|---|---|---|---|
| kimi-k2 | ?% | ? | ¥? |
| glm-4.6 | ?% | ? | ¥? |
| gpt-5 | ?% | ? | ¥? |
| claude-sonnet-4-6 | ?% | ? | ¥? |

---

## 6. Multi-Agent Demo（`demos/multi_agent_writer/`）

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

**不参与 benchmark 跑分**。用 asciinema 录 gif，README 里展示 multi-agent 协作过程。
