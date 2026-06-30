# 上下文增强与提示工程

## 1. 上下文窗口管理

RAG 系统的上下文窗口需要同时容纳：
- System Prompt（角色设定 + 指令）
- Retrieved Context（检索到的文档片段）
- User Query（用户问题）
- Conversation History（多轮对话历史）

上下文窗口容量规划：
```
总容量（如 128K tokens）
├─ System Prompt:  ~500 tokens（1%）
├─ 对话历史:       ~2000 tokens（2%）
├─ 检索上下文:     ~4000 tokens（3%）
├─ 用户查询:       ~200 tokens（0.2%）
└─ 预留给生成:     ~121K tokens（94%）
```

> 实际上，检索上下文通常占 10-30%，生成内容很少超 2000 tokens。关键是不要让噪声文档挤占窗口。

## 2. Lost-in-the-Middle 问题

LLM 对提示词中**中间位置**的信息关注度最低。

### 现象
```
提示词结构:       LLM 的注意力:
┌─────────┐      ████████████ 开始（高）
│ Doc 1   │
│ Doc 2   │      ██ 中间（低）
│ Doc 3   │
│ Doc 4   │      ████████████ 结束（高）
└─────────┘
```

### 解决方案
1. **重排序后置最相关文档在首尾**：最相关的放开头和结尾
2. **精简文档数量**：3-5 个高质量片段优于 10 个低质量片段
3. **文档摘要前置**：每个片段前加一句话摘要

```python
def arrange_context(docs, scores):
    """将最相关文档放在开头和结尾"""
    sorted_docs = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)

    if len(sorted_docs) <= 3:
        return [d for d, _ in sorted_docs]

    # 最相关 → 开头，次相关 → 结尾，其余 → 中间
    arranged = [sorted_docs[0], sorted_docs[-1]] + sorted_docs[1:-1]
    return [d for d, _ in arranged]
```

## 3. RAG 提示模板设计

### 3.1 基础模板
```
你是一个专业的问答助手。请严格基于以下参考资料回答用户的问题。

## 参考资料
{context}

## 要求
1. 只使用参考资料中的信息
2. 如果参考资料不足以回答问题，请明确说明"根据现有资料无法回答"
3. 在回答中标注信息来源编号

## 用户问题
{question}

## 回答
```

### 3.2 带引用的模板
```
你是一个需要提供准确来源引用的研究助手。

## 参考资料
[1] {doc1}
[2] {doc2}
[3] {doc3}

## 回答格式
- 每个事实声明后标注来源编号，如 [1]
- 如果信息来自多个来源，标注 [1][3]
- 不要编造参考资料中没有的信息

## 用户问题
{question}
```

### 3.3 多轮对话模板
```
## 对话历史
{history}

## 最新参考资料（可能与历史相关也可能无关）
{context}

## 当前问题
{question}
```

## 4. 上下文压缩

当检索到的文档太长时，需要压缩后再送入 LLM。

### 4.1 简单截断
```python
# 按 token 数截断，保留最相关的片段
def truncate_context(docs, max_tokens=4000):
    context = ""
    for doc in docs:
        if count_tokens(context + doc) > max_tokens:
            break
        context += doc + "\n\n"
    return context
```

### 4.2 LLMLingua 压缩
使用小模型对检索结果进行摘要压缩：
- 输入：原始检索文档（可能是几千 tokens）
- 输出：压缩后的关键信息（几百 tokens）
- 压缩比：3-10x
- 额外延迟：~200ms

### 4.3 选择性上下文
不是所有检索片段都有用，使用一个小模型判断每段是否真正相关：

```python
def filter_relevant(query, docs, filter_llm):
    relevant_docs = []
    for doc in docs:
        is_relevant = filter_llm.check(f"问题：{query}\n文档：{doc}\n是否相关？")
        if is_relevant:
            relevant_docs.append(doc)
    return relevant_docs
```

## 5. 上下文增强的进阶技巧

### 5.1 元数据注入
在每个文档片段前面添加元数据行：
```
[文档：RAG技术白皮书 | 章节：3.2 检索策略 | 日期：2025-06]
... 文档内容 ...
```

### 5.2 层级上下文
对于层级索引检索的结果，附加上下文结构：
```
[主章节] 第3章：RAG 检索策略
  [子章节 3.1] 稠密检索的原理...
  [子章节 3.2] 混合检索的实现... ← 你检索到的部分
  [子章节 3.3] 检索结果的评估...
```

### 5.3 对比性上下文
当查询涉及对比时，同时提供正反两面信息：
```
支持观点 A 的文档：
  [Doc1] ...
  [Doc2] ...

支持观点 B 的文档：
  [Doc3] ...
  [Doc4] ...
```
