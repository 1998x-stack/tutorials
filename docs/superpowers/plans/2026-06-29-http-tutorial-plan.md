# HTTP Protocol Tutorial Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 28-chapter HTTP protocol tutorial as static HTML pages, deploy via GitHub Pages on `1998x-stack/HTTP-protocol-tutorial`.

**Architecture:** Pure static HTML — no build step, no JS framework. Each chapter is a self-contained HTML file with shared CSS template, left sidebar navigation, and main content area. index.html uses minimal JS to render a card-grid course directory. Follows the exact layout and style conventions of the existing CAN-bus-tutorial.

**Tech Stack:** HTML5 + CSS3 + minimal vanilla JS (index.html card rendering). No dependencies, no frameworks, no build tools.

## Global Constraints

- All files live in repo root (GitHub Pages requirement for `/` serving)
- Chapter files named `01.html` through `28.html`
- Chinese language content, conversational teaching style with teacher persona
- Blue-purple color theme (HSL ~220-260 hue range) — differentiates from CAN's green
- Sidebar navigation is static HTML in every chapter (no JS dependency for nav)
- `index.html` uses JS to render course cards from data array, with `<noscript>` fallback
- Each chapter page must be fully self-contained (inline CSS, no external resources except system fonts)

## File Structure

```
HTTP-protocol-tutorial/
├── index.html          # Course directory — card grid, JS-rendered from data array
├── 01.html             # Ch 1: HTTP Protocol Overview
├── 02.html             # Ch 2: URL & Resource Addressing
├── ...                 # Ch 3-27
├── 28.html             # Ch 28: HTTP Project Practice
└── README.md           # Optional repo readme
```

**Key design decisions:**
- Each `.html` file is ~250-350 lines (inline CSS ~115 lines + sidebar ~35 lines + content ~100-200 lines)
- CSS is duplicated in every file (no shared stylesheet — keeps each file self-contained and GitHub Pages simple)
- Sidebar chapter list is static HTML repeated in every file with `.active` class toggled per-page

---

## Chapter Content Reference

Each chapter follows this content structure (actual teaching content written per chapter):

```
<h2>第X章：标题</h2>
<p>开场白（口语化，讲师口吻，1-2段引入话题）</p>
<h3>X.1 小节标题</h3>
<p>正文（口语化解释，穿插实战经验、类比、故事）</p>
<ul><li>要点列表</li></ul>
<table>...对比表格...</table>
<pre>代码/报文示例</pre>
<div class="tip">实战技巧（蓝色）</div>
<div class="warning">避坑指南（红色）</div>
<div class="highlight">核心总结（黄色）</div>
<h3>X.2 下一个小节...</h3>
...
<p>章节结束语，预告下一章</p>
```

## Tasks

### Task 1: Create project directory and base chapter template

**Files:**
- Create: `HTTP-protocol-tutorial/01.html`

**Produces:** Reusable HTML/CSS template for all 28 chapters. The blue-purple color theme, two-column layout with sidebar and main content.

- [ ] **Step 1: Create project directory**

```bash
mkdir -p /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
```

- [ ] **Step 2: Write 01.html as the base template with complete CSS**

