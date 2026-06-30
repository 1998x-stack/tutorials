# vLLM 推理引擎概述

## 1. 什么是 vLLM

vLLM（Very Large Language Model）是一个高吞吐量、低延迟的 LLM 推理引擎，由 UC Berkeley 的 Ion Stoica 团队开发，于 2023 年 10 月在 SOSP 2023（操作系统原理大会）上正式发表。它是目前业界最流行的开源 LLM 推理框架之一，日均安装量超过 10 万次。

vLLM 要解决的核心问题：**如何在 GPU 资源有限的情况下，最大化 LLM 推理的吞吐量和资源利用率？**

传统的 LLM 推理方式（如 Hugging Face Transformers）存在严重的内存浪费问题——KV cache 占用的 GPU 显存中，有 60-80% 是被浪费的。vLLM 通过 **PagedAttention** 算法，将这一浪费降至 4% 以下。

## 2. 为什么需要专门的推理引擎

| 问题 | 传统方式 | vLLM 方式 |
|------|---------|----------|
| KV cache 内存碎片 | 连续分配，内部+外部碎片 60-80% | 分页管理，碎片 <4% |
| 批处理效率 | 静态批处理，等最慢的请求 | 连续批处理，即完成即补充 |
| 系统 prompt 重复计算 | 每个请求独立计算 | 前缀缓存，共享前缀一次计算 |
| 多 GPU 利用 | 手动配置 | 一行参数自动张量并行 |

## 3. 核心创新

### 3.1 PagedAttention

vLLM 的算法核心，灵感来源于操作系统的虚拟内存分页机制：

- KV cache 被分成固定大小的 **block**（默认 16 个 token）
- 每个请求维护一张 **block table**（类似 OS 页表），将逻辑 token 位置映射到物理 GPU 显存 block
- block 按需分配，不预分配，不要求连续存储
- 物理 block 可以非连续地分散在 GPU 显存中，通过 block table "缝合"起来

**效果**：GPU 显存利用率达到 96%+，batch size 可以提高 2-8 倍。

### 3.2 Continuous Batching（连续批处理）

LLM 推理分为两个阶段：
- **Prefill（预填充）**：一次性处理所有 prompt token，计算密集型
- **Decode（解码）**：逐 token 生成，内存密集型

传统静态批处理中，一旦组 batch，必须等所有请求都完成才能换新 batch。由于输出长度差异巨大（5 tokens vs 2000 tokens），早已完成的请求白白占用 GPU 槽位。

Continuous batching 在每一步迭代后检查：有请求完成了？立刻换入新请求。GPU 利用率从 ~58% 提升到 92%+。

### 3.3 Prefix Caching（前缀缓存）

在 RAG 应用、多轮对话、few-shot prompting 等场景中，多个请求共享相同的前缀（system prompt、历史对话等）。Prefix caching 自动检测并复用已计算的 KV cache，避免重复计算。

## 4. vLLM 生态系统

```
vLLM 核心引擎
├── PagedAttention KV 管理
├── Continuous Batching 调度
├── 分布式推理 (Tensor/Data/Pipeline Parallel)
├── 模型支持 (Decoder-only, Encoder-Decoder, Multimodal)
├── 量化支持 (FP8, AWQ, GPTQ)
├── OpenAI 兼容 API 服务
└── 生产栈 (Kubernetes Helm, Prometheus/Grafana)
```

## 5. 性能数据

| 指标 | Hugging Face TF | vLLM | 提升 |
|------|----------------|------|------|
| 7B 吞吐量 | ~600 tok/s | ~2100 tok/s | 3.5× |
| GPU 利用率 | 35-58% | 92%+ | 1.6-2.6× |
| 内存浪费 | 60-80% | <4% | 15-20× |
| 75% 缓存命中吞吐 | - | 4× 基准 | - |

## 6. 发展历程

| 时间 | 事件 |
|------|------|
| 2023.06 | vLLM 项目启动 |
| 2023.10 | SOSP 2023 论文发表 |
| 2024 | v0.x 快速迭代，社区快速增长 |
| 2025.01 | V1 Alpha 发布，V0 弃用 |
| 2025 Q2 | V1 稳定版目标：自适应批处理、跨节点推理、安全沙箱 |
| 2026 | P-EAGLE 并行解码、多模态 Vision Encoder、Model Runner V2 |

## 7. 竞品对比

| 特性 | vLLM | SGLang | TensorRT-LLM | HuggingFace TGI |
|------|------|--------|-------------|-----------------|
| 开源 | ✅ | ✅ | ✅ | ✅ |
| PagedAttention | ✅ | RadixAttention | ❌ | ❌ |
| 前缀缓存 | ✅ 零开销 | ✅ Radix树 | 有限 | ❌ |
| 安装难度 | 简单 (pip) | 简单 (pip) | 复杂 (编译) | 简单 (docker) |
| 模型覆盖 | 极广 | 广 | NVIDIA 专用 | HuggingFace 模型 |
| 多硬件 | NVIDIA/AMD/Intel/TPU | NVIDIA/AMD | NVIDIA only | NVIDIA/AMD |

## 8. 哪些公司在用 vLLM

- **Stripe**：迁移至 vLLM 后推理成本降低 73%，1/3 GPU 集群处理 5000 万次日均调用
- **Anthropic**：内部广泛使用
- **Databricks**：集成到 MosaicML 产品线
- **AWS**：SageMaker 和 Bedrock 均提供 vLLM 部署选项
- **众多创业公司**：作为 LLM 推理的默认选择
