# nano-vLLM 详解

## 1. 什么是 nano-vLLM

nano-vLLM 是 vLLM 的一个**最小化、教育性**的重新实现，由社区开发者 GeeeekExplorer 创建。整个代码库约 **1200 行 Python** 加上少量 Triton kernel，旨在让 LLM 推理引擎的内部机制变得易于理解、实验和定制。

**GitHub**: https://github.com/GeeeekExplorer/nano-vllm

## 2. 设计哲学

> "如果你不能构建它，你就不真正理解它。"

nano-vLLM 遵循"最小可工作推理引擎"的设计理念：
- 只用 Python + Triton，没有 C++/CUDA 扩展
- 代码结构清晰，每个模块职责单一
- 保留 vLLM 核心功能，刻意简化边缘 case

## 3. 架构概览

```
nano-vLLM 模块结构:

┌─────────────────────────────────────┐
│               LLM                   │  ← 用户入口 (High-level API)
│  ┌──────────┐  ┌──────────────────┐ │
│  │ Scheduler│  │  LLMEngine       │ │
│  │ (调度器)  │  │  (推理编排器)     │ │
│  └────┬─────┘  └────────┬─────────┘ │
│       │                 │            │
│       │    ┌────────────▼────────┐   │
│       └───►│   ModelRunner       │   │  ← GPU 执行
│            │   (模型执行器)       │   │
│            │  ┌────────────────┐ │   │
│            │  │ CUDA Graph     │ │   │
│            │  │ Flash Attn     │ │   │
│            │  │ KV Cache Mgmt  │ │   │
│            │  └────────────────┘ │   │
│            └─────────────────────┘   │
└─────────────────────────────────────┘

辅助类:
├── Sequence: 单个请求的状态 (token IDs, status, params)
├── SamplingParams: 采样参数 (temperature, top_k, max_tokens)
├── BlockTable: KV cache 的页表管理
└── SequenceGroup: 一组相关序列 (用于 beam search)
```

## 4. 核心实现

### 4.1 KV Cache 管理

nano-vLLM 用自定义的 Triton kernel 替代 vLLM 的 PagedAttention C++/CUDA 实现：

```python
# Triton kernel: store_kvcache_kernel
@triton.jit
def store_kvcache_kernel(
    key_ptr, value_ptr,      # GPU 上的 K/V 张量
    key_cache_ptr, val_cache_ptr,  # KV cache 存储
    slot_mapping_ptr,        # 逻辑位置 → 物理位置的映射
    ...
):
    # 用 slot_mapping 将每个 token 的 K/V 存入对应的物理 block
    ...
```

### 4.2 CUDA Graph 加速

```python
class ModelRunner:
    def capture_cuda_graph(self):
        # 录制一次 forward pass
        self.graph = torch.cuda.CUDAGraph()
        with torch.cuda.graph(self.graph):
            self.output = self.model(**self.graph_inputs)
        # 后续推理直接 replay，减少 kernel launch 开销
```

### 4.3 连续批处理

```python
class Scheduler:
    def schedule(self, waiting_queue, running):
        # 1. 完成序列移出
        # 2. 从 waiting_queue 取新序列
        # 3. 构建 batch
        # 返回: (batch_tokens, block_tables, slot_mapping)
```

## 5. 性能对比

**硬件**: RTX 4070 Laptop (8GB)  
**模型**: Qwen3-0.6B  
**负载**: 256 序列, 输入/输出 100-1024 tokens

| 引擎 | 输出 Tokens | 时间 (s) | 吞吐量 (tok/s) |
|------|------------|---------|---------------|
| nano-vLLM | 133,966 | 93.41 | **1,434.13** |
| vLLM (官方) | 133,966 | 98.37 | 1,361.84 |

nano-vLLM 在这个 benchmark 上比官方 vLLM 快 5.3%。并非 nano-vLLM 更优，而是更轻量的实现在小模型/小 batch 场景中有更少的 overhead。

## 6. 快速上手

```python
from nanovllm import LLM, SamplingParams

# 加载模型
llm = LLM(
    "/path/to/Qwen3-0.6B",
    enforce_eager=True,       # 开发调试时关 CUDA graph
    tensor_parallel_size=1    # 单 GPU
)

# 设置采样参数
sampling_params = SamplingParams(
    temperature=0.6,
    top_p=0.9,
    max_tokens=256
)

# 推理
outputs = llm.generate(
    ["你好，nano-vLLM！请做一下自我介绍。"],
    sampling_params
)
print(outputs[0]["text"])
```

## 7. 与官方 vLLM 的差异

| 方面 | vLLM | nano-vLLM |
|------|------|-----------|
| 代码量 | 10,000+ LOC | ~1,200 LOC |
| 语言 | Python + C++ + CUDA | Python + Triton |
| KV Cache 实现 | PagedAttention (C++/CUDA) | 手动 Triton kernel + slot mapping |
| 张量并行 | 完整 (NCCL sharding) | 基础 (`torch.distributed`) |
| Flash Attention | 多种后端 | flash-attn v2 (可选) |
| 前缀缓存 | 零开销 (V1) | 基本实现 |
| 推测解码 | 多种方法 | 不支持 |
| 多模态 | 支持 | 不支持 |
| 安装 | 需编译环境 | `pip install` |
| 学习曲线 | 陡峭 | 平缓 |
| 适用场景 | 生产环境 | 学习、研究、原型验证 |

## 8. 衍生项目

| 项目 | 说明 |
|------|------|
| nano-vllm-npu | 适配华为昇腾 NPU（`torch.npu` + `NPUGraph`） |
| flex-nano-vllm | 用 PyTorch FlexAttention 替代定制 kernel |
| nano-vllm-omni | 扩展到扩散模型（Wan2.2-TI2V-5B 视频生成） |

## 9. 为什么 nano-vLLM 重要

1. **学习工具**：1200 行代码说清楚了 LLM 推理引擎的三个核心问题——怎么存 KV cache？怎么调度请求？怎么执行模型？
2. **研究平台**：修改几行代码就能验证新的调度策略或 KV cache 管理算法
3. **面试准备**：LLM 推理系统相关岗位常问"你了解 PagedAttention 的实现吗？"——读完 nano-vLLM 就能手写
4. **定制推理**：特殊硬件或特殊模型可以 fork nano-vLLM 快速适配