Write `/workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/01.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.5, user-scalable=yes">
    <title>第1章 HTTP协议概述</title>
    <style>
        * { margin:0; padding:0; box-sizing:border-box; }
        body {
            background: linear-gradient(135deg, #eef2ff 0%, #dce4f8 100%);
            font-family: system-ui, -apple-system, 'Segoe UI', 'PingFang SC', Roboto, 'Helvetica Neue', sans-serif;
            padding: 20px;
        }
        .double-border {
            max-width: 1400px;
            margin: 0 auto;
            background: #ffffff;
            border: 2px solid #818cf8;
            border-radius: 24px;
            box-shadow: 0 20px 30px -15px rgba(0,0,0,0.2), inset 0 1px 0 0 #eef2ff;
            overflow: hidden;
        }
        .container {
            display: grid;
            grid-template-columns: 260px 1fr;
            gap: 0;
        }
        .sidebar {
            background: #f8faff;
            border-right: 2px solid #c7d2fe;
            padding: 24px 16px;
        }
        .sidebar h3 {
            font-size: 1.4rem;
            color: #4338ca;
            border-left: 5px solid #6366f1;
            padding-left: 15px;
            margin-bottom: 25px;
        }
        .chapter-list {
            list-style: none;
        }
        .chapter-list li {
            margin-bottom: 6px;
        }
        .chapter-list a {
            display: block;
            padding: 8px 12px;
            text-decoration: none;
            color: #334155;
            border-radius: 12px;
            transition: all 0.2s;
            font-weight: 500;
            font-size: 0.92rem;
        }
        .chapter-list a:hover {
            background: #e0e7ff;
            transform: translateX(3px);
        }
        .chapter-list .active {
            background: #6366f1;
            color: white;
            box-shadow: 0 2px 5px rgba(99,102,241,0.3);
        }
        .main-content {
            padding: 30px 35px;
            background: white;
        }
        .footer-nav {
            margin-top: 45px;
            padding-top: 20px;
            border-top: 2px solid #e0e7ff;
            display: flex;
            justify-content: space-between;
        }
        .footer-nav a {
            padding: 8px 24px;
            background: #f1f5f9;
            text-decoration: none;
            color: #3730a3;
            border-radius: 40px;
            font-weight: 500;
            transition: 0.2s;
        }
        .footer-nav a:hover {
            background: #e0e7ff;
            transform: translateY(-2px);
        }
        footer {
            text-align: center;
            margin-top: 30px;
            padding: 15px;
            color: #64748b;
            border-top: 1px dashed #cbd5e1;
            font-size: 0.9rem;
        }
        h2 { color: #312e81; border-left: 6px solid #818cf8; padding-left: 15px; margin: 20px 0 15px; font-size: 1.5rem; }
        h3 { color: #3730a3; margin: 20px 0 10px; font-size: 1.2rem; }
        h4 { color: #4338ca; margin: 15px 0 8px; font-size: 1.05rem; }
        p { line-height: 1.85; margin: 10px 0; color: #1e293b; }
        ul, ol { margin: 10px 0 10px 20px; line-height: 1.8; }
        li { color: #1e293b; margin-bottom: 4px; }
        pre {
            background: #1e1b4b; color: #e8e8f0; padding: 15px; border-radius: 12px;
            overflow-x: auto; font-family: 'JetBrains Mono', 'Fira Code', monospace; margin: 15px 0;
            font-size: 0.9rem; line-height: 1.5;
        }
        code { background: #eef2ff; padding: 2px 6px; border-radius: 4px; font-family: 'JetBrains Mono', monospace; font-size: 0.85em; }
        pre code { background: none; padding: 0; }
        table {
            width: 100%; border-collapse: collapse; margin: 15px 0; background: #fafafe;
        }
        th, td {
            border: 1px solid #c7d2fe; padding: 10px; text-align: left;
        }
        th { background: #eef2ff; color: #3730a3; font-weight: 600; }
        .tip, .warning, .highlight {
            padding: 12px 18px; border-radius: 16px; margin: 15px 0; line-height: 1.7;
        }
        .tip { background: #e0f2fe; border-left: 6px solid #0ea5e9; }
        .warning { background: #fee2e2; border-left: 6px solid #ef4444; }
        .highlight { background: #fef9c3; border: 1px solid #facc15; }
        strong { color: #1e1b4b; }
        @media (max-width: 768px) {
            .container { grid-template-columns: 1fr; }
            .sidebar { border-right: none; border-bottom: 2px solid #c7d2fe; }
            body { padding: 10px; }
            .main-content { padding: 20px; }
        }
    </style>
</head>
<body>
<div class="double-border">
<div class="container">
    <div class="sidebar">
        <h3>🌐 课程目录</h3>
        <ul class="chapter-list">
            <li><a href="01.html" class="active">第1章 HTTP协议概述</a></li>
<li><a href="02.html">第2章 URL与资源定位</a></li>
<li><a href="03.html">第3章 HTTP报文结构</a></li>
<li><a href="04.html">第4章 HTTP请求方法</a></li>
<li><a href="05.html">第5章 HTTP状态码</a></li>
<li><a href="06.html">第6章 HTTP头部字段</a></li>
<li><a href="07.html">第7章 MIME类型与内容协商</a></li>
<li><a href="08.html">第8章 连接管理</a></li>
<li><a href="09.html">第9章 HTTP缓存</a></li>
<li><a href="10.html">第10章 Cookie与会话</a></li>
<li><a href="11.html">第11章 HTTP认证</a></li>
<li><a href="12.html">第12章 跨域资源共享(CORS)</a></li>
<li><a href="13.html">第13章 内容编码与传输</a></li>
<li><a href="14.html">第14章 重定向与URL重写</a></li>
<li><a href="15.html">第15章 HTTPS概述</a></li>
<li><a href="16.html">第16章 TLS深入</a></li>
<li><a href="17.html">第17章 证书管理</a></li>
<li><a href="18.html">第18章 HTTP安全头部</a></li>
<li><a href="19.html">第19章 代理与网关</a></li>
<li><a href="20.html">第20章 负载均衡与限流</a></li>
<li><a href="21.html">第21章 RESTful API设计</a></li>
<li><a href="22.html">第22章 HTTP性能优化</a></li>
<li><a href="23.html">第23章 HTTP调试工具</a></li>
<li><a href="24.html">第24章 Web服务器配置</a></li>
<li><a href="25.html">第25章 HTTP抓包实战</a></li>
<li><a href="26.html">第26章 HTTP/2</a></li>
<li><a href="27.html">第27章 HTTP/3 (QUIC)</a></li>
<li><a href="28.html">第28章 HTTP项目实战</a></li>
        </ul>
    </div>
    <div class="main-content">
        <!-- CHAPTER CONTENT GOES HERE -->
        <div class="footer-nav">
            <a href="javascript:void(0)">← 上一章</a>
            <a href="index.html">📖 返回目录</a>
            <a href="02.html">下一章 →</a>
        </div>
    </div>
</div>
</div>
</body>
</html>
```

- [ ] **Step 3: Verify template renders correctly**

```bash
ls -la /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/01.html
```

Open in browser: `file:///workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/01.html`
Verify: Sidebar with 28 chapter links, blue-purple theme, responsive at 768px breakpoint.

- [ ] **Step 4: Commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git init
git add 01.html
git commit -m "feat: add base chapter template with blue-purple theme"
```

---

### Task 2: Write Chapter 01 content — HTTP Protocol Overview

**Files:**
- Modify: `HTTP-protocol-tutorial/01.html` (fill in `<!-- CHAPTER CONTENT GOES HERE -->`)

**Consumes:** Task 1 template (01.html)
**Produces:** Completed 01.html with full teaching content

- [ ] **Step 1: Fill chapter content into 01.html**

Replace `<!-- CHAPTER CONTENT GOES HERE -->` in 01.html with:

```html
<h2>第一章：HTTP协议概述</h2>

<p>大家好，我是你们的讲师。在Web开发这行摸爬滚打十几年，HTTP协议可以说是天天打交道的老朋友了。今天咱们就来聊聊这个看似简单、实则门道不少的协议。</p>

<h3>1.1 HTTP的发展历史</h3>

<p>说到HTTP，得从上世纪90年代初说起。那时候互联网刚刚起步，Tim Berners-Lee 在 CERN 搞出了万维网（World Wide Web），随之而来的就是 HTTP 0.9。你没看错，最早的版本号是0.9，连1.0都不到。</p>

<p>我给大家列几个关键节点：</p>

<ul>
  <li><strong>1991年</strong>：HTTP/0.9 诞生，只能请求HTML，没有头部，没有状态码。服务器返回HTML字符串，TCP连接就关闭了。简单是简单，但功能太弱了。</li>
  <li><strong>1996年</strong>：HTTP/1.0 发布（RFC 1945），引入了HTTP头部、状态码、Content-Type、POST方法。每个请求还是独立TCP连接，效率不高。</li>
  <li><strong>1997年</strong>：HTTP/1.1 发布（RFC 2068，后更新为RFC 2616），这是目前使用最广泛的版本。引入了持久连接（Keep-Alive）、管线化、分块传输、Host头（虚拟主机）、缓存控制等核心特性。</li>
  <li><strong>2015年</strong>：HTTP/2 发布（RFC 7540），基于Google的SPDY协议。多路复用、头部压缩、服务器推送，性能大幅提升。</li>
  <li><strong>2022年</strong>：HTTP/3 发布（RFC 9114），基于QUIC协议，用UDP代替TCP，彻底解决了队头阻塞问题。</li>
