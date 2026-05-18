---
title: "从流水线到工作流：AI 知识库 V3 的多 Agent 进化"
date: 2026-05-18T10:00:00+08:00
tags: ["AI", "Agent", "LLM", "Multi-Agent", "LangGraph", "工作流"]
categories: ["技术"]
summary: "AI 知识库从 V2 的线性流水线，进化到 V3 的 LangGraph 多 Agent 工作流。引入审核循环、条件路由、设计模式和生产加固——探讨为什么线性 Pipeline 不够用，以及状态图如何解决它的问题。"
ShowToc: true
---

## V2 的局限：线性流水线的天花板

上一篇文章（[从零代码到工程化：AI 知识库 V2 的进化之路](../ai-knowledge-base-v2-pipeline/)）展示了 V2 如何把 V1 的声明式 Agent 升级为 Pipeline + Hooks + CI/CD 的工程化体系。四步流水线（Collect → Analyze → Organize → Save）解决了自动化和成本控制的问题。

但线性流水线有一个根本缺陷：**它没有反馈回路。**

V2 的质量校验是单向的——Organizer 检查格式、去重、打分，然后要么通过要么丢弃。如果一篇文章的摘要不够好、标签不够精准、技术深度不够，它不会被改进，而是直接被标成 C 级然后忽略。这就像一个只做终检的工厂——缺陷产品直接报废，而不是回到产线返修。

具体来说，V2 面临三个线性流水线无法解决的问题：

**质量不可迭代。** 分析结果一旦生成，就无法被改进。LLM 可能对某篇论文只给了一句泛泛的摘要，但 Organizer 的五维评分发现摘要质量只有 3/10——没有机制让 LLM "再试一次"。

**策略不可调节。** 无论今天要采集 5 条还是 50 条，流水线的参数都是写死的（或靠命令行参数手动传入）。没有"总指挥"来根据目标自动调整采集深度、审核力度、过滤阈值。

**异常不可处理。** GitHub API 限流、RSS 源挂掉、LLM 返回无效 JSON——V2 的做法是跳过错误继续跑。但如果连续几次分析质量都很差呢？流水线会默默产出大量低质内容，没有机制暂停并通知人工。

这三个问题指向同一个方向：**需要从线性管道进化为带反馈的状态图。**

## V3 的核心架构：LangGraph 状态图

