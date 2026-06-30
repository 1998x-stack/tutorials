# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains a collection of static HTML tutorial courses on technical topics, all published via GitHub Pages under the `1998x-stack` organization. Each tutorial is a standalone directory of self-contained HTML files with a two-column layout (sidebar navigation + main content), written in conversational Chinese.

Tutorials are generated using the **tutorial-publisher skill** (`.claude/skills/tutorial-publisher/`). The skill handles topic research, chapter outline design, HTML generation with themed CSS, and deployment.

## Active Tutorials

| Directory | Topic | URL |
|-----------|-------|-----|
| `CAN-bus-tutorial/` | CAN总线通信 | `https://1998x-stack.github.io/CAN-bus-tutorial/` |
| `HTTP-protocol-tutorial/` | HTTP协议 | `https://1998x-stack.github.io/HTTP-protocol-tutorial/` |
| `machine-learning-tutorial/` | 机器学习 | `https://1998x-stack.github.io/machine-learning-tutorial/` |
| `nlp-tutorial/` | 自然语言处理 | `https://1998x-stack.github.io/nlp-tutorial/` |
| `rag-tutorial/` | RAG检索增强生成 | `https://1998x-stack.github.io/rag-tutorial/` |
| `reinforcement-learning-tutorial/` | 强化学习 | `https://1998x-stack.github.io/reinforcement-learning-tutorial/` |
| `statistics-tutorial/` | 统计学 | `https://1998x-stack.github.io/statistics-tutorial/` |
| `vllm-tutorial/` | vLLM推理引擎 | `https://1998x-stack.github.io/vllm-tutorial/` |

## Tutorial Structure

Every tutorial directory follows the same pattern:
- `index.html` — Course directory with card grid (JS-rendered, `<noscript>` fallback)
- `01.html` through `NN.html` — Chapter pages (28 chapters for complex, 22 for medium, 15 for simple)
- `theme.css` — The theme stylesheet (loaded via `<link>` in every page)
- Each page has its own complete sidebar HTML with all chapter links (no shared nav via JS — navigation works with JS disabled)
- Navigation: prev/next chapter links + "返回目录" link at page bottom

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

## Constraints

- **Static HTML only** — no build tools, no frameworks, no CDN dependencies. System font stack only.
- **No modifications to existing tutorials** unless the user explicitly requests a fix or update.
- **Each tutorial directory has its own git history** (initialized fresh) when deployed to its own GitHub repo. The umbrella repo commits track the files as added to this workspace.