</ul>

<p>我个人习惯把HTTP的发展分为三个阶段：第一阶段是"能用"（0.9/1.0），第二阶段是"好用"（1.1），第三阶段是"够快"（2/3）。现在大多数网站还跑在HTTP/1.1上，但HTTP/2和HTTP/3的占比正在快速增长。</p>

<div class="highlight">
  <p><strong>核心要点</strong>：HTTP协议从诞生到现在30多年了，依然是互联网通信的基石。为什么它能活这么久？因为简单、可扩展、和语言/平台无关。我见过无数新技术想取代HTTP，最后都成了HTTP之上的应用层协议。</p>
</div>

<h3>1.2 客户端-服务器模型</h3>

<p>HTTP最核心的设计就是客户端-服务器模型。说白了就是：浏览器（客户端）发请求，Web服务器处理请求后返回响应。这个模型简单到你可能会觉得不值一提，但它的精妙之处在于：</p>

<ul>
  <li><strong>无状态</strong>：每个请求都是独立的，服务器不记得你上次请求了什么。这简化了服务器设计，但带来了Cookie、Session等补充机制。</li>
  <li><strong>请求-响应</strong>：一问一答，双方角色分明。客户端主动发起，服务器被动响应。</li>
  <li><strong>中间件友好</strong>：正因为请求和响应格式统一，代理、网关、CDN这些中间层才能无缝工作。</li>
</ul>

<p>你想想看，你去访问一个网页，浏览器会发多少个HTTP请求？看起来是一个页面，但背后可能要请求几十个甚至上百个资源（HTML、CSS、JS、图片、字体、API数据...）。每一个都是一个独立的HTTP请求。</p>

<div class="tip">
  <p><strong>实战技巧</strong>：打开浏览器DevTools的Network面板（F12），刷新任意一个网页，你就能直观看到所有HTTP请求。我刚入行时就喜欢干这事——看到一个网页"长什么样"不稀奇，能看到它"怎么加载的"才是真本事。</p>
</div>

<h3>1.3 HTTP的版本差异速览</h3>

<table>
  <tr><th>特性</th><th>HTTP/1.0</th><th>HTTP/1.1</th><th>HTTP/2</th><th>HTTP/3</th></tr>
  <tr><td>连接复用</td><td>否</td><td>是（Keep-Alive）</td><td>是（多路复用）</td><td>是（QUIC）</td></tr>
  <tr><td>头部压缩</td><td>无</td><td>无</td><td>HPACK</td><td>QPACK</td></tr>
  <tr><td>服务器推送</td><td>无</td><td>无</td><td>有</td><td>有</td></tr>
  <tr><td>传输层</td><td>TCP</td><td>TCP</td><td>TCP</td><td>UDP (QUIC)</td></tr>
  <tr><td>队头阻塞</td><td>有</td><td>有</td><td>部分解决</td><td>完全解决</td></tr>
  <tr><td>发布年份</td><td>1996</td><td>1997</td><td>2015</td><td>2022</td></tr>
</table>

<div class="warning">
  <p><strong>注意</strong>：HTTP/1.1虽然支持管线化（Pipelining），但实际上浏览器几乎都不开启。因为队头阻塞问题太严重——一个请求卡住了，后面全部排队等。我就吃过这个亏，早期用管线化做并发优化，结果生产环境经常超时，最后乖乖改回连接池方案。</p>
</div>

<h3>1.4 HTTP在整个网络栈中的位置</h3>

<p>HTTP是应用层协议，跑在TCP/IP之上。整个网络栈从上到下是：</p>

<ul>
  <li><strong>应用层</strong>：HTTP、FTP、DNS、WebSocket...</li>
  <li><strong>传输层</strong>：TCP（HTTP/1.1、HTTP/2）或 UDP（HTTP/3）</li>
  <li><strong>网络层</strong>：IP</li>
  <li><strong>链路层</strong>：Ethernet、WiFi</li>
</ul>

<p>理解这个分层很重要。比如，你在排查问题的时候，如果HTTP层一切正常但请求超时，那问题可能在TCP层（握手失败）或网络层（路由问题）。我建议你遇到问题从下往上排查，先ping通不通，再telnet端口开不开，最后才看HTTP报文。</p>

<p>好了，第一章的内容就到这里。我们建立了一个宏观的认知框架，下一章咱们开始深入HTTP的第一个核心概念——URL与资源定位。</p>
```

- [ ] **Step 2: Verify 01.html renders correctly**

Open `file:///workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/01.html` in browser.
Verify: Title "第一章：HTTP协议概述", sidebar Ch1 active highlighted, prev link disabled, next link points to 02.html.

- [ ] **Step 3: Commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 01.html
git commit -m "feat: complete Chapter 01 — HTTP protocol overview"
```

---

### Task 3: Write Chapters 02-04 (URL, Messages, Methods)

**Files:**
- Create: `HTTP-protocol-tutorial/02.html`, `03.html`, `04.html`

**Consumes:** Task 1 template (copy to each new file), Task 2 pattern for content writing

**Note for implementer:** Copy 01.html as starting point for each chapter, then:
1. Update `<title>`, sidebar `.active` class, footer nav links
2. Replace main content with chapter-specific teaching material
3. Each chapter is ~150-250 lines of content

- [ ] **Step 1: Write 02.html — URL & Resource Addressing**

Copy 01.html → 02.html. Update:
- `<title>` to "第2章 URL与资源定位"
- Sidebar: move `.active` from Ch1 to Ch2 link
- Footer nav: `← 上一章` → `01.html`, `下一章 →` → `03.html`

Content structure (write in conversational Chinese with teacher persona):
- **h2: 第二章：URL与资源定位**
- **2.1 URI、URL、URN的区别** — Explain URI as superset, URL as locator, URN as name. Practical examples.
- **2.2 URL的语法结构** — Break down `scheme://host:port/path?query#fragment` with table of each component, practical examples.
- **2.3 URL编码规则** — Percent-encoding, which characters need encoding, `encodeURIComponent` vs `encodeURI`, common pitfalls.
- **2.4 绝对URL与相对URL** — Base URL resolution, `../` traversal, when to use which.
- **2.5 URL长度限制** — Browser/server limits, GET vs POST, practical implications.

