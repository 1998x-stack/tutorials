# vLLM 分布式推理

## 1. 为什么需要分布式推理

### 1.1 显存墙

| 模型 | 参数量 | FP16 显存需求 |
|------|--------|-------------|
| Llama-3-8B | 8B | ~16 GB |
| Llama-3-70B | 70B | ~140 GB |
| DeepSeek-R1 | 671B (37B 激活) | ~140 GB + KV cache |
| Llama-3-405B | 405B | ~810 GB |

单张 A100 (80GB) 只能装下 8B 模型。更大的模型需要多 GPU 联合推理。

## 2. 三种并行策略

vLLM 基于 Megatron-LM 的并行算法，支持三种策略：

```
                    ┌─────────────┐
                    │  模型权重    │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │ Tensor       │ │ Pipeline     │ │ Data         │
   │ Parallelism  │ │ Parallelism  │ │ Parallelism  │
   │ (张量并行)    │ │ (流水线并行)  │ │ (数据并行)    │
   ├──────────────┤ ├──────────────┤ ├──────────────┤
   │ 每层分片到    │ │ 不同层分到    │ │ 完整模型副本   │
   │ 多个 GPU     │ │ 不同 GPU     │ │ 不同 GPU     │
   ├──────────────┤ ├──────────────┤ ├──────────────┤
   │ 单节点内      │ │ 多节点间      │ │ 单节点/多节点  │
   │ 高带宽需求    │ │ 低带宽可接受   │ │ 无通信开销    │
   └──────────────┘ └──────────────┘ └──────────────┘
```

## 3. 张量并行（Tensor Parallelism）

### 3.1 工作原理

将一个 Transformer 层的权重矩阵切分到多个 GPU 上：

```
原来 (单 GPU):
├── Attention: W_Q, W_K, W_V, W_O  [完整矩阵]
├── MLP: W_up, W_gate, W_down      [完整矩阵]
└── 每 GPU 存一份完整拷贝 → 显存不够

TP=4 (4 GPU):
├── Attention: 每个 GPU 存 1/4 的 heads
│   ├── GPU 0: heads[0:8]
│   ├── GPU 1: heads[8:16]
│   ├── GPU 2: heads[16:24]
│   └── GPU 3: heads[24:32]
├── MLP: 每 GPU 存 1/4 列
└── All-Reduce 同步中间结果
```

### 3.2 使用方式

```bash
# 单节点 4 GPU 张量并行
vllm serve meta-llama/Llama-3-70B \
    --tensor-parallel-size 4

# Python 离线推理
from vllm import LLM
llm = LLM("meta-llama/Llama-3-70B", tensor_parallel_size=4)
```

### 3.3 通信开销

张量并行每层都需要 All-Reduce 通信，因此要求高带宽：
- **NVLINK**：900 GB/s — 理想
- **PCIe 4.0**：64 GB/s — 可用，但 TP>2 时吞吐下降明显
- **InfiniBand**：网络通信 — 不推荐用于张量并行跨节点

## 4. 流水线并行（Pipeline Parallelism）

### 4.1 工作原理

将模型按层切分。例如一个 32 层模型在 4 个节点上：

```
节点 1: Layers 0-7   → 节点 2: Layers 8-15 →
节点 3: Layers 16-23 → 节点 4: Layers 24-31
```

每个节点处理完自己的层后，将中间激活传给下一个节点。

### 4.2 使用方式

```bash
# 2 节点 × 8 GPU: 适合 405B 级模型
vllm serve /path/to/model \
    --tensor-parallel-size 8 \
    --pipeline-parallel-size 2 \
    --distributed-executor-backend ray
```

### 4.3 何时使用

- 模型太大，单节点装不下 → PP 跨节点
- GPU 之间没有 NVLINK（如 L40S）→ 优先 PP 而非 TP

## 5. 数据并行（Data Parallelism）

### 5.1 工作原理

每个 GPU（或 GPU 组）拥有完整模型副本，独立处理不同的请求。

```bash
# 8 GPU: 4 个独立引擎，每个用 2 GPU (TP=2)
vllm serve meta-llama/Llama-3-70B \
    --data-parallel-size 4 \
    --tensor-parallel-size 2
```

### 5.2 限制

- CUDA Graphs 和 torch.compile **在 DP 模式下被禁用**
- 每个 engine 是独立进程，不共享 KV cache
- 前缀缓存在 engine 之间不共享

## 6. MoE 模型并行

Mixture-of-Experts 模型有特殊的并行策略：

- **Expert Parallelism**：将不同的 expert 分配到不同 GPU
- **All-to-All 通信**：每层完成后重排 token 到正确的 expert GPU

vLLM 原生支持 DeepSeek-V2/V3、Mixtral 等 MoE 架构。

## 7. 多节点部署

### 7.1 网络要求

| 通信类型 | 带宽需求 | 推荐 |
|---------|---------|------|
| Tensor Parallel（跨节点） | 高 | InfiniBand + GPUDirect RDMA |
| Pipeline Parallel | 中 | 25GbE+ |
| Data Parallel | 无 | 任何 |

### 7.2 调试网络

```bash
NCCL_DEBUG=TRACE vllm serve model --tensor-parallel-size 8

# 查看日志中 NCCL 通信路径:
# "via NET/IB/GDRDMA" → 使用了 InfiniBand RDMA (好)
# "via NET/Socket" → 回退到 TCP (慢)
```

## 8. 性能调优建议

| 场景 | 推荐配置 |
|------|---------|
| 8B 模型 + 单 A100 | 不需要并行 |
| 70B 模型 + 2×A100 | `--tensor-parallel-size 2` |
| 70B 模型 + 4×A100 | `--tensor-parallel-size 4` |
| 405B 模型 + 2 节点 × 8×A100 | `--tensor-parallel-size 8 --pipeline-parallel-size 2` |
| 高吞吐 8B 模型 + 4×A100 | `--data-parallel-size 4` |

## 9. 常见坑

1. **GPU 数不能被 TP 整除**：vLLM 会报错。使用 `pipeline_parallel_size` 替代
2. **跨节点 TP**：性能很差（除非有 InfiniBand）。优先用 PP 跨节点
3. **DP 模式下 prefix cache 失效**：每个 engine 有自己的 cache
4. **Docker 环境**：确保所有节点使用同样的 image 和模型路径
