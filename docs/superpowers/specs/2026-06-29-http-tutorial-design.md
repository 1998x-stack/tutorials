# HTTP Protocol Tutorial Design Spec

**Date:** 2026-06-29
**Status:** Approved
**Repo:** `1998x-stack/HTTP-protocol-tutorial`

## Overview

Build a 28-chapter HTTP protocol tutorial following the same structure, style, and deployment pattern as the existing CAN-bus-tutorial. Static HTML files served via GitHub Pages from repo root.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Chapter count | 28 | Match CAN tutorial scale (30), cover HTTP comprehensively |
| Content scope | HTTP/1.1 focused, HTTP/2 & /3 closing | Parallels CAN + CANopen/J1939 structure |
| Teaching style | Conversational Chinese, teacher persona, practical tips | Same as CAN tutorial |
| Repo name | `HTTP-protocol-tutorial` | Consistent with `CAN-bus-tutorial` naming |
| Color theme | Blue-purple | Differentiates from CAN's green, fits web/HTTP aesthetic |
| Build approach | One HTML per chapter, hand-written content | Same as CAN tutorial, quality over speed |

## Chapter Outline

### Part 1: HTTP Fundamentals (Ch 1-7)
- 01 - HTTP Protocol Overview
- 02 - URL & Resource Addressing
- 03 - HTTP Message Structure
- 04 - HTTP Request Methods
- 05 - HTTP Status Codes
- 06 - HTTP Header Fields
- 07 - MIME Types & Content Negotiation

### Part 2: Core Mechanisms (Ch 8-14)
- 08 - Connection Management
- 09 - HTTP Caching
- 10 - Cookies & Sessions
- 11 - HTTP Authentication
- 12 - Cross-Origin Resource Sharing (CORS)
- 13 - Content Encoding & Transfer
- 14 - Redirects & URL Rewriting

### Part 3: Security (Ch 15-18)
- 15 - HTTPS Overview
- 16 - TLS Deep Dive
- 17 - Certificate Management
- 18 - HTTP Security Headers

### Part 4: Advanced HTTP/1.1 (Ch 19-22)
- 19 - Proxy & Gateway
- 20 - Load Balancing & Rate Limiting
- 21 - RESTful API Design
- 22 - HTTP Performance Optimization

### Part 5: Tools & Practice (Ch 23-25)
- 23 - HTTP Debugging Tools
- 24 - Web Server Configuration
- 25 - HTTP Packet Capture Practice

### Part 6: Next Generation Protocols (Ch 26-28)
- 26 - HTTP/2
- 27 - HTTP/3 (QUIC)
- 28 - HTTP Project Practice

## File Structure

```
HTTP-protocol-tutorial/
├── index.html          # Course directory (card grid, JS-rendered)
├── 01.html ~ 28.html   # Chapter pages
└── README.md           # Optional
```

## Page Layout

Each chapter page uses a two-column grid layout (identical to CAN tutorial):

- **Left sidebar (260px):** Full chapter list with current chapter highlighted via `.active` class. Static HTML — no JS dependency.
- **Main content:** Chapter title (h2), sections (h3), conversational Chinese prose, tables, code blocks (`<pre>`), and three callout types (`.tip` blue, `.warning` red, `.highlight` yellow).
- **Footer nav:** Previous chapter / Back to index / Next chapter links.

Index page uses a responsive card grid with chapter number, title, knowledge points, and a "start learning" link. Cards rendered by JS from a data array, with `<noscript>` fallback.

## Navigation Rules

- Chapter 1: "Previous" link uses `javascript:void(0)`
- Chapter 28: "Next" link uses `javascript:void(0)`
- All other chapters: real links to adjacent chapter files
- Sidebar updated per page with correct `.active` class

## Deployment

- GitHub Pages from repo root (`main` branch, `/` root)
- No build step; pure static HTML
- Same pattern as CAN-bus-tutorial (from `f8aab60` commit: "move html files to repo root for GitHub Pages")

## Build Order

1. `index.html` skeleton (populate with chapter data as chapters are finalized)
2. Ch 01-07 (Part 1: Fundamentals)
3. Ch 08-14 (Part 2: Core Mechanisms)
4. Ch 15-18 (Part 3: Security)
5. Ch 19-22 (Part 4: Advanced)
6. Ch 23-25 (Part 5: Tools & Practice)
7. Ch 26-28 (Part 6: Next Generation)
8. Finalize `index.html` with complete 28-chapter data
9. Create GitHub repo `1998x-stack/HTTP-protocol-tutorial`
10. Push all files, enable GitHub Pages
