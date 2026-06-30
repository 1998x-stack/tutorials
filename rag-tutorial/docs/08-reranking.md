# 重排序技术详解

## 1. 为什么需要重排序

检索阶段使用 Bi-Encoder（双编码器）追求速度，但精度有限。重排序（Reranking）使用更强的 Cross-Encoder 对候选结果进行精细评分。

```
检索阶段（快速、低精度）         重排序阶段（较慢、高精度）
┌─────────────────────┐      ┌──────────────────────┐
│  Top-50 候选文档     │ ───→ │  Cross-Encoder       │
│  (Bi-Encoder 粗筛)   │      │  逐对精细打分         │
└─────────────────────┘      └────────┬─────────────┘
                                      ▼
                               Top-5 精选文档
                               (送入 LLM 生成)
```

## 2. Cross-Encoder vs Bi-Encoder

### Bi-Encoder（编码器，用于检索）
```
Query → BERT → q_vector     (提前计算)
Doc   → BERT → d_vector     (提前计算，存入索引)
Score = cosine(q_vector, d_vector)   ← 快速但粗糙
```

### Cross-Encoder（交叉编码器，用于重排序）
```
[CLS] Query [SEP] Document [SEP] → BERT → Score
         ↑                          ↑
    拼接后一起喂给模型            全注意力交互
```
Query 和 Document 的每个 token 都相互注意，精度远高于 Bi-Encoder。

## 3. 主流重排序模型

| 模型 | 类型 | 语言 | 最大长度 | 延迟 |
|------|------|------|---------|------|
| BGE-Reranker-v2-m3 | 开源 | 多语言 | 8192 | ~50ms |
| BGE-Reranker-v2-gemma | 开源 | 多语言 | 8192 | ~80ms |
| Cohere Rerank v3 | API | 多语言 | 4096 | ~50ms |
| Voyage Rerank-2 | API | 英文 | 16000 | ~60ms |
| Jina Reranker v2 | 开源 | 多语言 | 8192 | ~40ms |
| Llama-Nemotron-Rerank | NVIDIA | 英文 | 8192 | ~45ms |

## 4. 两阶段检索架构

### 阶段一：粗筛（快速）
```python
# 使用 Bi-Encoder 从百万级文档中筛选 Top-25
candidate_ids = vector_store.search(
    query_embedding,
    top_k=25  # 取较多的候选
)
```

### 阶段二：精排（精确）
```python
from FlagEmbedding import FlagReranker

reranker = FlagReranker('BAAI/bge-reranker-v2-m3')

pairs = [[query, documents[i]] for i in candidate_ids]
scores = reranker.compute_score(pairs)

# 按重排序分数重新排序
reranked = sorted(
    zip(candidate_ids, scores),
    key=lambda x: x[1], reverse=True
)[:5]  # 取 Top-5 送入 LLM
```

## 5. 重排序的性能权衡

### 延迟分析
```
Bi-Encoder 检索（Top-50）: ~10ms
Cross-Encoder 重排序（50对）: ~150ms
LLM 生成: ~2000ms
─────────────────────────────
总延迟: ~2160ms（重排序占 7%）
```

### 精度提升
- 仅 Bi-Encoder：MRR@10 ≈ 0.70
- Bi-Encoder + Cross-Encoder Rerank：MRR@10 ≈ 0.85
- **MRR 绝对提升 15 个百分点**

### 成本考量
- 本地部署：需要 GPU 推理（约 1-2GB 显存）
- API 调用：Cohere/OpenAI 按 token 计费
- 延迟预算：100-200ms 是合理的重排序延迟

## 6. 重排序最佳实践

1. **粗筛数量**：Top-25 到 Top-50 是合理范围。太少→遗漏相关文档；太多→重排序成本高
2. **精排数量**：最终取 Top-3 到 Top-5 送入 LLM
3. **模型选择**：
   - 自部署（数据隐私）→ BGE-Reranker-v2-m3
   - 免运维（追求便利）→ Cohere Rerank API
   - 长文档 → Voyage Rerank-2（支持 16K tokens）
4. **批处理**：多个 Query-Doc 对可以批量推理，降低延迟
5. **缓存**：高频查询的重排序结果可以缓存
