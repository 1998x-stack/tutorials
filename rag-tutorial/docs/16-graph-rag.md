# GraphRAG 与知识图谱增强

## 1. 向量检索的盲区

向量检索擅长找"相似的片段"，但无法回答需要**全局理解**和**多跳推理**的问题：

| 查询类型 | 向量检索 | GraphRAG |
|---------|---------|----------|
| "什么是RAG" | ✅ 语义匹配 | ✅ 定义查询 |
| "文档集涉及哪些主题" | ❌ 无法汇总全局信息 | ✅ 社区摘要 |
| "A 和 B 之间有什么关系" | ❌ 需要多跳推理 | ✅ 图遍历 |
| "哪些方法被引用了最多" | ❌ 需要聚合统计 | ✅ 中心性分析 |

## 2. 知识图谱构建

### 2.1 实体与关系抽取

```python
def extract_entities_relations(text, llm):
    prompt = f"""
    从以下文本中提取实体和关系，以 JSON 格式返回：

    文本：{text}

    输出格式：
    {{
        "entities": [
            {{"name": "GPT-4", "type": "MODEL"}},
            {{"name": "OpenAI", "type": "ORGANIZATION"}}
        ],
        "relations": [
            {{"source": "OpenAI", "target": "GPT-4", "relation": "DEVELOPED"}}
        ]
    }}
    """
    return llm.generate_json(prompt)
```

### 2.2 图存储（Neo4j）

```cypher
// 创建实体节点
CREATE (gpt4:Model {name: 'GPT-4', parameters: '1.76T'})
CREATE (openai:Organization {name: 'OpenAI'})
CREATE (transformer:Architecture {name: 'Transformer'})

// 创建关系
CREATE (openai)-[:DEVELOPED]->(gpt4)
CREATE (gpt4)-[:USES]->(transformer)

// 多跳查询：哪些组织开发了基于 Transformer 的模型？
MATCH (org:Organization)-[:DEVELOPED]->(m:Model)-[:USES]->(a:Architecture {name: 'Transformer'})
RETURN org.name, m.name
```

## 3. Microsoft GraphRAG

### 3.1 核心思想

GraphRAG 将文档自动构建为**分层社区图**：

```
原始文档集合
    ↓ 实体/关系抽取
知识图谱（实体 + 关系）
    ↓ 社区检测（Leiden 算法）
分层社区结构
    ↓ LLM 生成社区摘要
社区级摘要索引 ← 用于回答全局问题
```

### 3.2 两级检索

```
查询 → 向量检索（局部细节） + 社区摘要检索（全局背景）
                    ↓
            综合生成答案
```

### 3.3 适用场景

- **全局摘要查询**："这个数据集的主要主题是什么？"
- **趋势分析**："过去3年该领域的研究方向有何变化？"
- **关系发现**："哪些技术之间存在协同关系？"
- **异常检测**："是否有与主流观点矛盾的研究？"

## 4. GraphRAG 与向量 RAG 的关系

```
             用户查询
                │
         ┌──────┴──────┐
         ▼              ▼
    查询路由器（判断查询类型）
         │              │
    ┌────┴────┐   ┌────┴────┐
    ▼         ▼   ▼         ▼
  局部查询  全局查询  局部查询  全局查询
    │         │     │         │
    ▼         ▼     ▼         ▼
 向量检索  图遍历  向量RAG  GraphRAG
    │         │     │         │
    └────┬────┘     └────┬────┘
         └──────┬────────┘
                ▼
          综合生成答案
```

> GraphRAG 和向量 RAG 不是互斥的，而是互补的。2026 年生产级 RAG 系统通常同时部署两者，根据查询类型路由。

## 5. 图增强检索（Graph-Enhanced Retrieval）

### 单跳图增强
```
用户查询: "Transformer 的作者还写了哪些论文？"
Step 1: 向量检索找到 "Attention Is All You Need" 论文节点
Step 2: 图遍历找到作者 → 沿 WROTE 边 → 找到其他论文
Step 3: 返回这些论文的摘要
```

### 多跳图推理
```
用户查询: "与 GPT-4 使用相同架构的模型，在医疗领域有哪些应用？"
Step 1: 找到 GPT-4 → USES → Transformer
Step 2: Transformer ← USES ← 其他模型
Step 3: 过滤 MEDICAL 领域的应用
Step 4: 综合返回结果
```

## 6. TechGraphRAG（2026 最新）

13 步 Agentic GraphRAG 流水线：
1. 意图分类 + 证据充分性评分
2. 本地向量检索
3. 证据评分 → 不足则触发外部学术搜索
4. 知识图谱遍历（论文/作者/方法/引用关系）
5. 引用完整性验证
6. 生成 + 质量自检 + 自动重生成

**效果**：幻觉率降低 30%（Meta 2026 年报告）

## 7. GraphRAG 的成本与收益

| 维度 | 向量 RAG | GraphRAG |
|------|---------|----------|
| 索引构建 | 快（分钟-小时） | 慢（小时-天，需 LLM 抽取） |
| 索引成本 | 低（仅 Embedding） | 高（大量 LLM 调用） |
| 全局查询 | 差 | 优秀 |
| 局部查询 | 优秀 | 一般 |
| 多跳推理 | 差 | 优秀 |
| 运营复杂度 | 低 | 高 |