- [ ] **Step 2: Write 03.html — HTTP Message Structure**

Copy 01.html → 03.html. Update navigation as above (active Ch3, prev 02, next 04).

Content structure:
- **h2: 第三章：HTTP报文结构**
- **3.1 请求报文解剖** — Request line (method + URI + version), headers, blank line, body. Show raw HTTP request example in `<pre>`.
- **3.2 响应报文解剖** — Status line (version + status code + reason phrase), headers, body. Show raw HTTP response example.
- **3.3 HTTP头部的格式** — `Name: Value` format, multi-line headers, common header categories table.
- **3.4 消息体（Body）** — When body is present/absent, Content-Length vs Transfer-Encoding: chunked, MIME type relationship.
- **3.5 用curl查看原始报文** — `curl -v` examples with annotated output showing request and response.

- [ ] **Step 3: Write 04.html — HTTP Request Methods**

Copy 01.html → 04.html. Update navigation (active Ch4, prev 03, next 05).

Content structure:
- **h2: 第四章：HTTP请求方法**
- **4.1 方法概览** — Table of all 9 methods (GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS/TRACE/CONNECT), RFC, purpose.
- **4.2 安全性与幂等性** — Definitions, why they matter, table mapping each method's safety/idempotency.
- **4.3 GET详解** — Read-only, parameters in URL, caching implications, when NOT to use GET.
- **4.4 POST详解** — Creating resources, body content, CSRF concerns, POST vs PUT.
- **4.5 PUT、DELETE、PATCH的区别** — Full replace vs partial update, idempotency examples.
- **4.6 HEAD与OPTIONS的实用场景** — CORS preflight, resource existence check without download.

- [ ] **Step 4: Verify all three chapters render**

Open each file in browser. Verify sidebar navigation correct, colors consistent, tables/code blocks render.

- [ ] **Step 5: Commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 02.html 03.html 04.html
git commit -m "feat: add Chapters 02-04 — URL, Message Structure, Methods"
```

---

### Task 4: Write Chapters 05-07 (Status Codes, Headers, MIME Types)

**Files:**
- Create: `HTTP-protocol-tutorial/05.html`, `06.html`, `07.html`

**Approach:** Same pattern as Task 3 — copy 01.html template, update nav, write content.

- [ ] **Step 1: Write 05.html — HTTP Status Codes**

Content (write in Chinese conversational style):
- **5.1 状态码分类** — 1xx Informational, 2xx Success, 3xx Redirection, 4xx Client Error, 5xx Server Error. Table with class descriptions.
- **5.2 常用2xx** — 200 OK, 201 Created, 204 No Content. When to use each.
- **5.3 3xx重定向** — 301 vs 302 vs 307 vs 308, permanent vs temporary, method change behavior.
- **5.4 4xx客户端错误** — 400, 401 vs 403, 404, 405, 429 (rate limiting). Real-world debugging stories.
- **5.5 5xx服务端错误** — 500, 502, 503, 504. The difference between 502 and 504 (explained clearly).
- **5.6 自定义状态码的最佳实践** — Why you should stick to standard codes, how to use response body for detail.

- [ ] **Step 2: Write 06.html — HTTP Header Fields**

Content:
- **6.1 头部字段分类** — General headers, Request headers, Response headers, Entity headers. Table with examples.
- **6.2 常用通用头** — Cache-Control, Connection, Date, Transfer-Encoding.
- **6.3 常用请求头** — Host, User-Agent, Accept series, Authorization, Cookie, Referer.
- **6.4 常用响应头** — Server, Set-Cookie, Location, WWW-Authenticate, Access-Control-*.
- **6.5 自定义头部** — X- prefix deprecated, vendor-specific headers, when to use custom headers.
- **6.6 头部大小写与重复** — Headers are case-insensitive, duplicate headers (Set-Cookie is special), comma-joined values.

- [ ] **Step 3: Write 07.html — MIME Types & Content Negotiation**

Content:
- **7.1 什么是MIME类型** — History, structure (type/subtype), IANA registry.
- **7.2 常见MIME类型** — text/html, application/json, multipart/form-data, image/*, etc. Table with use cases.
- **7.3 Content-Type头** — charset parameter, boundary for multipart, sniffing risks.
- **7.4 内容协商机制** — Server-driven (Accept/Accept-Language/Accept-Encoding), client-driven (406 response).
- **7.5 Accept系列头实战** — Quality values (q=0.9), wildcards, common negotiation patterns.
- **7.6 字符编码** — UTF-8 dominance, charset declaration, practical debugging of garbled text.

- [ ] **Step 4: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 05.html 06.html 07.html
git commit -m "feat: add Chapters 05-07 — Status Codes, Headers, MIME Types"
```

---

### Task 5: Write Chapters 08-11 (Connection, Caching, Cookies, Auth)

**Files:**
- Create: `HTTP-protocol-tutorial/08.html`, `09.html`, `10.html`, `11.html`

- [ ] **Step 1: Write 08.html — Connection Management**

Content:
- **8.1 短连接与长连接** — HTTP/1.0 one-request-per-connection vs HTTP/1.1 persistent connections.
- **8.2 Keep-Alive机制** — Connection header, timeout, max requests, how browsers manage connection pool (6-per-host limit story).
- **8.3 HTTP管线化** — Theory vs reality, why browsers disabled it, head-of-line blocking explained.
- **8.4 TCP连接开销** — Three-way handshake, slow start, TLS handshake on top. Real latency numbers.
- **8.5 连接池实战** — How browsers pool connections, best practices for server-side keep-alive config in Nginx/Apache.

- [ ] **Step 2: Write 09.html — HTTP Caching**

Content:
- **9.1 为什么需要缓存** — Latency, bandwidth, server load. Numbers: cache hit vs miss timing.
- **9.2 Cache-Control详解** — max-age, public/private, no-cache, no-store, must-revalidate, immutable. Each with practical example.
- **9.3 ETag与If-None-Match** — Strong vs weak ETags, validation flow, generation strategies (hash, timestamp, version).
- **9.4 Last-Modified与If-Modified-Since** — Timestamp-based validation, 304 Not Modified, clock skew issues.
- **9.5 缓存层级** — Browser cache, service worker, CDN, reverse proxy. Freshness and validation flow.
- **9.6 缓存策略最佳实践** — Static assets (hash filenames + immutable), HTML (no-cache), API responses.

