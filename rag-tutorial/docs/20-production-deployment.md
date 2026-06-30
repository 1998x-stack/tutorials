# 生产级 RAG 部署

## 1. 生产部署总览

将 RAG 从原型推向生产，需要解决：
- 性能与延迟控制
- 高可用与容灾
- 成本优化
- 监控与告警
- 持续迭代

## 2. 部署架构

### 2.1 容器化部署

```yaml
# docker-compose.yml
services:
  embedding:
    image: bge-m3-server:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  vector-db:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
    environment:
      QDRANT__SERVICE__GRPC_PORT: 6334

  llm:
    image: vllm/vllm-openai:latest
    command: --model Qwen/Qwen2.5-7B-Instruct --max-model-len 8192
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1

  api:
    build: ./rag-api
    ports:
      - "8000:8000"
    depends_on:
      - embedding
      - vector-db
      - llm
    environment:
      EMBEDDING_URL: http://embedding:8080
      VECTOR_DB_URL: http://vector-db:6333
      LLM_URL: http://llm:8000

volumes:
  qdrant_data:
```

### 2.2 水平扩展策略

```
                   Load Balancer (Nginx/Traefik)
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
      API-1           API-2           API-3
          │               │               │
          └───────────────┼───────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        embedding-1  embedding-2  embedding-3
              │           │           │
              └───────────┼───────────┘
                          │
                    Vector DB Cluster
                    (Qdrant/Milvus)
```

## 3. Semantic Cache（语义缓存）

### 3.1 原理
缓存不仅是精确匹配，还要支持语义相近的查询命中缓存：

```python
class SemanticCache:
    def __init__(self, embedding_model, threshold=0.95):
        self.cache = {}  # embedding: (query, answer, timestamp)
        self.model = embedding_model
        self.threshold = threshold

    def get(self, query):
        """查找语义相近的缓存"""
        query_emb = self.model.encode(query)

        for cached_emb, (cached_query, answer, ts) in self.cache.items():
            if cosine_similarity(query_emb, cached_emb) > self.threshold:
                return answer, cached_query  # 返回缓存的答案和原始查询
        return None, None

    def set(self, query, answer):
        """存入缓存"""
        query_emb = self.model.encode(query)
        self.cache[tuple(query_emb)] = (query, answer, time.time())
```

### 3.2 缓存策略

| 缓存层级 | 命中条件 | 延迟节省 |
|---------|---------|---------|
| 精确缓存 | 完全相同的查询 | 100% |
| 语义缓存 | 相似度 > 0.95 | 80-90% |
| 检索缓存 | 相同 Embedding 的检索结果 | 50-70% |

## 4. GPU 资源规划

### 4.1 模型量化

```python
# 使用 bitsandbytes 量化为 4-bit
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype="float16"
)

model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-7B-Instruct",
    quantization_config=bnb_config,
    device_map="auto"
)
```

### 4.2 GPU 规格建议

| 场景 | Embedding 模型 | LLM | 推荐 GPU |
|------|---------------|-----|---------|
| 原型/小规模 | BGE-M3 | 7B (量化) | 1× T4 (16GB) |
| 中等规模 | BGE-M3 | 7B/13B | 1× A10 (24GB) |
| 大规模 | BGE-M3 + Reranker | 70B | 2× A100 (80GB) |
| 超大集群 | 多模型并行 | 多模型 | 4+× H100 |

## 5. 成本优化

### 5.1 Token 消耗拆解

```
一次典型 RAG 请求的 Token 消耗：
├─ Embedding API：~100 tokens（查询向量化）
├─ 检索：0 tokens（向量库查询不消耗 Token）
├─ Reranker API：~500 tokens（Query + Top-25 文档片段）
├─ LLM Prompt：~2000 tokens（System + Context + Query）
└─ LLM Generation：~500 tokens（输出）
─────────────────────────────
总计：~3100 tokens/请求
```

### 5.2 优化技巧

| 技巧 | 节省 | 代价 |
|------|------|------|
| Semantic Cache | 50-80%（高频查询） | 缓存管理成本 |
| Prompt 精简 | 10-20% | 可能影响质量 |
| 自适应 Top-K | 10-30% | 需要路由逻辑 |
| 模型量化 | 50% 显存 | 轻微精度损失 |
| 小模型做路由 | 5-10% | 增加一次调用 |

## 6. 监控与告警

```python
# Prometheus 指标示例
metrics = {
    # 延迟指标
    "rag_request_latency_seconds": Histogram(),
    "rag_retrieval_latency_seconds": Histogram(),
    "rag_generation_latency_seconds": Histogram(),

    # 质量指标
    "rag_cache_hit_ratio": Gauge(),
    "rag_faithfulness_score": Gauge(),

    # 系统指标
    "rag_requests_total": Counter(),
    "rag_errors_total": Counter(),
    "rag_active_requests": Gauge(),
}

# 告警规则
alerts = [
    ("P99延迟 > 5s", "warning"),
    ("错误率 > 1%", "critical"),
    ("忠实度分数 < 0.6", "warning"),
    ("缓存命中率 < 30%", "info"),
]
```

## 7. 容灾设计

| 组件 | 高可用方案 |
|------|-----------|
| API 层 | 多副本 + Load Balancer |
| Embedding | 多副本 + 备选模型 |
| Vector DB | 主从复制 + 自动故障转移 |
| LLM | 多模型备选（主模型故障 → 备用模型） |
| 缓存 | Redis Cluster |

## 8. 部署检查清单

```
□ 所有组件容器化
□ 健康检查和存活探针配置
□ 日志采集和聚合（ELK / Loki）
□ 链路追踪（OpenTelemetry / Jaeger）
□ 指标监控（Prometheus + Grafana）
□ 告警规则配置
□ 自动扩缩容策略
□ 备份策略（向量库 + 原始文档）
□ 灰度发布机制
□ 压力测试（目标 QPS + 50% 余量）
□ 安全加固（TLS/认证/授权）
```
