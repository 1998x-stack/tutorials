# 稠密检索与语义搜索

## 1. 稠密检索原理

稠密检索（Dense Retrieval）使用神经网络将查询和文档分别编码为稠密向量，通过向量空间中的距离度量来判断相关性。

与稀疏检索的本质区别：
- 稀疏：词级匹配，维度=词汇表大小（数十万维，大部分为 0）
- 稠密：语义级匹配，维度=模型隐层维度（768-4096 维，全部有值）

## 2. DPR（Dense Passage Retrieval）

DPR 是稠密检索的经典架构，使用双编码器（Bi-Encoder）：

```
查询编码器（BERT_Q）
   ↓
查询向量 q  ──→ 余弦相似度 ←── 文档向量 d1, d2, ..., dN
                                ↑
                          文档编码器（BERT_D）
```

### DPR 训练
- 正例：能回答问题的相关文档
- 负例：BM25 检索到的非相关文档（Hard Negatives 效果更好）
- 损失函数：NLL（Negative Log-Likelihood）
- In-Batch Negatives：同批次中其他查询的文档作为负例

## 3. 相似度度量

### 3.1 余弦相似度（最常用）
```python
cosine_sim(q, d) = dot(q, d) / (||q|| × ||d||)
```
范围 [-1, 1]，归一化后等价于点积。

### 3.2 点积
```python
dot_product(q, d) = Σ q[i] × d[i]
```
计算最快，但受向量模长影响。

### 3.3 欧氏距离
```python
euclidean(q, d) = sqrt(Σ (q[i] - d[i])²)
```
转换为相似度：`sim = 1 / (1 + distance)`

## 4. 稠密检索的实现

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('BAAI/bge-large-zh-v1.5')

# 索引阶段
documents = [
    "RAG是检索增强生成技术的缩写",
    "向量数据库用于存储和搜索Embedding",
    "BM25是一种传统的文本检索算法"
]
doc_embeddings = model.encode(documents, normalize_embeddings=True)

# 检索阶段
query = "什么是RAG"
query_embedding = model.encode(query, normalize_embeddings=True)

# 计算余弦相似度（归一化后等价于点积）
scores = np.dot(query_embedding, doc_embeddings.T)
top_k_indices = np.argsort(scores)[-3:][::-1]

for idx in top_k_indices:
    print(f"Score: {scores[idx]:.3f} | {documents[idx]}")
```

## 5. 稠密检索的关键挑战

### 5.1 语义漂移（Semantic Drift）
向量相似不等于答案相关。两段话可能语义相似但互相矛盾，或者语义相似但都答非所问。

**缓解方法**：
- 使用 Cross-Encoder 重排序
- 添加元数据过滤
- 结合 BM25 精确匹配

### 5.2 查询-文档不对称
用户查询（短、自然语言）和索引文档（长、正式文体）的 Embedding 空间不重合。

**缓解方法**：
- HyDE：先让 LLM 生成"假想答案"，再用假想答案去检索
- 查询改写：将简短查询改写为完整表述
- 使用适应检索的 Embedding 模型（如 `bge` 系列，Query 和 Passage 使用不同指令前缀）

### 5.3 BGE 的特殊用法
```python
# BGE 系列的 Query 和 Passage 使用不同前缀
query_embedding = model.encode(
    "为这个句子生成表示以用于检索相关文章：" + query
)
passage_embedding = model.encode(passage)  # 不需要前缀
```

## 6. ANN 搜索优化

在大规模场景中（百万级以上），需要 ANN 索引加速：

| 库 | 特点 |
|----|------|
| FAISS | Meta 出品，GPU 加速 |
| Annoy | Spotify，内存映射 |
| NMSLIB | 高效 HNSW 实现 |
| cuVS | NVIDIA GPU 加速 |

## 7. 稠密检索的局限

- 不擅长精确匹配（如产品编号"SKU-12345"）
- 对数字和日期不敏感
- Embedding 模型有领域偏见
- 无法解决多跳推理问题
