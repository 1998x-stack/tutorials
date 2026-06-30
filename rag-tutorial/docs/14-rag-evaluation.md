# RAG 评估体系

## 1. 评估的三维度

RAG 系统的评估需要从三个维度展开：

| 维度 | 评估什么 | 核心指标 |
|------|---------|---------|
| **检索质量** | 检索到的文档是否相关 | Recall@K, MRR, nDCG, Precision@K |
| **生成质量** | 生成的答案是否准确、忠实 | Faithfulness, Answer Relevance |
| **系统质量** | 端到端表现和用户体验 | Latency, TTFT, 用户满意度 |

## 2. RAGAS 框架

RAGAS（RAG Assessment）是 2025-2026 年最主流的 RAG 评估框架：

### 核心指标

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision,
)
from datasets import Dataset

# 准备评估数据
eval_dataset = Dataset.from_dict({
    "question": ["iPhone 15 的摄像头参数？"],
    "answer": ["iPhone 15 主摄像头为 4800 万像素..."],
    "contexts": [["iPhone 15 主摄 4800 万像素 f/1.6..."]],
    "ground_truth": ["iPhone 15 主摄4800万像素，超广角1200万像素"]
})

# 评估
result = evaluate(
    eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
)
print(result)
```

### 指标详解

| 指标 | 含义 | 计算方式 | 目标 |
|------|------|---------|------|
| **Faithfulness** | 答案是否忠于上下文 | 提取声明 → 验证是否被上下文支持 | > 0.8 |
| **Answer Relevancy** | 答案是否直接回答问题 | 反向生成问题 → 与原问题比较相似度 | > 0.7 |
| **Context Recall** | 检索是否找到了 Ground Truth 中的信息 | 检查 Ground Truth 各句是否在上下文中 | > 0.8 |
| **Context Precision** | 检索结果中相关文档的比例 | 相关文档数 / 总检索文档数 | > 0.7 |

## 3. 检索评估指标

### Recall@K
```
Recall@K = 相关且被检索到的文档数 / 总相关文档数
```
- 衡量"是否遗漏了重要信息"
- K 通常取 5, 10, 20
- 生产环境 Recall@10 应 > 0.85

### MRR（Mean Reciprocal Rank）
```
第一个相关文档的排名的倒数的平均值
MRR = (1/n) × Σ 1/rank_i
```
- 衡量"第一个正确答案出现的位置"
- rank=1 → 贡献 1.0；rank=10 → 贡献 0.1

### nDCG（Normalized Discounted Cumulative Gain）
- 考虑排序位置和相关性等级（非二元）
- 排名越靠前权重越大

## 4. Golden Dataset 构建

### 构建流程
```
1. 收集真实用户查询
2. 人工标注相关文档
3. 人工编写标准答案（Ground Truth）
4. 分测试集/验证集
5. 每次变更后自动跑评估
```

### Golden Dataset 规模建议
- 最小可用：50 条（快速迭代）
- 生产推荐：200-500 条（覆盖各种查询类型）
- 成熟产品：1000+ 条（涵盖边缘案例）

## 5. 评估流水线

```yaml
# CI/CD 中的评估流程
评估步骤:
  1. 代码变更触发评估
  2. 在 Golden Dataset 上运行 RAG 系统
  3. 计算 RAGAS 指标
  4. 与 baseline 对比
  5. 指标退化 → 告警/阻止合并
  6. 指标提升 → 记录为新的 baseline
```

## 6. 在线评估与反馈

### 用户反馈信号
- 👍/👎 按钮
- 答案是否被复制（被复制 = 有用）
- 用户是否追问（追问 = 答案不够好）
- 会话时长和交互轮次

### 自动标注反馈
```python
def auto_label_feedback(query, answer, context):
    """自动判断回答质量"""
    faithfulness_score = check_faithfulness(answer, context)
    if faithfulness_score < 0.6:
        return "negative", "疑似幻觉"
    if len(answer) < 20 and "无法" in answer:
        return "negative", "答案不充分"
    return "positive", "ok"
```

## 7. 评估结果解读

| 场景 | Faithfulness | Context Recall | 诊断 |
|------|-------------|---------------|------|
| | 高 | 高 | ✅ 检索好，生成好 |
| | 高 | 低 | ⚠️ 检索未召回，但 LLM 自身知识补上了（危险！）|
| | 低 | 高 | ❌ 检索到了但 LLM 没用好 |
| | 低 | 低 | ❌ 检索失败且生成失败 |
