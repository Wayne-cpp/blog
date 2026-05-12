---
title: "从零代码到工程化：AI 知识库 V2 的进化之路"
date: 2026-05-12T10:00:00+08:00
tags: ["AI", "Agent", "LLM", "自动化", "Pipeline", "MCP"]
categories: ["技术"]
summary: "AI 知识库从 V1 的纯 Agent 声明式驱动，进化到 V2 的 Pipeline + Hooks + CI/CD 工程化体系。探讨为什么零代码很好，但有时还是需要写点代码。"
ShowToc: true
---

## V1 的局限：零代码的理想与现实

上一篇文章（[用 AI Agent 构建零代码知识库](../ai-knowledge-base-with-agents/)）介绍了 V1 的设计：纯 Markdown 声明，LLM 充当执行引擎，零行业务代码。这个方案足够优雅——但它有几个绕不过去的问题：

**运行成本高。** V1 每次执行都需要 LLM 推理来协调三个 Agent 的工作。一次完整采集可能消耗上千 tokens 的"协调开销"，还没算上实际分析内容的 tokens。

**不确定性。** 同样的输入，LLM 可能走不同的执行路径。今天采集 20 条，明天可能因为模型"心情不同"采集了 18 条，或者对同一篇文章给出了差异明显的评分。

**无法定时。** 纯 Agent 模式依赖人工在 OpenCode 中触发 `@collector`，不能像 cron job 一样每天自动运行。

**质量校验靠自觉。** V1 的质量门控写在了 Markdown 声明文件里，LLM 是否严格执行取决于 prompt 的引导效果。没有程序化的强制校验。

这些局限指向同一个结论：**声明式系统适合表达意图，但工程化的可靠性需要代码来保障。**

## V2 的核心思路：声明式 + 代码混合

V2 并没有抛弃 V1 的 Agent 架构，而是在其基础上增加了三个工程化层：

```
V1 骨架（Agent 声明）+ Pipeline（自动化流水线）+ Hooks（质量校验）+ CI/CD（定时执行）
```

用一句话概括变化：**V1 让 LLM 既当决策者又当执行者，V2 让 LLM 只做它擅长的事——理解和分析——其余全部交给确定性代码。**

## 四步流水线：从手工到自动

V1 的三阶段（采集 → 分析 → 归档）需要手动触发三个 Agent。V2 用 `pipeline/pipeline.py` 把它们串成一条自动流水线：

```python
# 一行命令搞定 V1 中需要三次手动触发的完整流程
python pipeline/pipeline.py --sources github,rss --limit 20
```

四步流水线的执行逻辑：

```
Collect → Analyze → Organize → Save
   ↓         ↓         ↓
  去重     LLM分析    格式校验
```

关键改进在于**先去重再分析**。V1 中如果 GitHub API 返回 30 条结果，LLM 要逐条判断是否重复。V2 先用确定性代码（URL 精确匹配 + 标题相似度）过滤掉已知条目，只把新内容送给 LLM 分析。这一步就能省掉大量 token 开销。

流水线支持三种数据源：

| 来源 | 实现方式 | 特点 |
|------|---------|------|
| **GitHub** | 搜索 API，按 star 排序 | 覆盖开源项目动态 |
| **RSS** | 配置文件驱动，13 个源 | 覆盖技术博客、研究论文 |
| **手动** | Agent 对话触发（兼容 V1） | 灵活补充特定内容 |

### RSS 配置外置

RSS 数据源用 YAML 配置文件管理，新增数据源改配置不改代码：

```yaml
# rss_sources.yaml
sources:
  - name: "Hacker News (Best)"
    url: "https://hnrss.org/best?q=AI+OR+LLM+OR+agent"
    enabled: true

  - name: "arXiv cs.AI"
    url: "https://rss.arxiv.org/rss/cs.AI"
    enabled: true

  - name: "OpenAI Blog"
    url: "https://openai.com/blog/rss.xml"
    enabled: true
```

13 个数据源覆盖 Hacker News、Reddit、arXiv、OpenAI / Anthropic / Google 博客、Hugging Face、LangChain 等。每个源可以独立开关，V1 中添加新源需要写 Skill 声明文件，V2 只需加两行 YAML。

## 工厂模式的多模型客户端

V1 的模型选择是隐式的——OpenCode 框架自动路由。V2 显式管理 LLM 调用，用工厂模式封装了三种模型：

