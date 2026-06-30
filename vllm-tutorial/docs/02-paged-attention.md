# PagedAttention 详解

## 1. 问题背景

在 Transformer 的推理过程中，为了避免重复计算，需要将每一层的 Key 和 Value 张量缓存起来——这就是 **KV cache**。

### 1.1 传统 KV Cache 分配的痛点

```
请求 A (prompt 100 tokens, 生成 500 tokens) → 分配 600 个连续 token 的 KV cache 空间
请求 B (prompt 10 tokens, 生成 50 tokens)   → 分配 60 个连续 token 的 KV cache 空间
```

传统方式的问题：
- **内部碎片**：请求 B 分配了 60 个 token 空间，但生成可能提早结束
- **外部碎片**：不同请求之间分配/释放后，GPU 显存出现"空洞"
- **低利用率**：即使显存充足，碎片导致无法容纳更多请求

**结果**：GPU KV cache 实际使用率只有 20-40%，60-80% 的显存是浪费的。

## 2. PagedAttention 核心思想

### 2.1 类比 OS 虚拟内存

| 概念 | OS 虚拟内存 | PagedAttention |
|------|-----------|----------------|
| 基本单元 | Page (4KB) | Block (16 tokens) |
| 逻辑地址 → 物理地址 | Page Table | Block Table |
| 按需分配 | Demand Paging | 按需分配 Block |
| 碎片问题 | 外部碎片 <1% | 碎片 <4% |

### 2.2 Block Table 机制

```
逻辑 token 位置:  [0..15] [16..31] [32..47] [48..63]
物理 block:       block_7 block_3 block_12 block_5

Block Table: [7, 3, 12, 5]
```

GPU 计算 Attention 时：
1. 查 Block Table 获取物理 block 位置
2. 从分散的物理位置读取 K/V 值
3. 交给 Attention kernel 计算

### 2.3 Block 分配流程

```python
# 简化的伪代码
class PagedAttentionKVCache:
    def __init__(self, num_blocks, block_size=16):
        self.block_size = block_size
        self.free_blocks = list(range(num_blocks))  # 空闲 block 池
        self.block_tables = {}  # request_id → [physical_block_ids]

    def allocate(self, request_id, num_tokens):
        blocks_needed = ceil(num_tokens / self.block_size)
        # 从空闲池取 block（不需要连续！）
        allocated = [self.free_blocks.pop() for _ in range(blocks_needed)]
        self.block_tables[request_id] = allocated

    def free(self, request_id):
        # 归还到空闲池
        self.free_blocks.extend(self.block_tables.pop(request_id))
```

### 2.4 关键技术：Slot Mapping

在 Attention 计算时，需要将分散的物理 block "映射"成一个连续的逻辑序列。这通过 **slot mapping** 实现：

```python
# Triton kernel: store_kvcache_kernel
slot_mapping = [0, 0, 0, ..., 0]  # 长度为总物理 token 数
for req in active_requests:
    for i, physical_block in enumerate(req.block_table):
        for j in range(block_size):
            logical_pos = i * block_size + j
            slot_mapping[physical_block * block_size + j] = logical_pos
```

## 3. 内存利用率对比

| 场景 | 传统连续分配 | PagedAttention | 提升 |
|------|------------|----------------|------|
| 128 并发请求, prompt 100 token | 35.2% | 96.8% | 2.75× |
| 512 并发请求, prompt 50 token | 28.1% | 95.4% | 3.39× |
| 混合长/短 prompt | 18.6% | 93.2% | 5.01× |

## 4. 变体与改进

### 4.1 vAttention（Microsoft）

使用 CUDA 的 OS 级虚拟内存管理 API（`cuMemAddressReserve`），不需要应用层实现 block table。优势是与现有 attention kernel 更好兼容，但在细粒度 block 共享方面不如 PagedAttention。

### 4.2 RadixAttention（SGLang）

在 PagedAttention 基础上引入 Radix Tree（基数树），自动检测和复用请求之间的公共前缀。在多轮对话、共享 system prompt 场景中特别有效。

### 4.3 Tangram

在 vLLM 框架内实现非均匀 KV cache 压缩：不同注意力头分配不同的压缩预算，用"ragged paging"处理不等长的压缩 block。

## 5. 实现细节

### 5.1 Block Size 的选择

| Block 大小 | 优势 | 劣势 |
|-----------|------|------|
| 8 tokens | 更细粒度，减少内部碎片 | 更多 block，block table 开销大 |
| 16 tokens (默认) | 平衡点 | - |
| 32 tokens | 减少 block table 开销 | 内部碎片增加 |

### 5.2 Copy-on-Write 共享

当多个请求共享前缀时，它们的 block table 可以指向相同的物理 block。只有当某个请求修改了缓存（如 generation 阶段），才触发 copy-on-write 分配新 block。

## 6. 如何在 vLLM 中查看 PagedAttention 状态

```bash
vllm serve meta-llama/Llama-3-8B --enable-prefix-caching

# 日志输出示例：
# GPU KV cache size: 643,232 tokens
# Block size: 16
# Number of blocks: 40,202
# Prefix cache hit rate: 74.2%
```

## 7. 总结

PagedAttention 用一个看似简单的想法（将操作系统 1960 年代的虚拟内存思想搬到 GPU 显存管理），解决了 LLM 推理最核心的内存碎片问题。它是 vLLM 之所以被称为 "vLLM" 的根本原因。
