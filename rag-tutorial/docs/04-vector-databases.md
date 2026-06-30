# 向量数据库入门

## 1. 为什么需要向量数据库

向量数据库专门为高维向量的存储和相似度搜索而设计。与传统的精确搜索不同，向量数据库使用**近似最近邻搜索（ANN）** 在大规模数据中快速找到最相似的向量。

核心需求驱动：
- LLM Embedding 维度通常 768-4096，传统 B-Tree 索引无效
- 知识库可能有百万级文档
- 用户期望毫秒级响应

## 2. ANN 近似最近邻搜索

### 2.1 HNSW（Hierarchical Navigable Small World）

多层级图结构，每层减少节点数：
- 最顶层：少量节点，长距离跳跃
- 最底层：所有节点，精确搜索
- 搜索复杂度：O(log N)

### 2.2 IVF（Inverted File Index）

先聚类再搜索：
- 将向量空间划分为 K 个聚类（Voronoi 区域）
- 搜索时先找到最近的 N 个聚类中心，再在聚类内搜索
- 参数：nlist（聚类数）、nprobe（搜索聚类数）

### 2.3 PQ（Product Quantization）

向量压缩技术：
- 将高维向量切分为多个子向量
- 每个子向量独立量化
- 大幅减少内存占用（如 4096 维→128 字节）

## 3. 主流向量数据库对比

| 数据库 | 类型 | 索引算法 | 分布式 | 适用场景 |
|--------|------|---------|--------|---------|
| Milvus | 专用向量库 | HNSW/IVF/DiskANN | ✅ | 大规模生产 |
| Qdrant | 专用向量库 | HNSW | ✅ | 中等规模 |
| Weaviate | 向量+对象存储 | HNSW | ✅ | 混合存储 |
| Pinecone | 云服务 | 自动选择 | ✅ | 免运维 |
| Chroma | 嵌入式 | HNSW | ❌ | 原型开发 |
| FAISS | 算法库 | HNSW/IVF/PQ | ❌ | 研究/自建 |
| Elasticsearch | 搜索引擎 | HNSW(8.x+) | ✅ | 已有ES基础设施 |
| pgvector | PostgreSQL扩展 | IVFFlat/HNSW | ✅ | 与业务数据共存 |

## 4. 向量库基本操作

```python
# Chroma 示例
import chromadb

client = chromadb.Client()
collection = client.create_collection(name="my_docs")

# 插入
collection.add(
    documents=["这是第一篇文档的内容", "这是第二篇文档的内容"],
    metadatas=[{"source": "doc1.pdf"}, {"source": "doc2.pdf"}],
    ids=["id1", "id2"]
)

# 查询
results = collection.query(
    query_texts=["第一篇文档"],
    n_results=5
)

# 删除
collection.delete(ids=["id1"])
```

## 5. 索引参数调优

### HNSW 关键参数
- **M**：每个节点的最大连接数（越大越精确，内存越多），典型值 16-64
- **efConstruction**：构建时的搜索深度（越大越精确，构建越慢），典型值 200-500
- **efSearch**：查询时的搜索深度（越大越精确，查询越慢），典型值 50-200

### 延迟 vs 精度的权衡

| efSearch | 召回率 | 延迟 |
|----------|--------|------|
| 16 | 85% | ~5ms |
| 64 | 95% | ~10ms |
| 128 | 98% | ~15ms |
| 256 | 99% | ~25ms |

## 6. 生产部署考量

1. **存储成本**：100 万条 1024 维向量 ≈ 4GB（原始）+ 索引开销（1-2x）
2. **内存规划**：HNSW 需要全量数据在内存中
3. **磁盘索引**：DiskANN 支持部分数据在磁盘，适合超大规模
4. **副本策略**：至少 2 副本保证高可用
5. **监控指标**：QPS、P99 延迟、索引大小、召回率

## 7. 过滤搜索

生产环境的关键需求——按元数据过滤后再搜索：

```python
results = collection.query(
    query_texts=["产品介绍"],
    n_results=5,
    where={"department": "marketing", "year": {"$gte": 2024}}
)
```

实现要点：
- 预过滤：先过滤再 ANN 搜索（准确但可能遗漏）
- 后过滤：先 ANN 搜索再过滤（可能返回不足 K 条）
- 混合过滤：结合两种策略