- [ ] **Step 3: Write 10.html — Cookies & Sessions**

Content:
- **10.1 Cookie的由来** — HTTP statelessness problem, Netscape's invention, how Set-Cookie and Cookie headers work.
- **10.2 Cookie属性详解** — Domain, Path, Expires/Max-Age, HttpOnly, Secure, SameSite (Strict/Lax/None).
- **10.3 Session机制** — Server-side session storage, session ID in cookie, common pitfalls (sticky sessions, session fixation).
- **10.4 第三方Cookie** — Cross-domain tracking, privacy implications, browser phase-out timeline.
- **10.5 Cookie安全** — XSS theft (HttpOnly defense), CSRF, session hijacking, signing vs encryption.

- [ ] **Step 4: Write 11.html — HTTP Authentication**

Content:
- **11.1 认证 vs 授权** — Distinction, where HTTP fits.
- **11.2 Basic认证** — Base64 encoding, security flaws (no encryption), why HTTPS is mandatory.
- **11.3 Digest认证** — Challenge-response, better than Basic but still rare, why it lost.
- **11.4 Bearer Token** — JWT structure, Authorization header, OAuth2 flow overview, token storage.
- **11.5 API Key认证** — X-API-Key header pattern, when appropriate, rate limiting integration.
- **11.6 Session Cookie vs Token** — Trade-offs table, which to use when.

- [ ] **Step 5: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 08.html 09.html 10.html 11.html
git commit -m "feat: add Chapters 08-11 — Connection, Caching, Cookies, Auth"
```

---

### Task 6: Write Chapters 12-14 (CORS, Content Encoding, Redirects)

**Files:**
- Create: `HTTP-protocol-tutorial/12.html`, `13.html`, `14.html`

- [ ] **Step 1: Write 12.html — CORS**

Content:
- **12.1 同源策略** — Origin definition (scheme+host+port), why it exists, what it blocks.
- **12.2 CORS简介** — How servers relax same-origin, Access-Control-Allow-Origin, simple requests.
- **12.3 预检请求（Preflight）** — OPTIONS request, conditions that trigger preflight, Access-Control-Allow-Methods/Headers.
- **12.4 携带凭证** — Access-Control-Allow-Credentials, withCredentials/XHR, cookie behavior.
- **12.5 CORS常见问题** — Wildcard origin + credentials, caching preflight, debugging CORS errors in DevTools.

- [ ] **Step 2: Write 13.html — Content Encoding & Transfer**

Content:
- **13.1 内容编码（Content-Encoding）** — gzip, brotli, deflate. Compression ratios, when to compress.
- **13.2 传输编码（Transfer-Encoding）** — chunked transfer, how it works, Content-Length vs chunked.
- **13.3 范围请求（Range Requests）** — Accept-Ranges, Range/Content-Range headers, resumable downloads, video seeking.
- **13.4 断点续传实战** — Multi-part range responses, how download managers work.

- [ ] **Step 3: Write 14.html — Redirects & URL Rewriting**

Content:
- **14.1 重定向的类型** — 301/302/303/307/308 comparison table, when each is appropriate.
- **14.2 客户端重定向** — meta refresh, JavaScript redirect, when to use server-side vs client-side.
- **14.3 URL重写** — Nginx rewrite rules, Apache mod_rewrite, SPA fallback to index.html pattern.
- **14.4 重定向循环** — How they happen, debugging with curl -L, browser detection and breaking.

- [ ] **Step 4: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 12.html 13.html 14.html
git commit -m "feat: add Chapters 12-14 — CORS, Content Encoding, Redirects"
```

---

### Task 7: Write Chapters 15-18 (Security — HTTPS, TLS, Certs, Security Headers)

**Files:**
- Create: `HTTP-protocol-tutorial/15.html`, `16.html`, `17.html`, `18.html`

- [ ] **Step 1: Write 15.html — HTTPS Overview**

Content:
- **15.1 HTTP明文传输的风险** — Eavesdropping, tampering, MITM. Real-world attack scenarios.
- **15.2 加密基础** — Symmetric encryption (AES), asymmetric encryption (RSA/ECDSA), why HTTPS uses both.
- **15.3 TLS握手过程** — Step-by-step walkthrough: ClientHello, ServerHello, certificate verification, key exchange, finished. Diagram in text.
- **15.4 证书信任链** — Root CA → Intermediate CA → Server cert. How browsers verify.
- **15.5 HTTPS的误解** — "HTTPS is slow" (modern hardware), "only login pages need it" (session hijacking).

- [ ] **Step 2: Write 16.html — TLS Deep Dive**

Content:
- **16.1 TLS版本演进** — 1.0 (deprecated), 1.1, 1.2 (current), 1.3 (latest). What changed and why.
- **16.2 密码套件** — Cipher suite structure: key exchange, authentication, encryption, MAC. Modern recommended suites.
- **16.3 前向保密（Forward Secrecy）** — ECDHE key exchange, why it matters, which configurations support it.
- **16.4 ALPN** — Application-Layer Protocol Negotiation, how HTTP/2 is negotiated during TLS handshake.
- **16.5 TLS性能优化** — Session resumption, OCSP stapling, TLS 1.3 1-RTT vs 0-RTT.

- [ ] **Step 3: Write 17.html — Certificate Management**

Content:
- **17.1 证书类型** — DV, OV, EV certificates. What each validates.
- **17.2 CA签发流程** — CSR generation, domain validation methods (DNS/HTTP/email), issuance.
- **17.3 Let's Encrypt** — ACME protocol, certbot, auto-renewal, rate limits.
- **17.4 自签名证书** — When to use (internal/testing), trust issues, adding to trust store.
- **17.5 证书过期** — Monitoring, automated renewal, horror stories of production outages.

- [ ] **Step 4: Write 18.html — HTTP Security Headers**

