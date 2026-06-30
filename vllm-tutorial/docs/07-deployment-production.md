# vLLM 部署与生产实践

## 1. 安装方式

### 1.1 uv 安装（官方推荐）

```bash
pip install --upgrade pip
pip install uv
uv pip install -U vllm --torch-backend=auto
```

### 1.2 Docker 安装

```bash
docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model meta-llama/Llama-3-8B
```

### 1.3 从源码编译

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
pip install -e .
```

## 2. 基础服务部署

### 2.1 单模型推理服务

```bash
vllm serve meta-llama/Llama-3-8B --host 0.0.0.0 --port 8000
```

### 2.2 关键启动参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `--model` | 模型名或路径 | HuggingFace model ID 或本地路径 |
| `--host` | 监听地址 | `0.0.0.0` |
| `--port` | 端口 | `8000` |
| `--tensor-parallel-size` | 张量并行 GPU 数 | 等于 GPU 数量 |
| `--gpu-memory-utilization` | GPU 显存利用率 | `0.90` - `0.98` |
| `--max-model-len` | 最大模型上下文长度 | 根据业务需求设定 |
| `--max-num-seqs` | 最大并发序列数 | 根据显存和业务调整 |
| `--enable-prefix-caching` | 启用前缀缓存 | **强烈建议启用** |
| `--kv-cache-dtype` | KV cache 数据类型 | `fp8`（节省显存）或 `auto` |
| `--enforce-eager` | 禁用 CUDA Graph | 开发调试时开启 |
| `--quantization` | 量化方法 | `fp8`, `awq`, `gptq` |

### 2.3 在线服务 API

vLLM 提供 OpenAI 兼容 API：

```bash
# Chat Completion
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Llama-3-8B",
        "messages": [
            {"role": "system", "content": "你是一个有用的AI助手。"},
            {"role": "user", "content": "解释什么是vLLM？"}
        ],
        "temperature": 0.7,
        "max_tokens": 256
    }'

# 兼容 OpenAI SDK
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-needed")
```

### 2.4 离线批量推理

```python
from vllm import LLM, SamplingParams

llm = LLM("meta-llama/Llama-3-8B", tensor_parallel_size=2,
         gpu_memory_utilization=0.95, enable_prefix_caching=True)

sampling_params = SamplingParams(temperature=0.8, max_tokens=512)

prompts = [
    "什么是人工智能？",
    "Python和Rust有什么区别？",
    "解释一下量子计算"
]

outputs = llm.generate(prompts, sampling_params)
for output in outputs:
    print(output.outputs[0].text)
```

## 3. 生产栈部署（Kubernetes）

### 3.1 vLLM Production Stack

vLLM 官方提供 Kubernetes 原生部署方案：

```bash
# 添加 Helm repo
helm repo add vllm https://vllm-project.github.io/production-stack
helm install vllm vllm/vllm-stack \
    -f values.yaml
```

### 3.2 配置示例

```yaml
servingEngineSpec:
  modelSpec:
    - name: "my-model"
      repository: "vllm/vllm-openai"
      tag: "latest"
      modelURL: "meta-llama/Llama-3-8B"
      replicaCount: 2
      requestGPU: 1
      args:
        - --enable-prefix-caching
        - --kv-cache-dtype fp8
```

### 3.3 生产栈功能

| 功能 | 说明 |
|------|------|
| 前缀感知路由 | 自动将相同前缀的请求路由到同一实例 |
| KV Cache 共享 | 通过 LMCache/NIXL 跨实例共享 KV cache |
| Prometheus 指标 | TTFT、TPOT、端到端延迟 |
| Grafana 仪表板 | 预置可视化面板 |
| 自动扩缩 | 基于 GPU 利用率的 HPA |

## 4. 量化部署

### 4.1 量化方法选择

| 方法 | 压缩比 | 精度 | 适用场景 |
|------|--------|------|---------|
| FP8 | 2× | 几乎无损 | 首选，新硬件原生支持 |
| AWQ | 3-4× | 微量损失 | 节省显存、H100 以下 GPU |
| GPTQ | 3-4× | 微量损失 | 类似 AWQ，社区更广 |
| INT8 | 2× | 小损失 | 通用 |
| BitsandBytes 8bit | 2× | 小损失 | 开发测试 |

### 4.2 使用示例

```bash
# FP8 量化推理
vllm serve neuralmagic/Llama-3-8B-FP8 --quantization fp8

# AWQ 量化推理
vllm serve TheBloke/Llama-3-8B-AWQ --quantization awq
```

## 5. 可观测性

### 5.1 关键指标

| 指标 | 说明 | 目标 |
|------|------|------|
| TTFT (Time to First Token) | 首 token 延迟 | <200ms (P99) |
| TPOT (Time per Output Token) | 每个输出 token 时间 | <20ms (mean) |
| E2E Latency | 端到端延迟 | <3s (P95) |
| Throughput | tokens/s | 最大化 |
| GPU Utilization | GPU 利用率 | 85-95% |
| KV Cache Usage | KV 缓存使用率 | 70-90% |
| Queue Depth | 等待队列长度 | <10 |

### 5.2 日志与追踪

```bash
# 设置日志级别
export VLLM_LOGGING_LEVEL=INFO

# 结构化日志
vllm serve model --disable-log-requests  # 生产环境建议关闭请求日志
```

## 6. 安全最佳实践

- **API 鉴权**：使用反向代理（nginx, envoy）添加认证层
- **速率限制**：防止单个用户耗尽 GPU 资源
- **模型访问控制**：不同团队/应用路由到不同模型实例
- **输入过滤**：对用户 prompt 进行长度和内容检查
- **资源隔离**：使用 Kubernetes namespace 隔离不同环境

## 7. Stripe 案例分析

Stripe 从其他方案迁移到 vLLM 后的效果：

| 指标 | 迁移前 | 迁移后 |
|------|--------|--------|
| GPU 数量 | 3× 集群 | 1× 集群 |
| 推理成本 | 基准 | -73% |
| 日均调用量 | - | 5000 万次 |
| P99 延迟 | - | 显著降低 |

## 8. 常见部署坑

1. **`--ipc=host` 在 Docker 中很关键**：没有它 NCCL 共享内存通信会失败
2. **HuggingFace Token**：gated model 需要 `HF_TOKEN` 环境变量
3. **`--max-model-len` 设置过大**：浪费显存预分配，设置合理的值
4. **不启用 prefix caching**：在 RAG/多轮对话场景中损失 30-80% 吞吐
5. **CUDA Graph 兼容性**：某些自定义模型需要 `--enforce-eager`