V3 引入 [LangGraph](https://github.com/langchain-ai/langgraph) 来编排多 Agent 工作流。核心变化可以用一张拓扑图概括：

```
① plan → ② collect → ③ analyze → ④ review ┬─[pass]────→ ⑥ organize → END
                                              │
                                              ├─[fail]────→ ⑤ revise → ④ review（循环）
                                              │
                                              └─[>max]────→ ⑦ human_flag → END
```

和 V2 的线性管道相比，V3 多了三个关键机制：

| 机制 | 对应节点 | V2 中的对应物 |
|------|---------|-------------|
| **动态规划** | Planner (①) | 无，参数写死在命令行 |
| **审核循环** | Reviewer (④) + Reviser (⑤) | 无，Organizer 只做终检 |
| **优雅降级** | HumanFlag (⑦) | 无，错误被静默跳过 |

### 所有节点共享同一个状态

V2 的流水线是"函数调用链"——每个步骤的输出作为下一个步骤的输入，中间没有共享状态。V3 用 LangGraph 的 `TypedDict` 定义全局状态：

```python
class KBState(TypedDict):
    plan: dict               # Planner 输出的策略
    sources: list[dict]      # Collector 采集的原始数据
    analyses: list[dict]     # Analyzer 分析后的结构化结果
    articles: list[dict]     # Organizer 整理后的知识条目
    review_feedback: str     # Reviewer 的反馈意见
    review_passed: bool      # 审核是否通过
    iteration: int           # 当前审核循环次数
    needs_human_review: bool # 是否需要人工介入
    cost_tracker: dict       # Token 用量追踪
```

每个节点只修改自己负责的字段。Planner 写 `plan`，Collector 写 `sources`，Analyzer 写 `analyses`——职责清晰，互不干扰。条件路由（`review_passed`）决定了工作流的走向：通过 → 整理入库，未通过 → 定向修改后重来。

这个状态设计遵循一个原则：**数据属于图，不属于节点。** 没有哪个节点"持有"数据，所有节点都从共享状态读取、向共享状态写入。这使得任意两个节点之间的依赖关系由图的拓扑决定，而非代码调用顺序。

## 设计决策：为什么这样分 Agent

### 职责隔离：每个 Agent 只做一件事

V3 的 7 个节点各自遵循一个核心原则：

| Agent | 原则 | 为什么 |
|-------|------|-------|
| Planner | Plan, don't execute | 规划和执行分离，策略可独立测试 |
| Collector | Collect, don't analyze | 不做 LLM 调用，只管抓数据 |
| Analyzer | Analyze one at a time | 每条数据独立分析，保持职责单一 |
| Reviewer | Evaluate, don't modify | 只评估不修改，避免"自己改自己" |
| Reviser | Modify, don't evaluate | 只修改不评估，和 Reviewer 形成"审核-修改"对 |
| Organizer | Organize, don't review | 只负责格式化和写盘，不做质量判断 |
| HumanFlag | Fail gracefully | 优雅降级，不污染主知识库 |

这里最关键的设计是 **Reviewer 和 Reviser 的分离**。如果把审核和修改放在同一个 Agent 里，它就会陷入"自己给自己打高分"的利益冲突——改了之后立刻自己审，没有外部压力促使其认真改进。拆成两个独立的 Agent，Reviewer 的反馈（`review_feedback`）才真正有约束力。

### 条件路由：审核循环的核心

V3 工作流最核心的代码是 `route_after_review` 这个条件路由函数：

```python
def route_after_review(state: KBState) -> str:
    """review 节点之后的三路分支"""
    plan = state.get("plan", {}) or {}
    max_iter = int(plan.get("max_iterations", 3))
    iteration = state.get("iteration", 0)

    if state.get("review_passed", False):
        return "organize"      # 审核通过 → 整理入库
    elif iteration >= max_iter:
        return "human_flag"    # 超过上限 → 人工介入
    else:
        return "revise"        # 还有机会 → 定向修改
```

三路分支对应三种结局：

1. **通过（organize）**：加权总分 ≥ 7.0，进入整理入库（正常终点）
2. **修改（revise）**：未通过但迭代次数未达上限，Reviser 根据 Reviewer 反馈定向改进，然后回到 Reviewer 重新评分
3. **人工（human_flag）**：超过最大迭代次数仍未通过，数据写入 `pending_review/` 目录等待人工处理

这个循环最多跑 3 次（Full 策略）。实测中大部分内容在 1-2 轮就能通过审核。第 3 轮还没通过的，通常是数据本身的问题（比如标题就是营销文，怎么改摘要都没用），这时候 HumanFlag 的"请人来判断"比继续让 LLM 瞎改更合理。

### 加权评分：代码算总分，不让 LLM 做算术

Reviewer 的评分体系采用 5 维度加权：

```python
REVIEWER_WEIGHTS = {
    "summary_quality": 0.25,  # 摘要质量
    "technical_depth": 0.25,  # 技术深度
    "relevance":       0.20,  # 相关性
    "originality":     0.15,  # 原创性
    "formatting":      0.15,  # 格式规范
}
```

一个关键设计决策：**LLM 只给每个维度打分，加权总分用代码计算。**

```python
# 用代码重算加权总分，不信任模型算术
scores = result.get("scores", {})
weighted_total = sum(
    scores.get(dim, 0) * w for dim, w in REVIEWER_WEIGHTS.items()
)
```

为什么不让 LLM 直接算总分？因为 LLM 的算术不可靠。GPT-4 做三位数乘法都可能出错，更不用说带权重的加权求和。把权重写在代码里（而不是 prompt 里）还有一个好处：改权重不用改 prompt，也不怕模型看到权重后"照顾"弱项维度。

## Planner：从写死参数到动态策略

V2 的流水线参数全靠命令行传入或环境变量写死。V3 引入 Planner 节点，根据目标采集量自动选择策略：

```python
def plan_strategy(target_count: int) -> dict:
    if target_count >= 20:
        return {"strategy": "full", "per_source_limit": 20,
                "relevance_threshold": 0.4, "max_iterations": 3}
    elif target_count >= 10:
        return {"strategy": "standard", "per_source_limit": 10,
                "relevance_threshold": 0.5, "max_iterations": 2}
    else:
        return {"strategy": "lite", "per_source_limit": 5,
                "relevance_threshold": 0.7, "max_iterations": 1}
```

三档策略覆盖不同场景：

| 策略 | 每源采集 | 过滤阈值 | 审核迭代 | 适用场景 |
|------|---------|---------|---------|---------|
| **Full** | 20 条 | 0.4 | 3 次 | 周末深度采集，质量优先 |
| **Standard** | 10 条 | 0.5 | 2 次 | 日常运行，平衡成本和质量 |
| **Lite** | 5 条 | 0.7 | 1 次 | 快速冒烟测试，成本优先 |

Planner 不执行任何操作，它只把策略写入 `state["plan"]`。下游的 Collector 读 `per_source_limit` 决定抓多少条，Organizer 读 `relevance_threshold` 决定过滤标准，Reviewer 读 `max_iterations` 决定循环上限。**一个规划影响三个节点的行为，但不直接调用任何节点。**

这是 LangGraph 状态图的威力：节点之间不直接通信，它们通过共享状态间接协调。Planner 改了策略，下游节点"自动"跟着变——无需修改任何调用代码。

## Agent 设计模式：Router 和 Supervisor

V3 除了主工作流，还在 `patterns/` 目录下演示了两种经典的 Agent 协作模式。

### Router 模式：意图路由

Router 是最常见的 Agent 模式——接收请求，分类意图，路由到对应处理器：

```python
INTENT_HANDLERS = {
    "github_search":    github_search_handler,    # GitHub 搜索
    "knowledge_query":  knowledge_query_handler,   # 知识库查询
    "trending":         trending_handler,          # 热门推荐
    "general_chat":     general_chat_handler,      # 通用对话
}
```

分类策略采用**关键词优先 + LLM 兜底**的两级方案：

```python
def classify_intent(query: str) -> str:
    # 第一级：关键词匹配（零成本、确定性高）
    if any(kw in query for kw in ["github", "仓库", "repo"]):
        return "github_search"
    if any(kw in query for kw in ["知识库", "查", "搜索"]):
        return "knowledge_query"
    ...

    # 第二级：LLM 分类（覆盖长尾、模糊请求）
    result, _ = chat_json(f"分类意图: {query}", ...)
    return result.get("intent", "general_chat")
```

80% 的请求被关键词直接命中，剩下 20% 交给 LLM。这种分层策略在成本和覆盖面之间取得平衡。

### Supervisor 模式：主管调度

Supervisor 模式模拟真实团队：一个主管 Agent 负责任务分解和调度，多个工人 Agent 各自完成子任务：

```python
def supervisor_run(query: str) -> dict:
    # Step 1: 主管分析任务，分解为子任务
    tasks = plan_tasks(query)  # LLM 生成任务列表

    # Step 2: 并行分发给工人
    results = []
    for task in tasks:
        if task["worker"] == "collector":
            results.append(collector_worker(task))
        elif task["worker"] == "analyzer":
            results.append(analyzer_worker(task))

    # Step 3: 主管汇总并做最终决策
    return synthesize(query, results)  # LLM 综合分析
```

Router 和 Supervisor 的区别：

| 维度 | Router | Supervisor |
|------|--------|------------|
| 分发方式 | 1 对 1 | 1 对多 |
| 任务关系 | 互斥（只选一个处理器） | 可并行（多个工人同时工作） |
| 结果处理 | 直接返回处理器的输出 | 汇总多个工人的结果 |
| 适用场景 | 多功能路由（搜索/查询/聊天） | 复杂任务的分拆执行 |

## 生产加固：成本、安全、评估

V3 在 V2 的基础上新增了三个生产级保障模块。

### 成本守卫：Token 预算的三级防护

LLM 调用成本在生产环境中是核心关注点。V3 的 `CostGuard` 提供追踪 → 预警 → 熔断三级防护：

```python
guard = CostGuard(budget_yuan=1.0)  # 单次运行预算 1 元

guard.record("analyze", usage)      # 记录每次调用
status = guard.check()              # 检查是否超标

# status 可能是:
# "normal"      — 预算充足，继续
# "alert"       — 接近预算（默认 80%），发出警告
# "exceeded"    — 超出预算，抛出 BudgetExceededError 熔断
```

V2 也有成本估算（`estimate_cost`），但只是事后统计。V3 的 CostGuard 是**前置防护**——在每次 LLM 调用之前检查预算，超了就停，而不是跑完了才发现超支。

### 安全模块：防注入 + PII 检测 + 限流

Agent 系统直接暴露用户输入给 LLM，必须防范三类安全风险：

```python
# 1. 防 Prompt 注入 — 检测常见注入模式
cleaned, warnings = sanitize_input(user_input)
# 检测: "忽略之前所有指令"、"you are now a hacker"、"[INST]" 等

# 2. PII 检测 — 过滤输出中的敏感信息
safe_text, pii_found = filter_output(llm_output)
# 检测: 手机号、邮箱、身份证号、信用卡号、IP 地址

# 3. 速率限制 — 防止高频调用导致成本失控
limiter = RateLimiter(max_calls=10, window_seconds=60)
if not limiter.allow("user_123"):
    return "请求过于频繁，请稍后再试"
```

审计日志记录所有安全事件，出问题可以追溯到具体时间、节点、输入内容。

### 评估测试：LLM-as-Judge

V3 引入 `pytest` 格式的评估测试，用 LLM 来评估 LLM 的输出质量：

```python
@pytest.mark.skipif(not os.getenv("LLM_API_KEY"), reason="需要 API Key")
def test_summary_quality(sample_article):
    """请 LLM 评估摘要质量（1-5 分）"""
    result, _ = chat_json(f"评估摘要质量: {sample_article['summary']}")
    score = result.get("score", 0)
    assert score >= 3, f"摘要质量不达标: {score}/5"
```

V2 的质量评估是"代码规则 + LLM 评分"的静态检查。V3 加入了"LLM 评估 LLM"的动态测试——用另一个 LLM 调用来验证生成内容的语义质量，而不只是格式是否合规。

## V2 vs V3：架构对比总览

| 维度 | V2 | V3 |
|------|----|----|
| **编排方式** | 线性 Pipeline（函数调用链） | LangGraph 状态图（节点 + 条件边） |
| **质量保障** | Organizer 终检（单向） | Reviewer + Reviser 审核循环（双向反馈） |
| **策略控制** | 命令行参数 / 环境变量 | Planner 动态策略（lite/standard/full） |
| **异常处理** | 静默跳过错误 | HumanFlag 优雅降级 + 审计日志 |
| **设计模式** | 无 | Router（意图路由）+ Supervisor（主管调度） |
| **成本控制** | 事后估算 | 前置预算防护（追踪/预警/熔断） |
| **安全防护** | 无 | 防注入 + PII 检测 + 速率限制 |
| **测试覆盖** | Hook 校验脚本 | pytest + LLM-as-Judge 评估测试 |
| **Agent 数量** | 3 个（Collector/Analyzer/Organizer） | 7 个（+Planner/Reviewer/Reviser/HumanFlag） |
| **共享状态** | 无（管道传递） | KBState TypedDict（全局状态） |

## 核心收获

### 从管道到状态图

线性管道（Pipeline）是最直观的编排方式——A 的输出喂给 B，B 的输出喂给 C。但它无法表达反馈、条件分支和动态路由。LangGraph 的状态图让每个节点成为独立的"决策单元"，通过共享状态通信，通过条件边实现流程控制。

核心差异不在于"多了一个循环"，而在于**控制流的声明方式**。V2 的控制流是隐式的——代码调用顺序决定了执行顺序。V3 的控制流是显式的——图的拓扑定义了所有可能的路径，包括循环和分支。

### 职责隔离的设计原则

V3 的每个 Agent 都遵循"只做一件事"的原则：

- **Planner 只规划不执行** — 策略写入状态，下游节点自行解读
- **Reviewer 只评估不修改** — 给分和反馈，但不碰数据
- **Reviser 只修改不评估** — 根据反馈改，但不给自己打分
- **Organizer 只整理不审核** — 格式化写盘，不做质量判断

这种隔离使得每个 Agent 都可以独立测试、独立替换、独立调优。Reviewer 的评分阈值从 7.0 调到 8.0，不需要改 Reviser 的任何代码。

### 生产级 Agent 系统的必备组件

V3 展示了一个生产级 Agent 系统至少需要：

1. **成本守卫** — LLM 按 token 计费，没有预算控制就是在裸奔
2. **安全防护** — Agent 直接暴露用户输入，注入攻击是真实威胁
3. **评估测试** — LLM 输出不确定，必须用自动化测试兜底
4. **优雅降级** — 自动化不可能覆盖 100%，必须有"请人来判断"的出口

V2 解决了"能不能自动跑"的问题。V3 解决的是"能不能在生产环境可靠跑"的问题。