Content:
- **18.1 HSTS** — Strict-Transport-Security, max-age, includeSubDomains, preload list.
- **18.2 CSP** — Content-Security-Policy, directives (script-src, style-src, img-src), nonce/hash, report-only mode.
- **18.3 X-Frame-Options** — Clickjacking defense, DENY/SAMEORIGIN, why CSP frame-ancestors is better.
- **18.4 X-Content-Type-Options** — nosniff, MIME sniffing attacks.
- **18.5 Referrer-Policy** — Controlling referrer leakage, different policy values.
- **18.6 Permissions-Policy** — Feature/API access control for embedded contexts.

- [ ] **Step 5: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 15.html 16.html 17.html 18.html
git commit -m "feat: add Chapters 15-18 — HTTPS, TLS, Certificates, Security Headers"
```

---

### Task 8: Write Chapters 19-22 (Proxy, Load Balancing, REST, Performance)

**Files:**
- Create: `HTTP-protocol-tutorial/19.html`, `20.html`, `21.html`, `22.html`

- [ ] **Step 1: Write 19.html — Proxy & Gateway**

Content:
- **19.1 正向代理** — Client-side proxy, use cases (bypass restrictions, anonymity, caching).
- **19.2 反向代理** — Server-side proxy, Nginx as reverse proxy, benefits (SSL termination, caching, routing).
- **19.3 透明代理** — Interception proxy, CDN edge nodes, when you don't know you're behind one.
- **19.4 X-Forwarded-For链** — How IP forwarding works through proxy chain, trusting proxy headers.
- **19.5 CDN原理** — Edge caching, origin pull, cache purge/invalidation, CDN selection by DNS.

- [ ] **Step 2: Write 20.html — Load Balancing & Rate Limiting**

Content:
- **20.1 负载均衡算法** — Round-robin, least connections, IP hash, weighted. Pros/cons of each.
- **20.2 Nginx upstream配置** — upstream block, server directives, backup servers, health checks.
- **20.3 会话保持（Sticky Sessions）** — When needed, implementation methods, downsides.
- **20.4 限流策略** — Rate limiting (burst vs sustained), token bucket, Nginx limit_req_zone, 429 responses.
- **20.5 健康检查** — Active vs passive, failure thresholds, graceful degradation.

- [ ] **Step 3: Write 21.html — RESTful API Design**

Content:
- **21.1 REST约束** — Client-server, stateless, cacheable, uniform interface, layered system, code-on-demand.
- **21.2 资源建模** — Nouns vs verbs in URLs, collections vs items, nested resources, when to flatten.
- **21.3 HTTP方法映射** — GET=read, POST=create, PUT=replace, PATCH=partial update, DELETE=remove.
- **21.4 状态码的正确使用** — Real-world examples of good and bad status code usage.
- **21.5 API版本控制** — URL versioning vs header versioning vs content negotiation.
- **21.6 HATEOAS** — The most ignored REST constraint, practical examples, when it's worth it.

- [ ] **Step 4: Write 22.html — HTTP Performance Optimization**

Content:
- **22.1 请求数量优化** — CSS sprites, icon fonts → SVG, JS/CSS bundling, HTTP/2 changes the rules.
- **22.2 资源体积优化** — Minification, compression, image optimization (WebP/AVIF), tree shaking.
- **22.3 加载策略** — async vs defer for scripts, critical CSS inline, lazy loading images/iframes.
- **22.4 资源提示** — preload, prefetch, preconnect, dns-prefetch. Practical prioritization.
- **22.5 HTTP缓存策略** — Hash-based filenames for immutable assets, cache-control tuning.
- **22.6 HTTP/2与HTTP/1.1优化策略的差异** — Why bundling is harmful under HTTP/2, individual file serving.

- [ ] **Step 5: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 19.html 20.html 21.html 22.html
git commit -m "feat: add Chapters 19-22 — Proxy, Load Balancing, REST, Performance"
```

---

### Task 9: Write Chapters 23-25 (Tools, Server Config, Packet Capture)

**Files:**
- Create: `HTTP-protocol-tutorial/23.html`, `24.html`, `25.html`

- [ ] **Step 1: Write 23.html — HTTP Debugging Tools**

Content:
- **23.1 curl命令实战** — Common flags (-v, -I, -X, -H, -d, -L, -k), piping to jq, scripting with curl.
- **23.2 浏览器DevTools Network面板** — Filter, waterfall, timing breakdown (DNS→TCP→TLS→Request→Response), copy as curl.
- **23.3 Postman/Insomnia** — Collections, environments, pre-request scripts, test assertions.
- **23.4 Wireshark HTTP过滤** — http filter, follow HTTP stream, TLS decryption with SSLKEYLOGFILE.

- [ ] **Step 2: Write 24.html — Web Server Configuration**

Content:
- **24.1 Nginx基础配置** — server blocks, listen directives, root/location, access_log/error_log.
- **24.2 虚拟主机** — Name-based vs IP-based, server_name matching, default_server.
- **24.3 HTTPS配置** — ssl_certificate, ssl_protocols, ssl_ciphers, HTTP→HTTPS redirect.
- **24.4 静态文件服务优化** — sendfile, tcp_nopush, gzip_static, expires headers.
- **24.5 日志分析** — Log format variables, GoAccess, finding slow requests/bottlenecks.

- [ ] **Step 3: Write 25.html — HTTP Packet Capture Practice**

Content:
- **25.1 抓包环境搭建** — Wireshark setup, SSLKEYLOGFILE for TLS decryption, tcpdump basics.
- **25.2 一次完整HTTP请求的抓包分析** — Annotated packet capture: DNS lookup, TCP handshake, TLS handshake, HTTP request, response, connection close.
- **25.3 时序图解读** — Waterfall diagram, identifying bottlenecks (long DNS, slow TLS negotiation, slow server processing).
- **25.4 生产环境抓包注意事项** — Permission, sensitive data, sampling in high-traffic environments.

