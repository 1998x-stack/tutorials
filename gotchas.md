# Gotchas — Common Pitfalls & Hard-Won Lessons

A curated collection of real-world mistakes, debugging war stories, and lessons learned the hard way across GPU programming, systems, ML, and protocols.

---

## CUDA / GPU Programming

### Memory Access

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **AoS instead of SoA** | 3% of theoretical bandwidth. Each thread loads a struct, warp accesses scattered memory. | Convert Array-of-Structs to Struct-of-Arrays. Thread `i` should load element `i`. |
| **Non-coalesced access** | 45 GB/s on a 500 GB/s device. Strided access = 32 transactions per warp instead of 1. | Ensure adjacent threads access adjacent addresses. Check with Nsight Compute memory workload. |
| **Forgetting boundary checks** | Random crashes or silent wrong results on non-multiple-of-block-size data. | Always guard: `if (idx < N) { ... }` |
| **Uninitialized shared memory** | NaN values appearing randomly in output. | Initialize shared memory before use, or use `compute-sanitizer` to catch. |

### Synchronization

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **`__syncthreads()` inside conditional** | Kernel hangs at certain block sizes. Undefined behavior. | Move barrier outside any branch that not all threads take. |
| **Default stream serializes everything** | Expected 3x overlap from 3 streams, got 1x. Timeline shows everything sequential. | Use explicit `cudaStream_t` streams. The default stream waits for ALL other streams. |
| **Missing `cudaDeviceSynchronize` before timing** | Kernel appears 10x faster than physically possible. | Sync before stopping timer — kernel launches are async. |

### Tooling

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Optimizing without profiling first** | 3 weeks optimizing a kernel that was memory-bound all along. | Run Nsight Compute first. Check roofline — are you compute-bound or memory-bound? |
| **Measuring cold runs** | "Optimized" kernel appears 2x faster. Then next run is same as before. | Always do warmup iterations. GPU clock ramps up. |
| **Not using compute-sanitizer** | 3-day debugging session for NaN values. | `compute-sanitizer --tool memcheck` catches out-of-bounds instantly. |

### Streams & Concurrency

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Forgetting pinned memory for async** | `cudaMemcpyAsync` behaves synchronously. | Use `cudaMallocHost` — regular malloc'd memory can't do async transfers. |
| **Kernel launch overhead for tiny kernels** | CPU spends more time launching than GPU spends computing. | Fuse small kernels. Use CUDA Graphs for repetitive launches. |

### Libraries

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Hand-rolling what a library does** | 2 weeks writing a reduction kernel, 3x slower than `cub::DeviceReduce` (5 lines). | Check cuBLAS, cuFFT, Thrust, CUB, CUTLASS first. "Use libraries before writing kernels." |

---

## HTTP / Networking

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Keep-Alive timeout mismatch** | Client reuses connection, server already closed it. `Connection reset` errors. | Server keep-alive timeout must exceed client's. Add retry logic for idempotent requests. |
| **CORS preflight caching** | Every request sends OPTIONS. Latency doubled. | Set `Access-Control-Max-Age` to cache preflight results. |
| **HTTP/2 multiplexing + slow endpoint** | One slow stream blocks ALL streams (head-of-line blocking at TCP level). | Separate slow endpoints to different connections, or use HTTP/3 (QUIC). |

---

## Machine Learning

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Normalization mismatch** | Model works in training, fails in production. | Save normalization parameters (mean/std) with the model. Apply identically at inference. |
| **Data leakage through preprocessing** | Validation accuracy suspiciously high. | Fit scalers/encoders on training split only, then transform val/test. |
| **Shuffling with time-series data** | Model appears to have 99% accuracy. | Never shuffle temporal data before train/test split. |
| **Not pinning GPU memory in data loader** | GPU utilization oscillates 0% → 100% → 0%. | Use pinned memory + async transfers + multiple DataLoader workers. |

---