```python
class LLMProvider(ABC):
    """统一接口，所有模型实现这个协议"""
    def chat(self, messages) -> LLMResponse: ...

class DeepSeekProvider(LLMProvider): ...   # ¥0.14/M tokens
class QwenProvider(LLMProvider): ...       # ¥0.50/M tokens
class OpenAIProvider(LLMProvider): ...     # ¥5.00/M tokens

def create_provider() -> LLMProvider:
    """工厂函数，读环境变量自动切换"""
```

切换模型只需改一行环境变量：

```bash
LLM_PROVIDER=deepseek   # 生产用，便宜
LLM_PROVIDER=openai     # 测试用，质量高
```

### 成本追踪：每一分钱都算得清

每次 LLM 调用都记录 token 用量，并按模型定价实时估算成本：

```python
PRICING = {
    "deepseek-chat":  {"input": 0.0014, "output": 0.0028},  # USD/1K tokens
    "gpt-4o":         {"input": 0.005,  "output": 0.015},
}

def estimate_cost(model: str, usage: Usage) -> float:
    prices = PRICING.get(model, {"input": 0.002, "output": 0.006})
    return usage.prompt_tokens / 1000 * prices["input"] \
         + usage.completion_tokens / 1000 * prices["output"]
```

DeepSeek-Chat 和 GPT-4o 的成本差距高达 35 倍。有了成本追踪，可以量化"用 GPT-4o 分析 20 条文章值不值 ¥1.5" vs "DeepSeek 只要 ¥0.04"。

### 错误重试：API 限流不怕

LLM API 限流是常态。V2 实现了指数退避重试：

```python
def chat_with_retry(provider, messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            return provider.chat(messages)
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 429:  # Rate limited
                wait = 2 ** attempt
                time.sleep(wait)
            else:
                raise
```

V1 中 API 限流会导致整个 Agent 对话中断，需要人工重新触发。V2 自动重试，流水线继续。

## Hooks：程序化的质量校验

V1 的质量门控依赖 LLM "自觉遵守" prompt 中的评分规则。V2 把校验从 prompt 搬到了代码里，用两个 Hook 脚本实现。

### JSON 格式校验（validate_json.py）

纯规则逻辑，不涉及 LLM：

```python
# 校验规则示例
REQUIRED_FIELDS = {"id": str, "title": str, "source_url": str, "summary": str, "tags": list, "status": str}
ID_PATTERN = re.compile(r"^[a-z][\w:-]+-\d{8}-\d{3}$")  # github-20260512-001
VALID_STATUSES = {"draft", "review", "published", "archived"}
```

检查项包括：必填字段存在性、字段类型正确性、ID 格式、URL 格式、摘要最小长度、状态值合法性。全部通过返回 exit code 0，有错误返回 1。

### 五维质量评分（check_quality.py）

这是 V2 的重头戏。五个维度，加权总分 100：

| 维度 | 满分 | 评分逻辑 |
|------|------|---------|
| 摘要质量 | 25 | 长度 + 技术关键词密度 |
| 技术深度 | 25 | 基于 LLM 评分（1-10）映射 |
| 格式规范 | 20 | 必填字段完整性检查 |
| 标签精度 | 15 | 是否在预定义标签集合内 |
| 空洞词检测 | 15 | 检测"赋能""game-changing"等套话 |

等级划分：**A（≥80）/ B（≥60）/ C（<60）**，只有 B 级以上才标记为 `published`。

空洞词检测特别有意思。项目维护了一份中英文双语的"废话词典"：

```python
HOLLOW_WORDS_ZH = ["赋能", "抓手", "闭环", "打通", "全链路", "底层逻辑", "颗粒度", ...]
HOLLOW_WORDS_EN = ["groundbreaking", "revolutionary", "game-changing", "synergy", ...]
```

摘要中每出现一个空洞词扣 3 分。这个机制确保知识库过滤掉营销废话，只保留有实质技术内容的信息。

## CI/CD：每天自动采集

V1 只能手动触发。V2 用 GitHub Actions 实现了每日自动采集：

```yaml
# .github/workflows/daily-collect.yml
on:
  schedule:
    - cron: "0 8 * * *"    # 每天 UTC 8:00（北京时间 16:00）
  workflow_dispatch:         # 也支持手动触发
```

完整流程：

```
定时触发 → 安装依赖 → 运行 Pipeline → 校验文章 → 自动提交
```

