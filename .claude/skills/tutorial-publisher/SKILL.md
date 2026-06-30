---
name: tutorial-publisher
description: Generate comprehensive tutorial courses and publish them as static HTML via GitHub Pages under the 1998x-stack organization. Use this skill whenever the user wants to create a tutorial, course, or educational guide on any technical topic — even if they don't explicitly mention "GitHub Pages" or "publish." Triggers on: "create a tutorial about X," "make a course for X," "build tutorial pages," "generate HTML tutorial," "publish a tutorial," or any request to turn materials/knowledge into a structured online course. Also triggers when the user provides a topic and asks to "build chapters" or "publish on github pages."
compatibility: Requires gh CLI authenticated with 1998x-stack org access, git, and a GitHub account.
---

# Tutorial Publisher

Generate a complete tutorial course on any technical topic and publish it via GitHub Pages at `https://1998x-stack.github.io/<topic>-tutorial/`.

The output uses a two-column layout (sidebar navigation + main content) with conversational Chinese teaching style, card-grid index page, and external CSS themes — zero build steps.

## Asset System

This skill ships with pre-built assets in `assets/`:

```
assets/
├── chapter-template.html     # Skeleton for each chapter page
├── index-template.html       # Skeleton for the course directory
└── themes/                   # Pre-built CSS files — one per domain
    ├── blue-purple.css       # Protocol/Network (HTTP, TCP/IP, WebSocket...)
    ├── green.css             # Hardware/Embedded (CAN bus, I2C, SPI...)
    ├── warm.css              # Language/Framework (Python, React, Rust...)
    ├── violet-teal.css       # Data/AI (ML, SQL, transformers...)
    └── slate-blue.css        # Systems/Infra (Docker, K8s, Linux...)
```

Templates use `<link rel="stylesheet" href="theme.css">` — no inline CSS blocks. When generating a tutorial, copy the selected theme CSS into the project directory as `theme.css`. Both chapter pages and the index page load this single stylesheet.

**Read `assets/chapter-template.html` and `assets/index-template.html` before Phase 4** so you understand the exact structure and placeholders.

## Workflow

Follow these phases in order. Do not skip phases.

### Phase 0: Topic Research (Web Search)

Before designing the outline, research the topic to ensure content accuracy and currency.

**What to search:**
- Latest developments, current best practices, and major changes in the field
- Common beginner pitfalls and frequently asked questions
- Popular tools, libraries, or frameworks in the current year (2026)
- Any major shifts in the technology landscape (e.g., deprecated tools, new standards)

**How to research:**
Use `WebSearch` to run 3-5 queries covering different angles of the topic. Save key findings to inform the chapter outline.

### Phase 1: Understand the Topic

Ask the user for their topic. If they provide materials (PDFs, articles, notes, URLs), read them thoroughly.

Combine the user's input with research findings to determine:
- **Topic name** (Chinese + English, for titles and repo naming)
- **Domain category** (protocol, hardware, language, data/ai, systems)
- **Complexity** (simple → 15 chapters, medium → 22 chapters, complex → 28 chapters)
- **Target audience** (beginners, intermediates, or mixed)

Present a one-paragraph summary of what you understood, incorporating key research insights. Ask the user to confirm before proceeding.

### Phase 2: Design the Chapter Outline

Generate a chapter outline following this 6-part structure:

1. **Part 1: Fundamentals** (Ch 1-7) — History, core concepts, basic structures
2. **Part 2: Core Mechanisms** (Ch 8-14) — Key features, protocols, patterns
3. **Part 3: Security / Engineering Practice** (Ch 15-18) — Security, best practices, or engineering concerns
4. **Part 4: Advanced Topics** (Ch 19-22) — Deeper dives, specialized use cases
5. **Part 5: Tools & Practice** (Ch 23-25) — Debugging, tooling, hands-on
6. **Part 6: Next Generation & Project** (Ch 26-28) — Future directions, capstone project

Adapt part themes to the domain. A programming language tutorial might replace "Security" with "Engineering Practice."

Each chapter needs:
- A **title** (Chinese, ~5-10 characters)
- **3-4 knowledge points** (short phrases for the index card)
- A **content outline** (3-6 h3 sections with key topics to cover)

Present the full outline in a table. Ask the user to approve before proceeding.

### Phase 3: Select Visual Theme & Setup Project

Choose a theme CSS based on domain, and set up the project:

