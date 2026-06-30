# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains 56 static HTML tutorial courses on technical topics, all published via GitHub Pages under the `1998x-stack` organization. Each tutorial is a standalone directory of self-contained HTML files with a two-column layout (sidebar navigation + main content), written in conversational Chinese.

Tutorials are generated using the **tutorial-publisher skill** (`.claude/skills/tutorial-publisher/`). The skill handles topic research, chapter outline design, HTML generation with themed CSS, and deployment.

## Active Tutorials

### AI / 数据科学 (20)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `machine-learning-tutorial/` | 机器学习 | 28 | `https://1998x-stack.github.io/machine-learning-tutorial/` |
| `nlp-tutorial/` | NLP自然语言处理 | 28 | `https://1998x-stack.github.io/nlp-tutorial/` |
| `deep-learning-tuning-tutorial/` | 深度学习调参 | 28 | `https://1998x-stack.github.io/deep-learning-tuning-tutorial/` |
| `rag-tutorial/` | RAG检索增强生成 | 28 | `https://1998x-stack.github.io/rag-tutorial/` |
| `reinforcement-learning-tutorial/` | 强化学习 | 28 | `https://1998x-stack.github.io/reinforcement-learning-tutorial/` |
| `recommendation-system-tutorial/` | 推荐系统 | 28 | `https://1998x-stack.github.io/recommendation-system-tutorial/` |
| `multivariate-statistics-tutorial/` | 多元统计分析 | 28 | `https://1998x-stack.github.io/multivariate-statistics-tutorial/` |
| `statistics-tutorial/` | 统计学 | 28 | `https://1998x-stack.github.io/statistics-tutorial/` |
| `time-series-analysis-tutorial/` | 时间序列分析 | 28 | `https://1998x-stack.github.io/time-series-analysis-tutorial/` |
| `pytorch-tutorial/` | PyTorch深度学习 | 28 | `https://1998x-stack.github.io/pytorch-tutorial/` |
| `tts-model-history-tutorial/` | TTS模型发展史 | 28 | `https://1998x-stack.github.io/tts-model-history-tutorial/` |
| `llm-training-deployment-tutorial/` | 大模型训练与部署 | 28 | `https://1998x-stack.github.io/llm-training-deployment-tutorial/` |
| `sampling-tutorial/` | 抽样技术 | 28 | `https://1998x-stack.github.io/sampling-tutorial/` |
| `llama-tutorial/` | LLaMA大模型技术 | 28 | `https://1998x-stack.github.io/llama-tutorial/` |
| `deepseek-tutorial/` | DeepSeek模型系列 | 28 | `https://1998x-stack.github.io/deepseek-tutorial/` |
| `yolo-tutorial/` | YOLO目标检测 | 28 | `https://1998x-stack.github.io/yolo-tutorial/` |
| `CV-tutorial/` | 计算机视觉经典模型 | 28 | `https://1998x-stack.github.io/CV-tutorial/` |
| `MOE-tutorial/` | MoE混合专家模型 | 28 | `https://1998x-stack.github.io/MOE-tutorial/` |
| `llm-agent-engineering-tutorial/` | 大模型工程实践：Agent框架 | 28 | `https://1998x-stack.github.io/llm-agent-engineering-tutorial/` |
| `multi-agent-systems-tutorial/` | 多自主智能体系统 | 28 | `https://1998x-stack.github.io/multi-agent-systems-tutorial/` |
| `aigc-tutorial/` | AIGC生成模型 | 28 | `https://1998x-stack.github.io/aigc-tutorial/` |
| `pretraining-dataset-tutorial/` | LLM预训练数据集 | 28 | `https://1998x-stack.github.io/pretraining-dataset-tutorial/` |
| `dataset-analysis-tutorial/` | 数据集分析与实战 | 28 | `https://1998x-stack.github.io/dataset-analysis-tutorial/` |
| `post-training-dataset-tutorial/` | 后训练数据集分析与构建流程 | 28 | `https://1998x-stack.github.io/post-training-dataset-tutorial/` |

### 数学基础 (9)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `numerical-analysis-tutorial/` | 数值分析 | 28 | `https://1998x-stack.github.io/numerical-analysis-tutorial/` |
| `advanced-matrix-analysis-tutorial/` | 高级矩阵分析 | 28 | `https://1998x-stack.github.io/advanced-matrix-analysis-tutorial/` |
| `calculus-tutorial/` | 微积分 | 28 | `https://1998x-stack.github.io/calculus-tutorial/` |
| `electromagnetism-tutorial/` | 电磁学 | 28 | `https://1998x-stack.github.io/electromagnetism-tutorial/` |
| `operations-research-scheduling-tutorial/` | 运筹学与调度理论 | 28 | `https://1998x-stack.github.io/operations-research-scheduling-tutorial/` |
| `random-signal-analysis-tutorial/` | 随机信号分析 | 28 | `https://1998x-stack.github.io/random-signal-analysis-tutorial/` |
| `digital-signal-processing-tutorial/` | 数字信号处理DSP | 28 | `https://1998x-stack.github.io/digital-signal-processing-tutorial/` |
| `olympiad-number-theory-tutorial/` | 数论(高中奥数竞赛) | 22 | `https://1998x-stack.github.io/olympiad-number-theory-tutorial/` |
| `combinatorial-olympiad-tutorial/` | 高中奥数组合理论 | 28 | `https://1998x-stack.github.io/combinatorial-olympiad-tutorial/` |

