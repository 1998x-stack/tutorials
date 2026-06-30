# 混合检索策略

## 1. 为什么需要混合检索

单一检索方式各有盲区：

| 场景 | BM25 | 稠密检索 |
|------|------|---------|
| "SKU-88492 规格参数" | ✅ 精确匹配 | ❌ 可能找不到 |
| "如何降低服务器功耗" | ❌ 同义词问题 | ✅ 语义理解 |
| "2024年Q3财务报表" | ✅ 数字日期 | ⚠️ 可能不准 |
| "跟上次差不多的方案" | ❌ 口语化查询 | ✅ 理解意图 |

**结论**：混合检索取长补短，是 2026 年 RAG 生产环境的 baseline。

## 2. 混合检索架构

```
             用户查询
                │
      ┌─────────┼─────────┐
      ▼                   ▼
  BM25 检索          稠密向量检索
  (倒排索引)         (ANN 向量库)
      │                   │
      ▼                   ▼
   Top-K₁ 结果        Top-K₂ 结果
      │                   │
      └─────────┬─────────┘
                ▼
           结果融合（RRF）
                │
                ▼
        融合后的 Top-K 结果
```

## 3. Reciprocal Rank Fusion（RRF）

RRF 是最常用的结果融合算法，无需调参：

```
RRF_score(d) = Σ 1 / (k + rank_i(d))

其中：
- rank_i(d)：文档 d 在第 i 个检索器中的排名
- k：平滑参数（通常 k=60）
```

### RRF 实现

```python
def reciprocal_rank_fusion(results_list, k=60):
    """
    results_list: list of lists, each sorted by relevance score
    """
    scores = {}
    for results in results_list:
        for rank, doc_id in enumerate(results, start=1):
            if doc_id not in scores:
                scores[doc_id] = 0
            scores[doc_id] += 1 / (k + rank)

    # 按融合分数排序
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### RRF 的优势
- 不需要各检索器的原始分数（不同检索器的分数不可比）
- 不需要对分数做归一化
- 简单、鲁棒、效果稳定

## 4. 其他融合策略

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| RRF | 排名倒数加权 | 无需归一化 | 忽略原始分数信息 |
| 加权分数融合 | Score = α×BM25 + (1-α)×Dense | 可调控 | 需要分数归一化 |
| 级联融合 | BM25 粗筛 → Dense 精排 | 效率高 | 可能遗漏 |
| Learned Fusion | 训练一个模型学习最佳融合权重 | 最优效果 | 需要标注数据 |

## 5. 混合检索实战

```python
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    def __init__(self, documents, embeddings, bm25, dense_weight=0.5):
        self.documents = documents
        self.embeddings = embeddings
        self.bm25 = bm25
        self.dense_weight = dense_weight

    def search(self, query, query_embedding, top_k=10):
        # BM25 检索
        bm25_scores = self.bm25.get_scores(query.split())
        bm25_ids = np.argsort(bm25_scores)[-top_k*2:][::-1]

        # 稠密检索
        dense_scores = np.dot(query_embedding, self.embeddings.T)
        dense_ids = np.argsort(dense_scores)[-top_k*2:][::-1]

        # RRF 融合
        rrf_scores = {}
        for rank, idx in enumerate(bm25_ids, 1):
            rrf_scores[idx] = rrf_scores.get(idx, 0) + 1/(60+rank)
        for rank, idx in enumerate(dense_ids, 1):
            rrf_scores[idx] = rrf_scores.get(idx, 0) + 1/(60+rank)

        # 排序返回
        sorted_ids = sorted(rrf_scores, key=rrf_scores.get, reverse=True)
        return [self.documents[i] for i in sorted_ids[:top_k]]
```

## 6. 多路召回

除了 BM25 + 稠密向量，还可以引入更多检索路径：

```
多路召回架构：
├─ 向量检索（语义匹配）
├─ BM25 检索（关键词匹配）
├─ 知识图谱检索（实体关系）
├─ SQL/结构化查询（数据库）
└─ API 实时调用（外部数据）
```

关键设计原则：
- 每路召回数量宜多不宜少（如每路取 Top-50）
- 融合时统一做重排序
- 不同查询可动态调整各路权重

## 7. 混合检索效果数据

Google 2025 年实验：
- 混合检索（BM25 + Dense + RRF）比纯稠密检索 MRR 提升 **15-20%**
- 在包含专有名词的查询上提升更显著（30%+）

生产环境推荐配置：
1. 同时运行 BM25 和稠密检索
2. RRF 融合（k=60）
3. 取 Top-25 送入 Cross-Encoder 重排序
4. 取 Top-5 送入 LLM 生成