## RAG / LLM Applications

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Naive chunking breaks context** | Retrieved chunks are mid-sentence fragments, LLM generates incoherent answers. | Use semantic chunking with overlap. Respect sentence/paragraph boundaries. |
| **Embedding model mismatch** | Search returns irrelevant results. | Use the same embedding model for indexing and querying. Different models have different vector spaces. |
| **Ignoring metadata in retrieval** | "What was our Q4 revenue?" returns Q3 data. | Add document metadata (date, source, category) as filterable fields. |
| **No reranking** | Top-10 retrieval has the answer at position 7, but LLM only sees top 3. | Add a reranker step between retrieval and generation. |

---

## Systems / Infrastructure

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **vLLM default parameters in production** | OOM errors under load despite "enough" GPU memory. | Explicitly set `--gpu-memory-utilization` (default 0.9 is optimistic). Set `--max-num-seqs` based on your actual request patterns. |
| **Docker `COPY . .` with large context** | Image builds take minutes just to send context. | Use `.dockerignore` aggressively. Keep the build context minimal. |
| **Cache stampede on cold start** | Service crashes immediately after deploy when cache is empty. | Use cache warming, staggered TTLs, or request coalescing (single flight pattern). |

---

## CAN Bus / Embedded

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Termination resistor missing** | Bus works at low speed, fails at high speed with frame errors. | 120Ω resistor at both ends of the CAN bus. Always. |
| **Bit timing mismatch** | Two nodes can't communicate despite same baud rate. | Baud rate alone isn't enough — check sample point, SJW, propagation segment. Use a CAN analyzer. |

---

## Static HTML / GitHub Pages Tutorials

### Index page layout broken by theme.css grid

**Symptom:** Tutorial index.html page shows cards stacked in a narrow ~260px column instead of filling the full 1400px container width. The card grid looks broken — cards are squeezed into a sidebar-width column.

**Root cause:** Every `theme.css` sets `.container { display: grid; grid-template-columns: 260px 1fr; }` for the chapter sidebar layout. The index.html template uses the same `.container` class but needs a single-column layout. If the index template's `.container` doesn't explicitly set `display: block;`, the theme's `display: grid` bleeds through from the linked stylesheet.

**Fix:** The index-template.html MUST include `display: block;` as the first property in its `.container` rule:

```css
.container {
    display: block;  /* Override theme.css grid layout */
    max-width: 1400px;
    ...
}
```

**Prevention:** The canonical index-template.html lives in `.claude/skills/tutorial-publisher/assets/index-template.html`. Always use this template — never recreate the index page from scratch. When deploying index.html updates, copy from this template, not from another tutorial directory (which may be outdated).

**Why tutorials had same root cause:** All five theme CSS files (blue-purple.css, green.css, warm.css, violet-teal.css, slate-blue.css) define `.container { display: grid; }`. The index page links to theme.css, so the grid layout always leaks unless overridden.

### Shell scripts leave CWD in wrong repo

**Symptom:** `git add` fails with "pathspec did not match any files" when running batch scripts across multiple git repos. The working directory from a prior script run points into a different repository.

**Fix:** Always use `git -C <absolute-path>` rather than `cd` in batch scripts. When chaining commands with `&&`, each `git -C` is self-contained and immune to prior CWD state.

---

## NLP

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| **Tokenization mismatch in production** | Model outputs garbage text in deployment but works in notebook. | Save the tokenizer with the model. Different tokenizers produce different token IDs for the same text. |
| **Max length truncation** | Long documents silently truncated, losing critical information. | Log truncation events. Consider sliding window or hierarchical approaches for long texts. |

---

## General Principles

1. **Measure before optimizing.** The bottleneck is never where you think it is.
2. **Check return codes.** Silent failures cost days of debugging.
3. **The default is rarely what you want.** Default streams, default memory, default parameters — they're safe, not fast or correct.
4. **Write the test before the fix.** If you can't reproduce it, you haven't understood it.
5. **Libraries before custom code.** Someone smarter already solved your problem, and their solution is better tested.
