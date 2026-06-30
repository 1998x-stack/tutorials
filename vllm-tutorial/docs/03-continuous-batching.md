# Continuous Batching 机制

## 1. 为什么需要动态批处理

### 1.1 LLM 推理的特殊性

LLM 推理是 **自回归** 的：输出长度不确定。一个 prompt "写一首五言绝句" 可能生成 20 tokens，而 "给我写一篇 5000 字论文" 可能生成 8000+ tokens。

| 请求 | 输入 tokens | 输出 tokens | 总时间 |
|------|------------|-------------|--------|
| A: 简单问答 | 10 | 15 | 0.8s |
| B: 文章生成 | 50 | 2000 | 30s |
| C: 翻译 | 100 | 500 | 8s |

### 1.2 静态批处理的问题

```
时间线:
├── [A ✓] .......................... (等待 B 完成, 白白占用 GPU)
├── [B ============================]
├── [C ✓✓✓✓✓] .................. (等待 B 完成, 白白占用 GPU)
└── 新请求 D/E/F 只能排队等待...
```

静态批处理中，即便请求 A 只用 0.8 秒就完成了，它的 GPU 槽位仍然不能被释放，要等到最慢的 B 请求（30 秒）完成。

## 2. Continuous Batching 原理

### 2.1 迭代级动态调度

Continuous batching 将调度粒度从"请求级"细化到"token 级"：

```
每个 decode step 之后:
├── 检查已完成的请求 → 释放 KV cache block
├── 从等待队列取新请求 → 分配 KV cache block
├── 组新 batch → 执行下一个 forward pass
└── 重复...
```

### 2.2 两种操作：Prefill 和 Decode

| 操作 | 输入 | 输出 | 特点 | 耗时占比 |
|------|------|------|------|---------|
| Prefill | prompt tokens | 首个 output token + KV cache | 计算密集（矩阵乘） | 10-30% |
| Decode | 前一个 token | 下一个 token | 内存密集（KV cache 读写） | 70-90% |

**Chunked Prefill**（vLLM 默认启用）：将长 prompt 拆成多个 chunk，分散到多个 decode step 中处理，避免一个长 prompt 阻塞所有请求。

### 2.3 调度策略

vLLM 支持多种调度策略，通过 `--scheduling-policy` 配置：

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| FCFS (先来先服务) | 按请求到达顺序 | 通用，公平性好 |
| Priority (优先级) | 高优先级请求优先 | SLA 分级场景 |
| Preemption (抢占) | 低优先级请求可被中断 | 保证高优请求低延迟 |

## 3. Chunked Prefill 深入

### 3.1 问题

一个 8192 token 的 prompt，其 prefill 阶段需要大量计算。如果一次性处理，其他短请求的 decode 都会被阻塞。

### 3.2 解法

将 8192 token 拆成 4 个 2048 token 的 chunk，分 4 步处理：

```
Step 1: Prefill chunk[0:2048]   + Decode batch
Step 2: Prefill chunk[2048:4096] + Decode batch
Step 3: Prefill chunk[4096:6144] + Decode batch
Step 4: Prefill chunk[6144:8192] + Decode batch → 进入 decode
Step 5: Decode (已和其他短请求一起)
```

## 4. 实际效果

```
实验配置: Llama-3-8B, A100-80GB, 混合长短请求

静态批处理:
├── 平均 GPU 利用率: 58%
├── P99 延迟: 3200ms
└── 吞吐量: 850 tok/s

Continuous Batching:
├── 平均 GPU 利用率: 92%
├── P99 延迟: 1200ms
└── 吞吐量: 2100 tok/s (+147%)
```

## 5. 代码示意（简化）

```python
class Scheduler:
    def step(self, waiting_queue, running_requests):
        # 1. 移除已完成的请求
        finished = [r for r in running_requests if r.is_done()]
        for r in finished:
            self.kv_cache.free(r.id)
            running_requests.remove(r)

        # 2. 尝试加入新请求
        available_blocks = self.kv_cache.free_block_count()
        for r in waiting_queue:
            blocks_needed = r.blocks_needed()
            if blocks_needed <= available_blocks:
                self.kv_cache.allocate(r.id, blocks_needed)
                running_requests.append(r)
                waiting_queue.remove(r)
                available_blocks -= blocks_needed

        # 3. 组织 batch
        batch = []
        for r in running_requests:
            if r.is_prefill_phase():
                batch.append(('prefill', r))
            else:
                batch.append(('decode', r))

        return batch
```

## 6. 与 PagedAttention 的关系

Continuous batching 和 PagedAttention 是一对绝佳搭档：

- **PagedAttention** 解决了"每个请求的显存怎么高效存"的问题 → 可以容纳更多请求
- **Continuous batching** 解决了"什么时候给请求分配/释放显存"的问题 → 最大化吞吐

没有 PagedAttention，即便用了 continuous batching，GPU 内存碎片会导致无法高效换入新请求。
没有 Continuous batching，PagedAttention 节省的内存也无法转化为吞吐提升。

## 7. 生产中的注意事项

- **`--max-num-seqs`**：限制并发序列数，防止显存溢出
- **`--max-model-len`**：限制单个请求的最大 token 数，防止单个请求霸占全部 KV cache
- **GPU 利用率监控**：正常应在 85-95%，低于 70% 可能需要调整 batch 参数
