# KV Cache 管理与优化技术

## 1. KV Cache 基础知识

### 1.1 为什么需要 KV Cache

Transformer 的自注意力计算：

```
Attention(Q, K, V) = softmax(QK^T / √d) · V
```

在自回归生成中，每生成一个新 token，都需要重新计算所有之前 token 的 K 和 V。直接计算会导致 O(n²) 的重复运算。KV cache 将已计算的 K、V 存储起来，每步只需计算新 token 的 Q 并与缓存的 K、V 做注意力计算。

### 1.2 显存开销估算

以 Llama-3-8B 为例（FP16）：

```
每 token KV cache 大小 = 2 (K + V) × 32 (层数) × 32 (注意力头) × 128 (头维度) × 2 (bytes/FP16)
                      = 524,288 bytes = 0.5 MB/token

8192 token 单请求: 0.5 MB × 8192 = ~4 GB
128 并发请求 × 8192 token: 128 × 4 GB = 512 GB  ← 远超单张 A100 (80GB)
```

## 2. 前缀缓存（Prefix Caching）

### 2.1 原理

在许多场景中，多个请求共享相同的 prompt 前缀：

```
请求 A: [System Prompt (1000 tokens)] + "什么是机器学习？"
请求 B: [System Prompt (1000 tokens)] + "什么是深度学习？"
请求 C: [System Prompt (1000 tokens)] + "解释一下 Transformer"
```

传统方式计算 3 次 system prompt 的 KV cache。Prefix caching 计算一次，多次复用。

### 2.2 vLLM V1 的零开销实现

V1 用 **指针重定向** 替代数据拷贝：

```python
# V0 (有拷贝):
cached_kv = prefix_cache.lookup(prefix_hash)
if cached_kv:
    copy_kv_to_request(cached_kv, request)  # 拷贝开销

# V1 (零拷贝):
cached_blocks = prefix_cache.lookup(prefix_hash)
if cached_blocks:
    request.block_table = cached_blocks  # 只改指针，不拷贝数据
```

### 2.3 命中率与收益

| 场景 | 典型命中率 | 成本节约 |
|------|-----------|---------|
| RAG 应用（固定 system prompt） | 60-90% | 50-80% |
| 多轮对话 | 40-70% | 30-50% |
| 批量评估（共享 few-shot examples） | 80-95% | 60-85% |
| 单次问答 | 0-10% | 0-5% |

## 3. KV Cache 量化

### 3.1 为什么要量化

KV cache 是显存的主要消耗者（尤其长上下文场景）。量化可以 2-4× 压缩 KV cache，代价是轻微的精度损失。

### 3.2 量化方案对比

| 方案 | 量化位宽 | 压缩比 | 精度损失 | 特点 |
|------|---------|--------|---------|------|
| FP8 | 8 bit | 2× | 极小 | vLLM 原生支持，推荐首选 |
| INT8 | 8 bit | 2× | 小 | 静态量化，需校准 |
| KIVI | 2-4 bit | 4-8× | 中 | 按 channel 分组量化 |
| KVQuant | 3-4 bit | 4-5.3× | 中 | 引入非均匀量化 |
| GEAR | 动态 | 3-6× | 小 | 用低秩矩阵补偿误差 |

### 3.3 在 vLLM 中启用 FP8 KV Cache

```bash
vllm serve meta-llama/Llama-3-8B \
    --kv-cache-dtype fp8 \
    --max-model-len 32768
```

启用后，同样的显存可容纳 2× 的 token 数，支持更长的上下文或更大的 batch size。

## 4. KV Cache 逐出与压缩

### 4.1 问题：不是所有 token 都同等重要

Attention 权重不是均匀分布的：
- **"注意力沉没"（Attention Sink）**：前几个 token 吸收了不成比例的高注意力权重
- 中间 token 的注意力得分通常较低
- 最近的 token（局部窗口）对当前生成影响最大

### 4.2 逐出策略

| 策略 | 方法 | 效果 |
|------|------|------|
| SnapKV | 基于注意力得分保留 Top-K 重要的 KV | 可压缩 70%+ |
| KeyDiff | 基于相邻层 KV 差异判断重要性 | 更适合深层模型 |
| FastKVzip | 综合注意力得分 + 层间差异 | 更稳定 |
| 滑动窗口 | 只保留最近 W 个 token | 最简单，长程依赖会丢失 |

### 4.3 Tangram：非均匀 KV 缓存压缩

Tangram 的关键洞察：不同注意力头对 token 的"重要性判断"不同。与其用统一的压缩率，不如给每个头分配独立的压缩预算。

- 重要的头：保留更多 KV
- 不重要的头：激进压缩
- 用"ragged paging"处理不等长压缩块

## 5. GPU ↔ CPU 分层存储

对于超长上下文或多轮对话，可以将 KV cache 在 GPU 和 CPU 之间分层：

```
GPU (HBM): 热数据 → 当前活跃请求的 KV cache
    ↕ LRU/FIFO 交换
CPU (DRAM): 冷数据 → 之前轮次的对话历史 KV cache
    ↕
Disk (SSD): 归档数据 → 长期会话存档
```

LMCache（vLLM 生产栈内置）实现了这一机制，对长对话场景特别有效。

## 6. 监控与调优

```bash
# 查看 KV cache 使用情况
vllm serve model --enable-prefix-caching

# Grafana 关键指标
- kv_cache_usage_percent    # KV cache 使用率
- prefix_cache_hit_rate     # 前缀缓存命中率  
- num_blocks_total          # 总 block 数
- num_blocks_free           # 空闲 block 数
```

## 7. 最佳实践总结

| 场景 | 推荐配置 |
|------|---------|
| 通用推理 | `--enable-prefix-caching --kv-cache-dtype fp8` |
| 超长上下文（>128K） | `--enable-prefix-caching --kv-cache-dtype fp8 --max-model-len 128000` |
| 多轮对话（>10 轮） | `--enable-prefix-caching` + LMCache |
| 内存受限 GPU（24GB） | `--kv-cache-dtype fp8 --gpu-memory-utilization 0.95` |
| 评测/基准测试 | `--enable-prefix-caching`（覆盖 80-95%）|
