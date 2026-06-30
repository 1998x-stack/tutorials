# RAG 框架对比：LangChain vs LlamaIndex

## 1. 两大框架定位

| 维度 | LangChain | LlamaIndex |
|------|-----------|------------|
| 核心理念 | 通用 LLM 应用框架 | 专为数据索引和检索设计 |
| 优势领域 | Agent编排、复杂工作流 | 数据摄入、索引构建 |
| 设计哲学 | 组件化、Chainable | Data-Centric、可组合 |
| 学习曲线 | 中等 | 中等偏低 |
| 生态 | 极广（工具/模型/向量库集成） | 专注索引和检索 |

## 2. 核心抽象对比

### 2.1 LangChain

```python
from langchain.chains import RetrievalQA
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.llms import ChatOpenAI

# 经典的 RAG Chain
vector_store = Chroma(embedding_function=OpenAIEmbeddings())
retriever = vector_store.as_retriever(search_kwargs={"k": 5})

qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(),
    chain_type="stuff",  # stuff / map_reduce / refine / map_rerank
    retriever=retriever,
    return_source_documents=True
)

result = qa_chain({"query": "什么是RAG？"})
```

LangChain 的 Chain 类型：
- **stuff**：把所有文档一次性塞入 Prompt（最简单）
- **map_reduce**：每篇文档独立生成摘要，然后合并
- **refine**：逐篇迭代，每篇文档在上一篇基础上修改答案
- **map_rerank**：每篇文档打分，取最高分的生成答案

### 2.2 LlamaIndex

```python
from llama_index import (
    VectorStoreIndex, SimpleDirectoryReader, ServiceContext
)

# 文档加载和索引
documents = SimpleDirectoryReader("./data").load_data()

# 构建索引（自动完成分块、Embedding、存储）
index = VectorStoreIndex.from_documents(documents)

# 查询引擎
query_engine = index.as_query_engine(
    similarity_top_k=5,
    response_mode="compact"  # compact / refine / tree_summarize
)

response = query_engine.query("什么是RAG？")
print(response.response)      # 答案
print(response.source_nodes)  # 来源
```

LlamaIndex 的响应模式：
- **compact**：压缩上下文后生成（类似语境压缩）
- **refine**：逐篇迭代优化答案
- **tree_summarize**：层级总结
- **simple_summarize**：截断后一次性生成

## 3. 数据摄入对比

### 3.1 LangChain

```python
from langchain.document_loaders import (
    PyPDFLoader, TextLoader, CSVLoader, UnstructuredHTMLLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 加载各种格式
pdf_docs = PyPDFLoader("report.pdf").load()
text_docs = TextLoader("notes.txt").load()

# 统一分块
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500, chunk_overlap=50, separators=["\n\n", "\n", "。", "，"]
)
chunks = splitter.split_documents(pdf_docs + text_docs)
```

### 3.2 LlamaIndex

```python
from llama_index import SimpleDirectoryReader
from llama_index.node_parser import SimpleNodeParser

# 自动检测文件类型并解析
documents = SimpleDirectoryReader(
    input_dir="./data",
    recursive=True,
    required_exts=[".pdf", ".txt", ".md", ".csv"]
).load_data()

# 灵活性更高的解析
parser = SimpleNodeParser.from_defaults(
    chunk_size=512,
    chunk_overlap=50,
    include_metadata=True
)
nodes = parser.get_nodes_from_documents(documents)
```

> LlamaIndex 的 `SimpleDirectoryReader` 自动检测文件类型，LangChain 需要显式指定 Loader。

## 4. 高级功能对比

| 功能 | LangChain | LlamaIndex |
|------|-----------|------------|
| Agent/工具调用 | ✅ 一流的 Agent 框架 | ✅ 支持但不如 LC |
| 混合检索 | ✅ 需手动组合 | ✅ 内置 Hybrid Retriever |
| 层级索引 | 需手动构建 | ✅ 内置 TreeIndex |
| 重排序 | ✅ 通过组件组合 | ✅ 内置 NodePostprocessor |
| 查询优化 | 需手动编写 | ✅ 内置 QueryTransform |
| 多模态 | ✅ 通过组件 | ✅ 内置支持 |
| 评估 | ✅ LangSmith 生态 | ✅ 内置评估模块 |
| 生产部署 | ✅ LangServe | ✅ LlamaHub |

## 5. 选型决策树

```
你的核心需求是什么？
├─ 复杂 Agent 工作流、多工具编排 → LangChain
├─ 快速搭建知识库问答 → LlamaIndex
├─ 两者都需要 → 可以混用！
│   例如：LlamaIndex 建索引 + LangChain 做 Agent 编排
└─ 极简需求、不依赖框架 → 手工实现（参考本教程前面的代码）
```

## 6. 混用示例

```python
# LlamaIndex 负责索引构建
from llama_index import VectorStoreIndex
index = VectorStoreIndex.from_documents(documents)
retriever = index.as_retriever(similarity_top_k=10)

# 包装为 LangChain Retriever
from llama_index.langchain_helpers import LlamaIndexRetriever
lc_retriever = LlamaIndexRetriever(retriever)

# LangChain 负责 Agent 编排
from langchain.agents import create_openai_functions_agent
agent = create_openai_functions_agent(llm, [lc_retriever, calculator_tool])
```

## 7. 何时不用框架

以下场景建议直接手写：
- 单一向量库 + 单一 LLM 的简单 RAG
- 需要极致的性能控制（避免框架开销）
- 团队有特定安全/合规要求，依赖越少越好
- 学习阶段——先手写理解原理，再用框架
