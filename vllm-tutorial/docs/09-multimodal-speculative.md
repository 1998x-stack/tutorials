# 多模态推理与推测解码

## 1. vLLM 的多模态支持

### 1.1 支持的模态

vLLM 在 V1 中完整支持多模态模型：

| 模态 | 支持的模型 |
|------|-----------|
| 图像（Vision） | LLaVA, Qwen2-VL, Qwen2.5-VL, Qwen3-VL, InternVL2, Phi-3-Vision, Pixtral |
| 视频（Video） | LLaVA-NeXT-Video, Qwen2-VL (video) |
| 音频（Audio） | Whisper (encoder-decoder) |
| 混合 | Fuyu, PALIGEMMA, Chameleon |

### 1.2 多模态推理流程

```
图像输入
    │
    ▼
┌──────────────┐
│  Vision       │  视觉编码器 (ViT/CLIP/SigLIP)
│  Encoder      │  图像 → 视觉 tokens
└──────┬───────┘
       │ visual tokens
       ▼
┌──────────────────────────────────────┐
│  文本 prompt ──► Embedding ──► [visual tokens | text tokens]  │
│                                          │
│                                    LLM Backbone
│                                          │
│                                    生成文本输出
└──────────────────────────────────────┘
```

### 1.3 Vision Encoder CUDA Graphs（2025-2026 新特性）

将 ViT 编码器纳入 CUDA Graph 录制/回放：

```bash
vllm serve Qwen/Qwen2.5-VL-7B-Instruct \
    --mm-processor-cache-gb 4 \
    --limit-mm-per-prompt image=16
```

**性能提升**（4×GPU 场景）：
- P99 延迟降低 84.9%（使用 FLASHINFER backend）
- 支持多 token budget 等级：2048/4096/8192/13824
- Greedy bin-packing 运行时调度

### 1.4 多模态使用示例

```python
from vllm import LLM, SamplingParams

llm = LLM("Qwen/Qwen2.5-VL-7B-Instruct")

sampling_params = SamplingParams(temperature=0.1, max_tokens=256)

outputs = llm.generate([
    {
        "prompt": "这张图片里有什么？",
        "multi_modal_data": {
            "image": "path/to/image.jpg"
        }
    }
])
```

## 2. 推测解码（Speculative Decoding）

### 2.1 问题

自回归解码每步只能生成一个 token，模型有多少层就要等多久。70B 模型每步 ~10ms，生成 1000 tokens 需要 10 秒。

### 2.2 核心思想

用一个**小模型（draft model）**快速生成候选 tokens，然后**大模型（target model）**一次验证多个。

```
Target Model (70B):    慢 (10ms/token) 但准确
Draft Model (0.5B):    快 (0.5ms/token) 但不够好

流程:
1. Draft Model 生成 [T1, T2, T3, T4, T5]  (5×0.5ms = 2.5ms)
2. Target Model 一次验证 5 个 token       (1×10ms = 10ms)
3. 如果 T1-T4 正确, T5 错误:
   接受 T1-T4, 从 T5 开始下一轮
   实际: 12.5ms 生成 4 个 token = 3.1ms/token (比原来的 10ms 快 3.2×)
```

### 2.3 vLLM 支持的方法

| 方法 | 说明 | Draft 来源 |
|------|------|-----------|
| n-gram | 基于已生成文本的 n-gram 匹配 | 无需额外模型 |
| EAGLE | 用特征层预测草稿 | 轻量级 head |
| Medusa | 多个预测头同时猜多个 future tokens | 轻量级 head |
| MLP Speculator | 小型 MLP 做草稿模型 | 独立小网络 |

### 2.4 使用示例

```bash
# n-gram 推测解码（无需 draft model）
vllm serve meta-llama/Llama-3-70B \
    --speculative-config '{"method": "ngram", "num_speculative_tokens": 5}'

# EAGLE 推测解码
vllm serve meta-llama/Llama-3-70B \
    --speculative-config '{"method": "eagle", "draft_model": "path/to/eagle-draft"}'
```

### 2.5 P-EAGLE（2026 方向）

并行分支扩展 + 动态置信度评估：
- 不只是线性生成候选序列，而是并行生成多个可能分支
- 动态评估每个分支的置信度，选择最可能的分支
- 8192 tokens 场景加速 6.7×

## 3. LoRA 适配器

### 3.1 为什么需要 LoRA

LoRA（Low-Rank Adaptation）允许在共享基础模型上加载不同任务的适配器，无需每个任务保存完整模型副本。

### 3.2 vLLM 的 LoRA 支持

```bash
# 启动时加载多个 LoRA
vllm serve meta-llama/Llama-3-8B \
    --enable-lora \
    --max-lora-rank 64 \
    --lora-modules my-math-lora=./loras/math/ \
                   my-code-lora=./loras/code/

# 请求时指定 LoRA
curl http://localhost:8000/v1/chat/completions \
    -d '{
        "model": "my-math-lora",
        "messages": [{"role": "user", "content": "解方程 x² + 3x - 4 = 0"}]
    }'
```

### 3.3 Punica：高效 Multi-LoRA 服务

vLLM 使用 Punica 算法实现多 LoRA 共享 GPU。核心技巧是 **Segmented Gather** 矩阵乘法——将不同 LoRA 的请求分组，高效批量处理。

## 4. 结构化输出

vLLM 支持约束解码保证输出格式：

```python
from pydantic import BaseModel

class PersonInfo(BaseModel):
    name: str
    age: int
    city: str

outputs = llm.generate(
    "提取人物信息：张三，28岁，来自北京。",
    guided_options=GuidedDecodingParams(json=PersonInfo)
)
# 输出保证是合法 JSON: {"name": "张三", "age": 28, "city": "北京"}
```

## 5. 多模态 + LoRA + 推测解码：局限

目前 vLLM 的以下组合存在限制或暂不支持：
- 多模态模型 + LoRA：有限支持（依赖模型架构）
- 推测解码 + LoRA：官方正在开发中
- 多模态 + 推测解码：暂不支持

## 6. 总结

| 功能 | 成熟度 | 推荐场景 |
|------|--------|---------|
| 多模态推理 | 🟢 生产可用 | 视觉问答、文档理解 |
| Vision Encoder CUDA Graph | 🟡 新功能 | 高吞吐多模态 |
| 推测解码 (n-gram) | 🟢 生产可用 | 通用加速 |
| 推测解码 (EAGLE) | 🟢 生产可用 | 需要训练 draft 模型 |
| LoRA | 🟢 生产可用 | 多任务共享基座模型 |
| 结构化输出 | 🟢 生产可用 | API 输出格式化 |
