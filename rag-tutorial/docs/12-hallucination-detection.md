# 幻觉检测与事实核查

## 1. RAG 为什么还会幻觉

即使有检索增强，RAG 系统仍然可能产生幻觉：

| 幻觉类型 | 原因 | 示例 |
|---------|------|------|
| **上下文违背** | LLM 忽略了检索到的正确信息 | 资料说"A>B"，回答却说"B>A" |
| **过度推理** | LLM 在检索信息基础上过度推断 | 资料说"产品降价10%"，回答推测"公司财务状况不佳" |
| **信息拼接** | LLM 将不同来源的信息错误关联 | 从文档A取X，从文档B取Y，声称"X 导致 Y" |
| **资料不足时的补充** | 检索到的信息不够，LLM 用自身知识填补 | 资料没有"价格"，LLM 用训练数据中的价格 |
| **检索失败连锁** | 检索到错误信息→基于错误信息生成→错误放大 | 检索到的文档本身包含过时/错误信息 |

## 2. 幻觉检测方法

### 2.1 基于 NLI 的事实一致性检查

使用自然语言推理（NLI）模型判断生成答案是否与检索上下文一致：

```python
from transformers import pipeline

nli = pipeline("text-classification", model="MoritzLaurer/DeBERTa-v3-large-mnli")

def check_consistency(context, claim):
    """
    检查 claim 是否可以从 context 推导出来
    """
    result = nli(f"{context} </s></s> {claim}")
    # 返回: entailment(蕴含) / contradiction(矛盾) / neutral(中性)
    return result
```

### 2.2 分段事实校验

将生成的答案拆分为原子声明，逐一验证：

```python
def extract_claims(answer, llm):
    """将答案分解为独立的声明列表"""
    prompt = f"""
    将以下回答拆解为独立的原子事实声明，每行一个：

    回答：{answer}

    声明列表：
    """
    return llm.generate(prompt).strip().split('\n')

def verify_claims(claims, context_docs):
    """逐一校验每个声明"""
    results = []
    for claim in claims:
        supported = False
        for doc in context_docs:
            if nli_check(doc, claim) == "entailment":
                supported = True
                break
        results.append({
            "claim": claim,
            "supported": supported,
            "verdict": "✅" if supported else "❌ 无据可查"
        })
    return results

# 示例输出:
# [✅] iPhone 15 主摄像头为 4800 万像素
# [✅] 支持 2 倍光学变焦
# [❌ 无据可查] iPhone 15 将于明年降价 20%
```

### 2.3 SelfCheckGPT

让 LLM 多次生成对同一问题的回答，检测一致性：
- 如果多次生成的回答高度一致 → 可能是基于训练知识（非幻觉）
- 如果某次回答与其他回答显著不同 → 可能是幻觉

## 3. 事实核查工具链

### 主流框架对比

| 工具 | 方式 | 适用阶段 |
|------|------|---------|
| DeepEval | 评估框架，内置幻觉检测指标 | 离线评估 |
| Guardrails AI | 实时护栏，在生成时拦截 | 在线推理 |
| LangSmith | 链路追踪 + 人工标注反馈 | 持续监控 |
| RAGAS | 端到端 RAG 评估 | 离线+在线 |

### DeepEval 示例

```python
from deepeval import evaluate
from deepeval.metrics import FaithfulnessMetric
from deepeval.test_case import LLMTestCase

test_case = LLMTestCase(
    input="iPhone 15 的摄像头参数？",
    actual_output="iPhone 15 主摄像头为 4800 万像素，支持 2 倍光学变焦。",
    retrieval_context=[
        "iPhone 15 主摄搭载 4800 万像素传感器，ƒ/1.6 光圈",
        "超广角摄像头为 1200 万像素",
    ]
)

metric = FaithfulnessMetric(threshold=0.7)
metric.measure(test_case)
print(f"忠实度分数: {metric.score}")
print(f"原因: {metric.reason}")
```

## 4. 幻觉检测的流程设计

```
LLM 生成答案
      │
      ▼
┌─────────────┐
│ 声明提取     │  (拆解为原子事实)
└──────┬──────┘
       ▼
┌─────────────┐
│ 逐一校验     │  (NLI 模型逐一比对)
└──────┬──────┘
       ▼
  ┌────┴────┐
  ▼         ▼
全部通过   发现无据声明
  │         │
  ▼         ▼
输出答案   标记⚠️ → 决定：重新生成 / 降级回复 / 告知用户
```

## 5. 幻觉率降低策略

| 策略 | 效果 | 代价 |
|------|------|------|
| 低 temperature (0.0-0.1) | 减少编造，但不保证 | 回答变机械 |
| 增强 Prompt 约束 | 有效减少上下文违背 | 可能降低流畅度 |
| 检索质量提升 | 从源头减少错误信息 | 工程投入大 |
| 事实校验 + 重新生成 | 可以过滤大部分幻觉 | 增加延迟 ~500ms |
| 多模型交叉验证 | 鲁棒 | 成本翻倍 |
