# 多模态 RAG

## 1. 多模态 RAG 的场景

传统 RAG 仅处理文本，但现实世界的信息是多模态的：

| 模态 | 示例场景 | 检索需求 |
|------|---------|---------|
| 图片 | "展示产品外观设计" | 基于图片内容或文字描述检索 |
| 表格 | "Q3 各部门的预算对比" | 结构化数据检索 |
| 图表 | "近三年营收趋势图" | 图表类型+数据匹配 |
| 公式 | "牛顿第二定律的数学表达式" | LaTeX/公式检索 |
| 音频 | "上次会议中提到的 deadline" | 语音转文本后检索 |

## 2. 多模态 Embedding

### 2.1 CLIP 模型
CLIP 将文本和图像映射到同一向量空间：
- 文本"一只狗"和狗的图片在空间中距离很近
- 支持图文跨模态检索

```python
from transformers import CLIPModel, CLIPProcessor
import torch

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

# 图片 Embedding
image = load_image("product.jpg")
image_inputs = processor(images=image, return_tensors="pt")
image_embedding = model.get_image_features(**image_inputs)

# 文本 Embedding
text_inputs = processor(text=["一款白色智能手机"], return_tensors="pt", padding=True)
text_embedding = model.get_text_features(**text_inputs)

# 跨模态相似度
similarity = torch.cosine_similarity(image_embedding, text_embedding)
```

### 2.2 多模态向量模型（2026）

| 模型 | 模态 | 特点 |
|------|------|------|
| Llama-Nemotron-Embed-VL | 文本+图像 | NVIDIA，支持图文联合嵌入 |
| Jina CLIP v2 | 文本+图像 | 支持中英文 |
| BridgeTower | 文本+图像+音频 | 三模态联合 |

## 3. 表格检索

### 3.1 策略一：表格转文本
```python
# 将表格转为结构化文本用于检索
def table_to_text(table):
    rows = []
    for row in table:
        rows.append(" | ".join(str(cell) for cell in row))
    return "\n".join(rows)

# 表格 Embedding = 文本 Embedding
table_embedding = embedding_model.encode(table_to_text(table))
```

### 3.2 策略二：Text-to-SQL
```python
def text_to_sql_retrieval(query, db_schema, llm):
    """将自然语言查询转为 SQL"""
    prompt = f"""
    数据库 schema:
    {db_schema}

    将以下问题转为 SQL 查询：
    {query}
    """
    sql = llm.generate(prompt)
    result = execute_sql(sql)
    return result
```

### 3.3 策略三：混合检索
```
表格查询: "Q3 销售额最高的产品"
    ├─ 语义检索：找"销售额""产品排名"相关的文档段落
    └─ SQL 查询：SELECT product, SUM(revenue) FROM sales WHERE quarter='Q3'
         ↓
    结果融合：文档段落 + 表格数据 → LLM 生成综合答案
```

## 4. 多模态 RAG 架构

```python
class MultimodalRAG:
    def __init__(self):
        self.text_index = TextVectorStore()
        self.image_index = ImageVectorStore()
        self.table_db = SQLDatabase()

    def search(self, query, query_type="auto"):
        results = {}

        # 文本检索
        results["text"] = self.text_index.search(query)

        # 图片检索
        results["images"] = self.image_index.search(query)

        # 结构化查询
        if self._needs_sql(query):
            results["tables"] = self.table_db.query(query)

        return self._fuse_results(results, query)
```

## 5. 多模态生成

将不同模态的检索结果整合到 LLM 上下文中：

### VLM（Vision-Language Model）方案

```python
# 支持图片理解的生成
response = vlm_client.chat(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "基于提供的图片和文档回答问题"},
        {"role": "user", "content": [
            {"type": "text", "text": f"文档：{text_context}\n问题：{query}"},
            {"type": "image_url", "image_url": {"url": retrieved_image_url}},
        ]}
    ]
)
```

### 表格辅助生成
```
上下文：
[文本] 公司Q3财报显示营收增长15%
[表格] 各部门Q3营收：
        | 部门 | Q3营收 | 增长率 |
        | A    | 1200万 | 18%    |
        | B    | 800万  | 12%    |
        | C    | 500万  | 8%     |

LLM 生成：
"公司Q3整体营收增长15%。其中A部门表现最强劲，贡献1200万营收，增长18%；
B部门增长12%至800万；C部门增速相对较慢，仅增长8%至500万。"
```

## 6. 多模态 RAG 的挑战

| 挑战 | 说明 | 当前方案 |
|------|------|---------|
| 对齐问题 | 不同模态的向量空间不一致 | CLIP 类联合训练模型 |
| 存储成本 | 图片/音频向量远大于文本 | 压缩和分层存储 |
| 检索融合 | 不同模态的检索结果如何统一排序 | RRF 或加权融合 |
| 生成整合 | LLM 可能不擅长解读图表 | 使用 VLM 或先转文字 |