- [ ] **Step 4: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 23.html 24.html 25.html
git commit -m "feat: add Chapters 23-25 — Debugging Tools, Server Config, Packet Capture"
```

---

### Task 10: Write Chapters 26-28 (HTTP/2, HTTP/3, Project Practice)

**Files:**
- Create: `HTTP-protocol-tutorial/26.html`, `27.html`, `28.html`

- [ ] **Step 1: Write 26.html — HTTP/2**

Content:
- **26.1 HTTP/1.1的局限性** — Head-of-line blocking, header redundancy, connection limits, why HTTP/2 was needed.
- **26.2 二进制帧层** — Frame format, stream multiplexing, how multiple requests share one TCP connection.
- **26.3 HPACK头部压缩** — Static/dynamic tables, Huffman coding, dramatic header size reduction.
- **26.4 服务器推送（Server Push）** — How it works, when to use, push promise, browser cache digests.
- **26.5 流优先级与流量控制** — Stream dependencies, weight, how browsers set priorities.

- [ ] **Step 2: Write 27.html — HTTP/3 (QUIC)**

Content:
- **27.1 为什么需要HTTP/3** — TCP head-of-line blocking still exists in HTTP/2, packet loss impact.
- **27.2 QUIC协议基础** — UDP-based, built-in TLS 1.3, connection IDs, no 4-tuple binding.
- **27.3 0-RTT连接建立** — How it works, replay safety concerns, when 0-RTT is safe.
- **27.4 连接迁移** — Switching networks (WiFi→Cellular) without reconnection, connection ID routing.
- **27.5 QPACK** — Header compression redesigned for QUIC, avoiding HPACK's HOL blocking.
- **27.6 HTTP/3部署现状** — Browser support, server support (Nginx, Caddy, Cloudflare), adoption metrics.

- [ ] **Step 3: Write 28.html — HTTP Project Practice**

Content:
- **28.1 项目目标** — Build a complete HTTP service: Nginx reverse proxy + backend + HTTPS.
- **28.2 环境准备** — Let's Encrypt cert, Nginx config, backend setup (any language).
- **28.3 Nginx配置实战** — Full config file with SSL, gzip, caching, security headers.
- **28.4 验证与测试** — curl tests, browser verification, SSL Labs test, security headers check.
- **28.5 性能基准测试** — Apache Bench / wrk, interpreting results, optimization iterations.
- **28.6 回顾与展望** — What you've learned end-to-end, where to go next (WebSocket, gRPC, GraphQL).

Footer nav: next link `javascript:void(0)` (last chapter).

- [ ] **Step 4: Verify and commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add 26.html 27.html 28.html
git commit -m "feat: add Chapters 26-28 — HTTP/2, HTTP/3, Project Practice"
```

---

### Task 11: Build index.html — Course Directory Page

**Files:**
- Create: `HTTP-protocol-tutorial/index.html`

**Consumes:** Complete chapter list from all prior tasks
**Produces:** Course directory page with 28 card grid, blue-purple theme

- [ ] **Step 1: Write index.html**

Write `/workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/index.html` using the CAN-bus-tutorial index.html as structural template, adapted with:
- Blue-purple color scheme matching chapter pages (gradients, card colors, link colors)
- HTTP emoji 🌐 in header instead of 🚌
- Title: "HTTP协议从入门到精通实战 · 课程目录"
- JS courses array with all 28 chapters (num, title, topics array, file)

The JS data array:

```javascript
const courses = [
    { num: '01', title: 'HTTP协议概述', topics: ['发展历史', '客户端-服务器模型', '版本差异速览'], file: '01.html' },
    { num: '02', title: 'URL与资源定位', topics: ['URI/URL/URN', 'URL语法结构', 'URL编码规则'], file: '02.html' },
    { num: '03', title: 'HTTP报文结构', topics: ['请求报文解剖', '响应报文解剖', '头部/消息体'], file: '03.html' },
    { num: '04', title: 'HTTP请求方法', topics: ['GET/POST/PUT/DELETE', '安全性与幂等性', 'HEAD/OPTIONS'], file: '04.html' },
    { num: '05', title: 'HTTP状态码', topics: ['1xx~5xx分类', '常用状态码详解', '状态码选择实践'], file: '05.html' },
    { num: '06', title: 'HTTP头部字段', topics: ['通用头/请求头/响应头', '实体头/自定义头', '大小写与重复'], file: '06.html' },
    { num: '07', title: 'MIME类型与内容协商', topics: ['MIME结构', 'Content-Type', 'Accept系列头'], file: '07.html' },
    { num: '08', title: '连接管理', topics: ['短连接/长连接', 'Keep-Alive', '管线化/连接池'], file: '08.html' },
    { num: '09', title: 'HTTP缓存', topics: ['Cache-Control', 'ETag/Last-Modified', '缓存层级策略'], file: '09.html' },
    { num: '10', title: 'Cookie与会话', topics: ['Set-Cookie/Cookie', 'Session机制', 'SameSite/HttpOnly'], file: '10.html' },
    { num: '11', title: 'HTTP认证', topics: ['Basic/Digest/Bearer', 'OAuth2流程', 'JWT结构'], file: '11.html' },
    { num: '12', title: '跨域资源共享(CORS)', topics: ['同源策略', '预检请求', 'Access-Control头'], file: '12.html' },
    { num: '13', title: '内容编码与传输', topics: ['gzip/brotli压缩', '分块传输', '范围请求/断点续传'], file: '13.html' },
    { num: '14', title: '重定向与URL重写', topics: ['301/302/307/308', 'Nginx rewrite', '重定向循环'], file: '14.html' },
    { num: '15', title: 'HTTPS概述', topics: ['对称/非对称加密', 'TLS握手过程', '证书信任链'], file: '15.html' },
    { num: '16', title: 'TLS深入', topics: ['版本演进', '密码套件', '前向保密/ALPN'], file: '16.html' },
    { num: '17', title: '证书管理', topics: ['DV/OV/EV证书', 'Let\'s Encrypt', '自签名证书'], file: '17.html' },
    { num: '18', title: 'HTTP安全头部', topics: ['HSTS/CSP', 'X-Frame-Options', 'Referrer-Policy'], file: '18.html' },
    { num: '19', title: '代理与网关', topics: ['正向/反向代理', 'X-Forwarded-For', 'CDN原理'], file: '19.html' },
    { num: '20', title: '负载均衡与限流', topics: ['负载均衡算法', 'Nginx upstream', '限流策略'], file: '20.html' },
    { num: '21', title: 'RESTful API设计', topics: ['REST约束', '资源建模', '状态码/HATEOAS'], file: '21.html' },
    { num: '22', title: 'HTTP性能优化', topics: ['请求/体积优化', '资源提示', 'HTTP/2优化差异'], file: '22.html' },
    { num: '23', title: 'HTTP调试工具', topics: ['curl实战', 'DevTools Network', 'Wireshark抓包'], file: '23.html' },
    { num: '24', title: 'Web服务器配置', topics: ['Nginx基础', 'HTTPS配置', '日志分析'], file: '24.html' },
    { num: '25', title: 'HTTP抓包实战', topics: ['抓包环境搭建', '请求分析', '时序图解读'], file: '25.html' },
    { num: '26', title: 'HTTP/2', topics: ['二进制帧层', 'HPACK头部压缩', '服务器推送'], file: '26.html' },
    { num: '27', title: 'HTTP/3 (QUIC)', topics: ['UDP/QUIC基础', '0-RTT连接', '连接迁移'], file: '27.html' },
    { num: '28', title: 'HTTP项目实战', topics: ['Nginx+HTTPS搭建', '安全头部验证', '性能基准测试'], file: '28.html' }
];
```