### 系统 / 基础设施 (15)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `vllm-tutorial/` | vLLM推理引擎 | 28 | `https://1998x-stack.github.io/vllm-tutorial/` |
| `distributed-systems-tutorial/` | 分布式系统 | 28 | `https://1998x-stack.github.io/distributed-systems-tutorial/` |
| `operating-system-tutorial/` | 操作系统 | 28 | `https://1998x-stack.github.io/operating-system-tutorial/` |
| `CSAPP-tutorial/` | 深入理解计算机系统 | 28 | `https://1998x-stack.github.io/CSAPP-tutorial/` |
| `cuda-tutorial/` | CUDA并行计算 | 28 | `https://1998x-stack.github.io/cuda-tutorial/` |
| `linux-kernel-source-tutorial/` | Linux内核源码 | 28 | `https://1998x-stack.github.io/linux-kernel-source-tutorial/` |
| `memcached-source-analysis-tutorial/` | Memcached源码分析 | 28 | `https://1998x-stack.github.io/memcached-source-analysis-tutorial/` |
| `claude-code-agent-tutorial/` | Claude Code Agent | 28 | `https://1998x-stack.github.io/claude-code-agent-tutorial/` |
| `backend-development-tutorial/` | 后端开发实战 | 28 | `https://1998x-stack.github.io/backend-development-tutorial/` |
| `compiler-principles-tutorial/` | 编译原理 | 28 | `https://1998x-stack.github.io/compiler-principles-tutorial/` |
| `computer-architecture-tutorial/` | 计算机体系结构 | 28 | `https://1998x-stack.github.io/computer-architecture-tutorial/` |
| `integrated-circuit-design-tutorial/` | 集成电路设计 | 28 | `https://1998x-stack.github.io/integrated-circuit-design-tutorial/` |
| `fpga-verilog-tutorial/` | FPGA/Verilog 嵌入式开发 | 28 | `https://1998x-stack.github.io/fpga-verilog-tutorial/` |
| `sqlite-source-analysis-tutorial/` | SQLite源码分析 | 28 | `https://1998x-stack.github.io/sqlite-source-analysis-tutorial/` |
| `redis-source-analysis-tutorial/` | Redis源码分析 | 28 | `https://1998x-stack.github.io/redis-source-analysis-tutorial/` |

### 网络 / 通信 (5)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `HTTP-protocol-tutorial/` | HTTP协议 | 28 | `https://1998x-stack.github.io/HTTP-protocol-tutorial/` |
| `CAN-bus-tutorial/` | CAN总线通信 | 30 | `https://1998x-stack.github.io/CAN-bus-tutorial/` |
| `ethernet-tutorial/` | 以太网通信 | 28 | `https://1998x-stack.github.io/ethernet-tutorial/` |
| `antenna-propagation-tutorial/` | 电波传播与天线 | 28 | `https://1998x-stack.github.io/antenna-propagation-tutorial/` |
| `sensor-network-tutorial/` | 传感网技术 | 28 | `https://1998x-stack.github.io/sensor-network-tutorial/` |

### 经济 / 金融 (11)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `investment-tutorial/` | 投资学 | 28 | `https://1998x-stack.github.io/investment-tutorial/` |
| `economics-principles-tutorial/` | 经济学原理 | 28 | `https://1998x-stack.github.io/economics-principles-tutorial/` |
| `macroeconomics-tutorial/` | 宏观经济学 | 28 | `https://1998x-stack.github.io/macroeconomics-tutorial/` |
| `microeconomics-tutorial/` | 微观经济学 | 28 | `https://1998x-stack.github.io/microeconomics-tutorial/` |
| `monetary-banking-tutorial/` | 货币银行学 | 28 | `https://1998x-stack.github.io/monetary-banking-tutorial/` |
| `financial-analysis-tutorial/` | 金融分析 | 28 | `https://1998x-stack.github.io/financial-analysis-tutorial/` |
| `financial-statement-analysis-tutorial/` | 财报实战分析 | 28 | `https://1998x-stack.github.io/financial-statement-analysis-tutorial/` |
| `cryptocurrency-tutorial/` | 加密货币 | 28 | `https://1998x-stack.github.io/cryptocurrency-tutorial/` |
| `shenlun-tutorial/` | 申论体系 | 28 | `https://1998x-stack.github.io/shenlun-tutorial/` |
| `insurance-tutorial/` | 保险学 | 28 | `https://1998x-stack.github.io/insurance-tutorial/` |
| `securities-investment-analysis-tutorial/` | 证券投资分析 | 28 | `https://1998x-stack.github.io/securities-investment-analysis-tutorial/` |

### 机器人 / 控制 (1)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `robotics-tutorial/` | 机器人学 | 28 | `https://1998x-stack.github.io/robotics-tutorial/` |

