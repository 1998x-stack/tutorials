# 查询优化与重写技术

## 1. 用户查询的典型问题

现实中的用户查询往往不规范，直接用于检索效果很差：

| 问题类型 | 示例 | 检索困难 |
|---------|------|---------|
| 过于简短 | "怎么做" | 缺少上下文 |
| 口语化表述 | "那个能把文档变成向量的东西" | 术语不匹配 |
| 多意图混杂 | "对比一下RAG和微调还有提示工程" | 需要拆解 |
| 指代不明 | "上次说的那个方案" | 缺少会话历史 |
| 太抽象/太具体 | "AI 的发展" | 范围太宽/窄 |

## 2. HyDE（Hypothetical Document Embeddings）

HyDE 的核心思想：**先让 LLM 生成假想答案，用假想答案的 Embedding 去检索**。

### 为什么有效
- 文档是"答案"，查询是"问题"，两者在 Embedding 空间中分布不同
- 假想答案的 Embedding 更接近真实文档的 Embedding
- 即使假想答案包含错误信息，其 Embedding 仍然指向正确方向

### 实现流程
```
用户查询: "RAG 如何解决 LLM 幻觉问题"

Step 1 - 生成假想答案:
LLM 输出: "RAG 通过从外部知识库检索相关文档，将检索到的真实信息
作为上下文注入 LLM 的提示词中，使 LLM 基于这些可靠信息生成答案，
从而减少幻觉..."

Step 2 - 用假想答案 Embedding 检索向量库
Step 3 - 用检索到的真实文档生成最终答案
```

### 代码实现
```python
def hyde_retrieval(query, llm, embedding_model, vector_store):
    # 1. 生成假想文档
    hyde_prompt = f"请根据以下问题，写一段简短的回答：{query}"
    hypothetical_doc = llm.generate(hyde_prompt)

    # 2. 用假想文档的 Embedding 检索
    hyde_embedding = embedding_model.encode(hypothetical_doc)
    results = vector_store.search(hyde_embedding, top_k=10)

    return results
```

> **注意**：HyDE 增加了一次 LLM 调用，延迟约 200-500ms。适合复杂/抽象查询，简单关键词查询不建议使用。

## 3. Step-Back Prompting（后退提示）

将具体问题提升到更高概念层面后再检索：

```
原始查询: "2024年奥运主办国的GDP增长率是多少？"
后退查询: "法国 GDP 增长率 2024"  // 先确定"2024奥运主办国=法国"

原始查询: "GPT-4 的 attention 层数和参数量的关系是什么？"
后退查询: "GPT-4 模型架构参数"  // 先检索模型基本信息
```

## 4. 查询分解（Query Decomposition）

将复杂问题拆解为子问题，分别检索后再综合答案。

### 复杂查询场景
```
用户: "对比 RAG 和微调在成本、效果、维护三方面的优劣"

拆解为:
├─ 子查询1: RAG 的部署成本是多少
├─ 子查询2: 微调大模型的成本是多少
├─ 子查询3: RAG 在问答任务上的效果评估
├─ 子查询4: 微调大模型在问答任务上的效果
├─ 子查询5: RAG 知识库的维护成本
└─ 子查询6: 微调模型的更新维护流程
```

### 实现框架
```python
def decompose_and_retrieve(query, llm, retriever):
    # 1. 拆解查询
    decompose_prompt = f"""
    将以下复杂问题拆解为 2-4 个独立的简单子问题，
    每个子问题一行：
    问题：{query}
    """
    sub_queries = llm.generate(decompose_prompt).split('\n')

    # 2. 并行检索
    all_docs = []
    for sq in sub_queries:
        docs = retriever.search(sq, top_k=5)
        all_docs.extend(docs)

    # 3. 去重 + 重排序
    unique_docs = deduplicate(all_docs)
    return rerank(unique_docs, query)
```

## 5. 查询路由（Query Routing）

根据查询意图分发到不同的检索源：

```python
def route_query(query, router_llm):
    intent = router_llm.classify(query, categories=[
        "产品规格查询",    # → 结构化数据库
        "技术文档查询",    # → 向量知识库
        "实时信息查询",    # → Web Search API
        "统计报表查询",    # → SQL 数据库
        "闲聊对话"         # → 直接回复
    ])

    retrievers = {
        "技术文档查询": vector_retriever,
        "产品规格查询": sql_retriever,
        "实时信息查询": web_search_retriever,
    }

    return retrievers[intent].search(query)
```

## 6. 查询优化策略选择

| 查询特征 | 推荐策略 | 额外延迟 |
|---------|---------|---------|
| 简短口语化 | 查询改写 | ~200ms |
| 抽象概念化 | HyDE | ~400ms |
| 复杂多跳 | 查询分解 | ~600ms |
| 混合意图 | 查询路由 | ~200ms |
| 简单关键词 | 直接检索 | 0ms |

> **生产建议**：不是所有查询都需要优化。使用轻量级分类器判断查询复杂度，简单查询直接检索，复杂查询才触发优化流程。