关键设计：API Key 全部存在 GitHub Secrets 中，不在代码里暴露。采集结果自动 commit 到仓库，commit message 包含文章数量和日期。

整个 CI/CD 流程的运行成本：**每天不到 ¥0.1**（使用 DeepSeek 作为 LLM 后端）。

## MCP Server：让 AI 工具直接查询知识库

这是 V2 中最"面向未来"的特性。手写了一个 JSON-RPC 2.0 的 MCP Server，提供三个工具：

```python
search_articles(keyword, limit)  # 按关键词搜索
get_article(article_id)          # 按 ID 获取详情
knowledge_stats()                # 统计信息
```

配置到 OpenCode 后，任何支持 MCP 协议的 AI 工具都能直接搜索你的知识库：

```json
// opencode.json
{
  "mcp": {
    "knowledge": {
      "type": "local",
      "command": ["python3", "mcp_knowledge_server.py"]
    }
  }
}
```

这意味着在 Cursor、Claude Desktop、OpenCode 等工具中，你可以直接问"最近有什么关于 MCP 协议的新进展？"，AI 会自动调用 MCP Server 搜索知识库并给出答案。

V1 的知识库是静态的 JSON 文件，只能通过文件系统查询。V2 通过 MCP 协议把知识库变成了 AI 工具的"原生能力"。

## V1 vs V2：对比总结

| 维度 | V1（声明式） | V2（工程化） |
|------|-------------|-------------|
| **代码量** | 零行业务代码 | ~800 行 Python |
| **触发方式** | 手动 @agent | 手动 + Pipeline + CI/CD |
| **数据源** | GitHub（Skill 声明） | GitHub + RSS（YAML 配置） |
| **LLM 管理** | 框架隐式路由 | 工厂模式 + 成本追踪 |
| **质量校验** | Prompt 引导 | 程序化 Hook + 五维评分 |
| **可观测性** | Agent 对话记录 | 日志 + token 统计 + 成本估算 |
| **定时运行** | 不支持 | GitHub Actions 每日自动 |
| **外部查询** | 文件系统 | MCP Server |
| **日运行成本** | 不确定 | ¥0.1 以内 |

## 反思：什么时候该写代码

V1 的零代码方案不是错的，而是**阶段性的**。在项目初期，你不确定流水线该怎么设计、评分维度是什么、哪些数据源有价值——这时候声明式方案帮你快速探索。Markdown 声明文件就是一份活的文档，记录了你对系统的理解。

当这些理解稳定下来——评分维度确定了、数据源固定了、流程不再变了——就该把确定性逻辑从 prompt 搬到代码里。代码提供 LLM 无法保证的东西：**可重复性、可度量性、可自动化。**

V2 的架构本质上是一个**声明式与命令式的混合体**：

- Agent 声明（V1 遗产）→ 管理角色定义和权限边界
- Pipeline 代码（V2 新增）→ 管理执行流程和确定性逻辑
- YAML 配置（V2 新增）→ 管理数据源和模型选择
- MCP Server（V2 新增）→ 管理外部集成

每一层都用最合适的表达方式。Markdown 声明角色意图，Python 保证执行确定性，YAML 管理配置，MCP 协议打通外部工具。

## 总结

V2 的改进不是"V1 不够好所以要重写"，而是"V1 帮我们想清楚了需求，V2 帮我们把需求落地为可靠的工程"。

几个值得带走的设计思路：

- **先用声明式探索，再用代码固化**：Markdown 适合表达意图，Python 适合保证确定性
- **先用 LLM 再去重**：反过来浪费 token，先去重可以把 LLM 调用量降低 50%+
- **质量校验用代码不用 prompt**：五维评分写死在 Python 里，不依赖 LLM 的"理解力"
- **成本追踪从第一天做起**：每次 LLM 调用都算钱，不知道花了多少就不可能优化
- **MCP 协议是 AI 工具的 HTTP**：让知识库可以被任何 AI 工具直接查询，而不是锁在文件系统里

下一步？也许 V3 会把多 Agent 协作做得更深，加入 RAG 检索，或者支持多用户。但无论怎么演进，V1 的声明式骨架 + V2 的工程化肌肉这个混合架构，提供了一个很好的起点。

---

*项目基于 OpenCode 框架，使用 DeepSeek / Qwen 等国产 LLM 作为推理引擎。完整源码可在 [GitHub](https://github.com/huangjia2019/ai-knowledge-base) 查看。*
