# 向量嵌入技术

## 1. Embedding 的数学直觉

Embedding 是将文本映射为高维空间中的点的过程。核心直觉：**语义相似的文本在向量空间中距离更近**。

- 输入：一段文本（词/句/段落）
- 输出：一个固定维度的浮点数向量（如 768 维、1024 维、4096 维）
- 相似度：通常用余弦相似度（Cosine Similarity）度量

## 2. 主流 Embedding 模型

### 2.1 闭源模型

| 模型 | 维度 | 最大输入 | 特点 |
|------|------|---------|------|
| OpenAI text-embedding-3-small | 512/1536 | 8191 tokens | 性价比高 |
| OpenAI text-embedding-3-large | 256/1024/3072 | 8191 tokens | 精度最高 |
| Cohere Embed v4 | 1024 | 8192 tokens | 多语言强 |
| Voyage-3 | 1024 | 32000 tokens | 长文本优势 |

### 2.2 开源模型

| 模型 | 维度 | 最大输入 | 特点 |
|------|------|---------|------|
| BGE-M3 | 1024 | 8192 | 多语言，支持稠密+稀疏 |
| BGE-Large-zh | 1024 | 512 | 中文最佳 |
| M3E-Large | 1024 | 512 | 中文社区首选 |
| Qwen3-Embedding-4B | 2560 | 32768 | 长文本，多任务 |
| Multilingual-E5-Large | 1024 | 512 | 多语言通用 |

## 3. 嵌入模型架构

### 3.1 Bi-Encoder（双编码器）
- Query 和 Document 分别编码，独立计算向量
- 适合大规模检索（可以预先索引所有文档）
- 检索速度最快

### 3.2 Cross-Encoder（交叉编码器）
- Query 和 Document 拼接后一起输入模型
- 全注意力交互，精度远高于 Bi-Encoder
- 速度慢，不适合大规模检索——用于重排序

### 3.3 ColBERT（多向量表示）
- 每个 Token 单独生成一个向量
- 检索时使用 MaxSim（最大相似度）匹配
- 精度接近 Cross-Encoder，速度接近 Bi-Encoder
- 存储成本：每个文档存储 N×d 个浮点数

## 4. 稀疏嵌入（Sparse Embedding）

除了稠密向量，还有稀疏表示方法：

| 方法 | 原理 | 适用场景 |
|------|------|---------|
| BM25 | 基于词频和逆文档频率的统计方法 | 精确关键词匹配 |
| SPLADE | 学习到的稀疏+稠密对齐表示 | 捕捉语义但保持稀疏性 |
| BGE-M3 稀疏模式 | BGE-M3 同时输出稠密和稀疏向量 | 混合检索 |

## 5. Embedding 微调

### 5.1 什么时候需要微调
- 领域术语与通用语料差异大（医疗、法律、工业）
- 通用模型的相似度判断不符合业务需求
- 多语言场景下某些语言效果差

### 5.2 微调方法
- **Triplet Loss**：锚点样本 + 正例 + 负例 三元组训练
- **Hard Negatives**：挖掘模型当前判断错误的负样本
- **对比学习**：SimCSE、Contriever 等方法

### 5.3 实际效果
在工业手册场景中，对 BGE-M3 进行领域微调：
- Recall@5：0.67 → 0.72（提升 7.5%）
- 使用 Hard Negative 挖掘后进一步提升至 0.78

## 6. Embedding 模型选择指南

```
是否需要中文？ 
├─ 是 → BGE-M3 / M3E / Qwen3-Embedding
│   ├─ 长文档（>8K）→ Qwen3-Embedding-4B
│   └─ 短文本 → BGE-M3
└─ 否 → 
    ├─ 追求精度 → text-embedding-3-large / Voyage-3
    ├─ 预算有限 → text-embedding-3-small
    └─ 自部署 → BGE-M3 / Multilingual-E5
```