| Domain | CSS File | Hue | Example Topics |
|--------|----------|-----|----------------|
| Protocol/Network | `blue-purple.css` | HSL 230-250 | HTTP, TCP/IP, WebSocket |
| Hardware/Embedded | `green.css` | HSL 120-150 | CAN bus, I2C, SPI |
| Language/Framework | `warm.css` | HSL 10-40 | Python, React, Rust |
| Data/AI | `violet-teal.css` | HSL 260-290 | SQL, ML, transformers |
| Systems/Infra | `slate-blue.css` | HSL 200-220 | Docker, K8s, Linux |

**Setup steps:**
1. Create the project directory: `mkdir -p <english-topic>-tutorial`
2. Copy the selected theme CSS from `assets/themes/<file>.css` into the project as `theme.css`
3. Announce the theme and repo name. Proceed without waiting for confirmation unless the user objects.

### Phase 4: Generate HTML Files with Subagent Batching

This is the core generation phase. Use subagents to write chapters in parallel batches.

**Before starting, read:**
- `assets/chapter-template.html` — the HTML skeleton for every chapter
- `assets/index-template.html` — the HTML skeleton for the index page

**Key architectural change:** HTML files use `<link rel="stylesheet" href="theme.css">` — NO inline `<style>` blocks in chapter pages. The `theme.css` file in the project root provides all styles. The index page adds a small inline `<style>` block for card-grid overrides on top of `theme.css`.

#### Chapter Page Construction

For each chapter, take `assets/chapter-template.html` and replace:

| Placeholder | What goes there |
|-------------|-----------------|
| `{{PAGE_TITLE}}` | `第X章 <章节标题>` — browser tab title |
| `{{SIDEBAR_EMOJI}}` | Topic-appropriate emoji (🌐 for web, 🔧 for hardware, 🤖 for AI, etc.) |
| `{{SIDEBAR_LINKS}}` | All chapter `<li><a>` entries, ONE with `class="active"` on the current chapter |
| `{{CHAPTER_CONTENT}}` | The full teaching content (h2, h3, p, tables, pre, div callouts) |
| `{{PREV_LINK}}` | Previous chapter file, or `javascript:void(0)` for chapter 1 |
| `{{NEXT_LINK}}` | Next chapter file, or `javascript:void(0)` for the last chapter |

**Sidebar links format:**
```html
<li><a href="01.html" class="active">第1章 标题</a></li>
<li><a href="02.html">第2章 标题</a></li>
...
```

#### Content Writing Rules

- **Voice:** Conversational Chinese, teacher persona ("大家好，我是你们的讲师")
- **Hook:** Each chapter opens with a 1-2 paragraph hook explaining why the topic matters
- **Real stories:** Include at least one real-world anecdote or "踩坑" (pitfall) story per chapter — these are the most valuable part for learners
- **Elements:** Every chapter must have at least one `<table>`, one `<pre><code>` block, and one callout div
- **Callout types:** `<div class="tip">` (blue info), `<div class="warning">` (red danger), `<div class="highlight">` (yellow key insight)
- **Transitions:** Close each chapter with a transition sentence leading to the next topic
- **Terminology:** Keep technical terms in English where natural (e.g., "TCP连接", "API端点")
- **Length:** Each chapter should be 150-300 lines of content HTML
- **Accuracy:** Use information from Phase 0 research — cite current versions, dates, and best practices

#### Subagent Batching Strategy

Dispatch subagents to write chapters in parallel. The minimum is **3 subagents at a time**. Each subagent handles one batch:

| Batch | Chapters | Subagent Description |
|-------|----------|---------------------|
| A | 01 (template + Ch1) | Single chapter first to set the tone |
| B | 02-07 | Foundation chapters |
| C | 08-14 | Core content chapters |
| D | 15-18 | Practice/security chapters |
| E | 19-22 | Advanced topics |
| F | 23-25 | Tools & practice |
| G | 26-NN | Final chapters |
| H | index.html | Course directory (run AFTER all chapters, since it needs chapter titles/topics) |

