# Agentic RAG：从被动检索到主动编排

## 1. 什么是 Agentic RAG

Agentic RAG 将 LLM 从**被动的文本生成器**升级为**主动的检索编排者**。LLM Agent 不再只是"接收上下文→生成答案"，而是能够**规划、执行、评估和重试**。

### 传统 RAG vs Agentic RAG
```
传统 RAG:    查询 → 检索 → 生成 → 输出（一次完成）
Agentic RAG: 查询 → 规划 → [子任务→检索↔评估↔重试]* → 综合 → 验证 → 输出
```

## 2. Agentic RAG 核心能力

| 能力 | 描述 | 示例 |
|------|------|------|
| **任务规划** | 将复杂查询拆解为执行步骤 | "对比A和B" → 先查A，再查B，最后对比 |
| **工具使用** | 调用多种检索工具 | 向量搜索、SQL查询、Web搜索、计算器 |
| **自评估** | 判断检索结果是否足够 | 信息不足→换关键词重试 |
| **自纠错** | 发现错误后自动修正 | 检索到矛盾信息→多方验证 |
| **记忆** | 记住多步推理的中间结果 | 第一步结果作为第二步的输入 |

## 3. LangGraph 实现 Agentic RAG

LangGraph 是目前最主流的 Agentic RAG 编排框架：

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class RAGState(TypedDict):
    query: str
    plan: List[str]
    retrieved_docs: List[str]
    answer: str
    quality_score: float

# 构建 Agent 图
workflow = StateGraph(RAGState)

# 定义节点
workflow.add_node("plan", plan_node)          # 规划步骤
workflow.add_node("retrieve", retrieve_node)   # 执行检索
workflow.add_node("evaluate", evaluate_node)   # 评估结果
workflow.add_node("generate", generate_node)   # 生成答案
workflow.add_node("verify", verify_node)       # 验证答案

# 定义边和条件路由
workflow.set_entry_point("plan")
workflow.add_edge("plan", "retrieve")
workflow.add_edge("retrieve", "evaluate")
workflow.add_conditional_edges(
    "evaluate",
    router_function,  # 根据评估结果决定：重新检索 or 继续生成
    {"retrieve": "retrieve", "generate": "generate"}
)
workflow.add_edge("generate", "verify")
workflow.add_conditional_edges(
    "verify",
    quality_gate,     # 质量不达标→重新生成，达标→结束
    {"generate": "generate", END: END}
)

app = workflow.compile()
```

## 4. Agent 工具设计

### 4.1 常用工具集

```python
tools = [
    {
        "name": "vector_search",
        "description": "在向量知识库中搜索与查询语义相关的文档",
        "parameters": {"query": "搜索查询", "top_k": "返回数量"}
    },
    {
        "name": "sql_query",
        "description": "查询结构化数据库",
        "parameters": {"sql": "SQL 查询语句"}
    },
    {
        "name": "web_search",
        "description": "搜索互联网获取最新信息",
        "parameters": {"query": "搜索关键词"}
    },
    {
        "name": "calculator",
        "description": "进行数学计算",
        "parameters": {"expression": "数学表达式"}
    }
]
```

### 4.2 ReAct 模式（Reasoning + Acting）

```
思考: 我需要找到iPhone 15的发售价和当前折扣价
行动: vector_search("iPhone 15 价格 发售价")
观察: 检索到3条结果，其中2条提到发售价5999元，但没提到折扣
思考: 还需要查找当前的折扣信息
行动: web_search("iPhone 15 2025年6月折扣价")
观察: 京东当前售价4999元
思考: 我有了发售价和当前价，可以用计算器算折扣比例
行动: calculator("(5999-4999)/5999*100")
观察: 16.67%
思考: 信息完全，可以生成最终答案
```

## 5. NVIDIA Agentic RAG Blueprint（v2.6）

2026 年 6 月的最新方案：
- **LangGraph 规划-执行管线**
- 流式输出阶段事件和推理过程
- 按请求可开启 Agentic 模式（`agentic: true`）
- 可选验证阶段

```yaml
# NVIDIA blueprint 配置模式
pipeline:
  mode: agentic  # 或 standard
  stages:
    - scope_discovery    # 分析查询范围
    - parallel_subtasks  # 并行执行子任务
    - synthesis          # 综合结果
    - verification       # 可选验证
  streaming: true
  reasoning_trace: true
```

## 6. Agentic RAG 的设计考量

### 何时启用 Agentic 模式
- ✅ 需要多步推理的复杂查询
- ✅ 信息分布在多个知识源的查询
- ✅ 需要计算/比较/分析的任务
- ❌ 简单事实查询（成本浪费）
- ❌ 对延迟敏感的实时场景（Agent 循环增加延迟）

### 避免无限循环
```python
# 设置最大迭代次数和超时
MAX_ITERATIONS = 10
TIMEOUT_SECONDS = 30

def agentic_loop(query):
    iteration = 0
    while iteration < MAX_ITERATIONS:
        # ... agent 逻辑 ...
        iteration += 1

        if quality_score > threshold:
            break

    return answer
```

### 成本控制
| 模式 | Token 消耗 | 延迟 |
|------|-----------|------|
| 标准 RAG | 基准 | 基准 |
| Agentic（2-3轮） | 2-3x | 2-4x |
| Agentic（复杂查询 5+轮） | 5-10x | 5-10x |

> 建议：使用轻量路由判断是否需要 Agentic 模式，简单查询走快速通道。
