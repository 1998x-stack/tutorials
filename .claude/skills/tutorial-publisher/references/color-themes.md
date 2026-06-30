# Color Theme Quick Reference

Each theme is a standalone CSS file in `assets/themes/`. Copy the chosen file into the tutorial project directory as `theme.css`.

## Theme Selection Guide

| Domain | CSS File | Hue Range | Example Topics |
|--------|----------|-----------|----------------|
| Protocol/Network | `blue-purple.css` | HSL 230-250 | HTTP, TCP/IP, WebSocket, DNS, gRPC, GraphQL, MQTT |
| Hardware/Embedded | `green.css` | HSL 120-150 | CAN bus, I2C, SPI, UART, GPIO, PCB design, ARM |
| Language/Framework | `warm.css` | HSL 10-40 | Python, JavaScript, React, Rust, Go, Vue, TypeScript |
| Data/AI | `violet-teal.css` | HSL 260-290 | SQL, Machine Learning, transformers, NumPy, Pandas, Spark |
| Systems/Infra | `slate-blue.css` | HSL 200-220 | Docker, Kubernetes, Linux, Nginx, CI/CD, Terraform |

Each CSS file contains the complete set of styles: layout, typography, code blocks, tables, callout boxes, and responsive breakpoints. HTML pages need only `<link rel="stylesheet" href="theme.css">` — no inline `<style>` blocks.

## Key Color Values by Theme

### Blue-Purple (`blue-purple.css`)
- Body gradient: `#eef2ff → #dce4f8`
- Sidebar: `#f8faff`, border `#c7d2fe`
- Active nav: `#6366f1`
- h2: `#312e81`, border `#818cf8`
- Code bg: `#1e1b4b`

### Green (`green.css`)
- Body gradient: `#f9fcf5 → #e8f3e8`
- Sidebar: `#fefcf5`, border `#fdebb3`
- Active nav: `#3b82f6`
- h2: `#1e3a8a`, border `#f59e0b`
- Code bg: `#2d2f36`

### Warm (`warm.css`)
- Body gradient: `#fff7ed → #ffe4d0`
- Sidebar: `#fffbf5`, border `#fed7aa`
- Active nav: `#ea580c`
- h2: `#7c2d12`, border `#f97316`
- Code bg: `#2d1f12`

### Violet-Teal (`violet-teal.css`)
- Body gradient: `#f5f3ff → #e8f5f0`
- Sidebar: `#faf8ff`, border `#c4b5fd`
- Active nav: `#7c3aed`
- h2: `#4c1d95`, border `#8b5cf6`
- Code bg: `#1e1035`

### Slate-Blue (`slate-blue.css`)
- Body gradient: `#f0f4f8 → #dce4ec`
- Sidebar: `#f8fafc`, border `#cbd5e1`
- Active nav: `#475569`
- h2: `#0f172a`, border `#3b82f6`
- Code bg: `#0f172a`