### 算法 / 数据结构 (4)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `data-structures-tutorial/` | 数据结构与算法 | 28 | `https://1998x-stack.github.io/data-structures-tutorial/` |
| `path-planning-algorithms-tutorial/` | 路径规划算法 | 28 | `https://1998x-stack.github.io/path-planning-algorithms-tutorial/` |
| `combinatorial-optimization-tutorial/` | 组合优化 | 28 | `https://1998x-stack.github.io/combinatorial-optimization-tutorial/` |
| `leetcode-tutorial/` | LeetCode算法面试 | 28 | `https://1998x-stack.github.io/leetcode-tutorial/` |

### 编程语言 (1)
| Directory | Topic | Chapters | URL |
|-----------|-------|----------|-----|
| `rust-tutorial/` | Rust编程语言 | 28 | `https://1998x-stack.github.io/rust-tutorial/` |

## Tutorial Structure

Every tutorial directory follows the same pattern:
- `index.html` — Course directory with card grid (JS-rendered from `courses.json`)
- `courses.json` — Chapter data: title, emoji, subtitle, chapterCount, and chapters array (each with num, title, topics, file)
- `01.html` through `NN.html` — Chapter pages (28 chapters for complex, 22 for medium, 15 for simple)
- `theme.css` — The theme stylesheet (loaded via `<link>` in every page)
- Each page has its own complete sidebar HTML with all chapter links (no shared nav via JS — navigation works with JS disabled)
- Navigation: prev/next chapter links + "返回目录" link at page bottom

### JSON-Driven Index Pages

**Per-tutorial `courses.json`** — each tutorial directory has a `courses.json` that drives its `index.html`:
```json
{
  "title": "CAN总线 · 从入门到精通",
  "emoji": "🚌",
  "subtitle": "📘 30章 实战体系",
  "chapterCount": 30,
  "chapters": [
    { "num": "01", "title": "CAN总线概述", "topics": ["发展历史", "汽车电子应用"], "file": "01.html" }
  ]
}
```

**Root `tutorials.json`** — the hub `index.html` loads from this file, which organizes all tutorials by discipline category. Adding a new tutorial requires only:
1. Create the tutorial directory with chapters and `courses.json`
2. Add an entry to `tutorials.json` under the appropriate category
3. Update `README.md` and `CLAUDE.md` with the new tutorial entry

**Root `index.html`** — the hub page at `https://1998x-stack.github.io/tutorials/` loads `tutorials.json` and renders categorized sections with tutorial cards, each linking to its GitHub Pages URL.

## Generating New Tutorials

Use the `/tutorial-publisher` slash command or invoke the skill directly. The skill follows 7 phases:
1. **Phase 0**: Web search for current, accurate topic information
2. **Phase 1**: Understand topic — confirm scope, audience, chapter count
3. **Phase 2**: Design chapter outline — 6 parts, request user approval
4. **Phase 3**: Select theme CSS from `assets/themes/` and setup project directory
5. **Phase 4**: Generate chapters via parallel subagents (minimum 3 at a time)
6. **Phase 5**: Deploy to GitHub Pages via `gh` CLI
7. **Phase 6**: Self-review with subagent, fix issues, present summary

### Theme Selection

| Domain | CSS File | Example Topics |
|--------|----------|----------------|
| Protocol/Network | `blue-purple.css` | HTTP, TCP/IP |
| Hardware/Embedded | `green.css` | CAN bus, I2C |
| Language/Framework | `warm.css` | Python, React |
| Data/AI | `violet-teal.css` | ML, SQL, NLP |
| Systems/Infra | `slate-blue.css` | Docker, vLLM |

## Deployment

Each tutorial is deployed as its own GitHub repo under `1998x-stack/`:
```bash
cd <tutorial-dir>
git init && git add . && git commit -m "Initial commit"
git branch -m master main
gh repo create 1998x-stack/<repo-name> --public --source=. --remote=origin --push
gh api repos/1998x-stack/<repo-name>/pages -X POST -f "source[branch]=main" -f "source[path]=/"
```

The `gh` CLI must be authenticated as the `1998x-stack` account.

## Commit Conventions

Commits follow batch-based grouping (observed from git history):
- `Add Chapter 01: <title> — <summary>` (first chapter solo)
- `Add Chapters 02-07: <part-name> — <summary>`
- `Add Chapters 08-14: <part-name> — <summary>`
- Continue per batch through all chapters
- `Add index.html — course directory with <N>-chapter card grid`

All commits include `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>`.

## Post-Tutorial Checklist

After completing a new tutorial, always update:
1. **`courses.json`** in the tutorial directory — drives its `index.html`
2. **`tutorials.json`** at repo root — add entry under the correct discipline category
3. **`README.md`** — add the tutorial to the appropriate category table
4. **`CLAUDE.md`** — add the tutorial to the Active Tutorials section

## Constraints

- **Static HTML only** — no build tools, no frameworks, no CDN dependencies. System font stack only.
- **No modifications to existing tutorials** unless the user explicitly requests a fix or update.
- **Each tutorial directory has its own git history** (initialized fresh) when deployed to its own GitHub repo. The umbrella repo commits track the files as added to this workspace.
