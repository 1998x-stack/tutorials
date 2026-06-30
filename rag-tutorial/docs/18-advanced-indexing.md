# 高级索引架构

## 1. 超越扁平索引

基础 RAG 使用扁平索引（所有 Chunk 在一个向量空间中）。高级架构使用多层级、多类型的索引方案。

## 2. 层级索引（Hierarchical Index）

### 2.1 两级索引架构

```
第一级：摘要索引（Summary Index）
├─ 每个文档生成 1-3 句摘要
├─ Embedding → 存入摘要向量库
└─ 先在这里快速找到相关文档

第二级：细粒度索引（Chunk Index）
├─ 检索限定在第一步筛选出的文档中
└─ 精确定位相关段落
```

### 2.2 检索流程
```
查询 → 摘要索引（粗筛 3-5 个文档）→ 细粒度索引（在这些文档中精搜）
```

### 2.3 KohakuRAG 四层索引

```
Document（文档级）
  └── Section（章节级）
       └── Paragraph（段落级）
            └── Sentence（句子级）

Embedding 聚合：子节点 Embedding 加权平均 → 父节点 Embedding
检索：从粗到细，逐层下钻
```

**效果**：2025 年 WattBot 挑战赛获胜方案

## 3. 自修正索引（Self-Correcting Index）

### 3.1 核心思想
索引不应该是一次性构建的静态结构。当检索失败时，系统应该从失败中学习。

### 3.2 斯坦福 2025 方案
```
检索失败分析
    ↓
自动重新分块（在失败边界处重新切分）
    ↓
更新索引
    ↓
验证改进效果
```

**效果**：生产环境中噪声减少 25%

## 4. 混合索引类型

一个完整的知识库可能需要多种索引并存：

```python
class HybridIndexManager:
    def __init__(self):
        self.indexes = {
            "vector": VectorStore(),       # 语义搜索
            "sparse": BM25Index(),         # 关键词搜索
            "graph": Neo4jGraph(),         # 关系查询
            "summary": SummaryIndex(),     # 层级索引顶层
            "structured": SQLDatabase(),   # 结构化数据
        }

    def route_and_search(self, query, intent):
        results = {}
        if intent == "factual":
            results["sparse"] = self.indexes["sparse"].search(query)
            results["vector"] = self.indexes["vector"].search(query)
        elif intent == "relational":
            results["graph"] = self.indexes["graph"].traverse(query)
        elif intent == "overview":
            results["summary"] = self.indexes["summary"].search(query)

        return fuse(results)
```

## 5. 增量更新与实时索引

### 5.1 更新策略对比

| 策略 | 做法 | 适用场景 |
|------|------|---------|
| 全量重建 | 定期完整重建索引 | 数据量小、更新频率低 |
| 增量更新 | 只处理新增/修改的文档 | 数据持续增长 |
| 实时流式 | 数据变化立即反映到索引 | 信息时效性要求高 |

### 5.2 增量更新实现

```python
class IncrementalIndex:
    def add_documents(self, new_docs):
        for doc in new_docs:
            chunks = chunker.split(doc)
            embeddings = embedder.encode(chunks)
            self.vector_store.add(embeddings, chunks)

    def update_document(self, doc_id, new_content):
        # 1. 删除旧版本
        self.vector_store.delete(filter={"doc_id": doc_id})
        # 2. 索引新版本
        self.add_documents([new_content])

    def delete_document(self, doc_id):
        self.vector_store.delete(filter={"doc_id": doc_id})
```

### 5.3 Delta Encoding（差分编码）
```
版本 v1: 文档全文（1000 tokens）
版本 v2: 仅存储 diff（+50 tokens，-10 tokens）
```
适用于频繁小幅更新的文档。

## 6. 多租户索引设计

企业环境中，不同部门/用户的知识库需要隔离：

```python
# 方案一：物理隔离（独立 Collection）
collections = {
    "engineering": chroma_client.get_collection("eng_docs"),
    "marketing": chroma_client.get_collection("mkt_docs"),
    "legal": chroma_client.get_collection("legal_docs"),
}

# 方案二：逻辑隔离（同一 Collection，元数据过滤）
collection.query(
    query_texts=[query],
    where={"department": user_department, "access_level": {"$lte": user_level}}
)
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| 物理隔离 | 安全性强、性能独立 | 管理复杂、无法跨部门搜索 |
| 逻辑隔离 | 灵活、支持跨部门 | 过滤有开销、权限漏洞风险 |

## 7. 索引设计的决策树

```
数据规模？
├─ < 10K 文档 → 扁平向量索引（最简单）
├─ 10K-100K → 扁平 + 元数据过滤
├─ 100K-1M → 层级索引 + 混合索引
└─ > 1M → 分布式索引 + DiskANN

更新频率？
├─ 周/月 → 全量重建
├─ 天 → 增量更新
└─ 实时 → 流式索引 + Kafka

数据类型？
├─ 纯文本 → 向量索引
├─ 文本 + 表格 → 向量 + SQL
├─ 文本 + 关系 → 向量 + 图索引
└─ 多模态 → 多库并存
```