CSS adaptations from CAN's green theme to HTTP's blue-purple theme:
- Body gradient: `#eef2ff` / `#dce4f8` (blue-tinted)
- Card border/accent: indigo tones
- Header h1 gradient: `#3730a3` / `#6366f1`
- Chapter-num colors: cycle through indigo, violet, sky, slate, rose, amber
- Rainbow bar: `#818cf8, #a78bfa, #6366f1, #60a5fa`
- Footer nav links: indigo-based
- `.tip` / `.warning` / `.highlight` same as chapter pages

Full index.html should be ~400 lines (CSS + HTML + JS). Copy the CAN-bus-tutorial index.html as a starting point and adapt all colors, data, and emojis.

- [ ] **Step 2: Verify index.html renders**

Open in browser. Verify: 28 cards in responsive grid, each with chapter number, title, topics, "开始学习" link pointing to correct html file.

- [ ] **Step 3: Commit**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add index.html
git commit -m "feat: add course directory index page with 28-chapter card grid"
```

---

### Task 12: GitHub repo setup and GitHub Pages deployment

**Files:**
- Create: `HTTP-protocol-tutorial/README.md` (optional)
- Git remote: `git@github.com:1998x-stack/HTTP-protocol-tutorial.git`

- [ ] **Step 1: Create README.md**

Write `/workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial/README.md`:

```markdown
# HTTP协议从入门到精通实战

28章完整HTTP协议教程，涵盖HTTP/1.1、HTTPS、HTTP/2、HTTP/3。

## 课程目录

### 第一部分：HTTP基础 (第1-7章)
- 01 HTTP协议概述
- 02 URL与资源定位
- 03 HTTP报文结构
- 04 HTTP请求方法
- 05 HTTP状态码
- 06 HTTP头部字段
- 07 MIME类型与内容协商

### 第二部分：核心机制 (第8-14章)
- 08 连接管理
- 09 HTTP缓存
- 10 Cookie与会话
- 11 HTTP认证
- 12 跨域资源共享(CORS)
- 13 内容编码与传输
- 14 重定向与URL重写

### 第三部分：安全 (第15-18章)
- 15 HTTPS概述
- 16 TLS深入
- 17 证书管理
- 18 HTTP安全头部

### 第四部分：高级HTTP/1.1 (第19-22章)
- 19 代理与网关
- 20 负载均衡与限流
- 21 RESTful API设计
- 22 HTTP性能优化

### 第五部分：工具与实践 (第23-25章)
- 23 HTTP调试工具
- 24 Web服务器配置
- 25 HTTP抓包实战

### 第六部分：新一代协议 (第26-28章)
- 26 HTTP/2
- 27 HTTP/3 (QUIC)
- 28 HTTP项目实战

## 在线访问

GitHub Pages: https://1998x-stack.github.io/HTTP-protocol-tutorial/
```

- [ ] **Step 2: Create GitHub repo and push**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
gh repo create 1998x-stack/HTTP-protocol-tutorial --public --source=. --remote=origin --push
```

Note: This requires `gh` CLI authenticated with `1998x-stack` org permissions. If `gh` is not authenticated or the repo already exists, use:
```bash
git remote add origin git@github.com:1998x-stack/HTTP-protocol-tutorial.git
git branch -M main
git push -u origin main
```

- [ ] **Step 3: Enable GitHub Pages**

```bash
gh api repos/1998x-stack/HTTP-protocol-tutorial/pages \
  -X POST \
  -f "source[branch]=main" \
  -f "source[path]=/"
```

Or manually: Go to repo Settings → Pages → Source: "Deploy from a branch" → Branch: main, folder: / (root) → Save.

- [ ] **Step 4: Verify deployment**

Wait ~30 seconds, then verify:
```bash
curl -sI https://1998x-stack.github.io/HTTP-protocol-tutorial/ | head -5
```
Expected: `HTTP/2 200` and valid HTML content.

Also verify a chapter page:
```bash
curl -sI https://1998x-stack.github.io/HTTP-protocol-tutorial/01.html | head -5
```

- [ ] **Step 5: Commit README**

```bash
cd /workspace/data/xieming/other-codes/tutorials/HTTP-protocol-tutorial
git add README.md
git commit -m "docs: add README with course outline and GitHub Pages link"
git push
```

---

## Execution Summary

| Task | Files | Description |
|------|-------|-------------|
| 1 | `01.html` | Base template with blue-purple CSS + sidebar |
| 2 | `01.html` | Chapter 01 content: HTTP overview |
| 3 | `02-04.html` | URL, Message Structure, Methods |
| 4 | `05-07.html` | Status Codes, Headers, MIME Types |
| 5 | `08-11.html` | Connection, Caching, Cookies, Auth |
| 6 | `12-14.html` | CORS, Content Encoding, Redirects |
| 7 | `15-18.html` | HTTPS, TLS, Certificates, Security Headers |
| 8 | `19-22.html` | Proxy, Load Balancing, REST, Performance |
| 9 | `23-25.html` | Debugging Tools, Server Config, Packet Capture |
| 10 | `26-28.html` | HTTP/2, HTTP/3, Project Practice |
| 11 | `index.html` | Course directory card grid |
| 12 | Repo + Pages | GitHub repo creation, push, Pages deploy |

**Total:** 29 HTML files (28 chapters + index) + README, 12 tasks, 6 parts.