**Batches B through G can run in parallel** (they're independent). Batch A should run first to establish the pattern. Batch H runs last.

**Subagent instructions template** — give each subagent:
1. The exact chapters to write (with numbers and titles from Phase 2)
2. The content outline for those chapters
3. Path to `assets/chapter-template.html` (read it to get the HTML skeleton)
4. Path to `assets/themes/<selected>.css` (to understand what CSS classes are available)
5. The path to the project directory (to write files)
6. Content writing rules (from above)

**Important:** The `theme.css` file must be copied into the project directory during Phase 3. Each subagent writes HTML files that `<link rel="stylesheet" href="theme.css">` — they do NOT embed inline styles. This keeps chapter files lean and makes theme changes possible by swapping one CSS file.

#### After Each Subagent Completes

1. Verify the generated chapters have `<link rel="stylesheet" href="theme.css">` (not inline styles)
2. Verify sidebars are complete and `class="active"` is on the correct link
3. Commit the batch: `git add <files> && git commit -m "<message>"`
4. If a subagent's output has issues, fix them before committing — or dispatch a targeted fix

#### Index Page Construction

After all chapters are written and committed, build the index page using `assets/index-template.html`:

- Fill in `{{COURSE_TITLE}}`, `{{CHAPTER_COUNT}}`, `{{HEADER_EMOJI}}`, `{{COURSE_EMOJI}}`
- Fill `{{COURSES_JSON}}` with a JS array of all chapters:
  ```javascript
  [
      { num: '01', title: '章节标题', topics: ['知识点1', '知识点2', '知识点3'], file: '01.html' },
      ...
  ]
  ```
- The index page loads `theme.css` plus has its own inline `<style>` for card-grid overrides

### Phase 5: Deploy to GitHub Pages

1. **Initialize git repo in the project directory:**
   ```bash
   cd <project-dir> && git init && git add . && git commit -m "Initial commit"
   git branch -m master main
   ```

2. **Create the repo and push:**
   ```bash
   gh repo create 1998x-stack/<repo-name> --public --source=. --remote=origin --push
   ```
   If the repo already exists, add remote and push instead.

3. **Enable GitHub Pages:**
   ```bash
   gh api repos/1998x-stack/<repo-name>/pages -X POST -f "source[branch]=main" -f "source[path]=/"
   ```

4. **Verify deployment** (~30 seconds after enabling):
   ```bash
   curl -sI https://1998x-stack.github.io/<repo-name>/ | head -5
   ```

5. **Report the URL:** `https://1998x-stack.github.io/<repo-name>/`

### Phase 6: Self-Review and Fix

After deployment, dispatch a review subagent that inspects all generated HTML files and checks for:

1. **Structural issues:**
   - All chapter-to-chapter navigation links (prev/next) are correct
   - Sidebar has all chapters with correct `class="active"` on each page
   - External CSS loads properly: `<link rel="stylesheet" href="theme.css">` present in every file
   - `theme.css` exists in the project root

2. **Content quality:**
   - Every chapter has at least one table, one `<pre><code>`, and one callout (`tip`/`warning`/`highlight`)
   - No placeholder text like "TODO", "lorem ipsum", or untranslated English
   - Technical claims are consistent across chapters (e.g., version numbers match)
   - Each chapter closes with a transition to the next topic

3. **Visual consistency:**
   - All chapters use the same CSS class names (no custom inline styles)
   - Heading hierarchy is consistent (h2 for sections, h3 for subsections)

**The review subagent should:**
- Produce a concise report listing issues by severity: CRITICAL (broken links/structure), WARNING (missing elements), INFO (minor suggestions)
- For CRITICAL and WARNING issues, fix them directly in the files
- Re-commit any fixes

**Report the review results** to the user: number of issues found and fixed, and any INFO-level suggestions they may want to consider.

### Phase 7: Final Summary

Present:
- Total files generated (chapters + index + theme.css)
- Published URL
- Chapter outline table (from Phase 2)
- Review results (issues found and fixed)
- Any concerns or suggestions for future improvement

## Key Principles

- **External CSS only.** HTML pages use `<link rel="stylesheet" href="theme.css">`. No inline `<style>` blocks in chapter pages. The theme is copied from `assets/themes/` in Phase 3.
- **Static sidebar in every file.** Each page has its own complete sidebar HTML — navigation works with JS disabled.
- **Subagent parallelism.** Use at least 3 subagents simultaneously for chapter generation. Independent batches run in parallel.
- **Research-informed content.** Phase 0 web search grounds the content in current, accurate information.
- **Self-review before shipping.** The review subagent catches broken links, missing elements, and inconsistencies before the user sees them.
- **Surgical commits.** Each commit touches only the files in the current batch. Never modify an already-committed chapter without reason.
- **The `gh` CLI must be authenticated.** If `gh auth status` fails, tell the user and pause.
