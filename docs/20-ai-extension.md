# AI Infrastructure Extension Spine (dependency-gated)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`40-deepening-queue.md`](40-deepening-queue.md) (step H is the microstructure-features row) · [`90-resumes.md`](90-resumes.md) Appendix B (the AI-infra resume this feeds)

Purpose: extend the **Core Quant Spine** into AI-infrastructure / ML-systems capability by distributing the AI skill loop across your *existing* projects. This is not a separate AI project and not a profile pivot. It is the same systems curriculum continued into feature pipelines, point-in-time data, model export, inference serving, CUDA, and evaluation rigor. Resume framing remains the serving-and-performance layer, not ML research.

**Activation rule:** core-phase design hooks are built inline; AI extension builds activate only when their prerequisite core artifact exists **and the current core gate is green**. If the core schedule slips, AI modules pause rather than cannibalize the risk engine. A module can be deferred without breaking the core spine.

**Core hooks already built into the phases:**

| Core artifact | Hook preserved for the extension spine |
| --- | --- |
| Deterministic replay (Phase 1) | Versioned decision inputs/configuration → future model-version pinning |
| Feed → Orderbook → strategy pipeline (Phase 3) | Clean zero-alloc feature boundary → future microstructure features |
| Latency/ablation harness (Phases 3–4) | Pluggable decision backend + batch-size dimension → future serving sweep |
| Backtester Monte Carlo | Seeded deterministic partitioning → future CPU/GPU replay |
| Backtester simulation data | Point-in-time boundaries → future leakage-free datasets |

**The extension pipeline** — each step feeds the next. H→B→C→E→F is ~4–5 weeks of small items; A is another ~4–5 weeks (it now carries the kernels bench); G is ~1–1.5 weeks; D can run anytime after C. H is the microstructure-features row in the [deepening queue](40-deepening-queue.md), reframed as the pipeline's feature layer.

| Step | Project | Natural slot / gate | Build | AI skill it teaches | Effort | Resume line shape |
| --- | --- | --- | --- | --- | --- | --- |
| H | Orderbook | After Phase 3, if the feed/book pipeline is green | Zero-alloc feature layer: OFI, microprice, queue-position estimate (already queued in [`40-deepening-queue.md`](40-deepening-queue.md)) | Feature engineering at latency | 1–2 wks | "Zero-alloc microstructure feature layer at X msgs/sec" |
| B | Backtester | After H produces usable features and the Backtester replay is stable | Point-in-time dataset generation from sims with strict leakage discipline (labels from future data = banned) | Data engineering for ML | ~1 wk | "Leakage-free point-in-time dataset pipeline" |
| C | Backtester | After B produces a validated dataset | Tiny model (logistic/GBM, PyTorch/Python) trained on B's data, exported to ONNX | Training-loop literacy | 2–3 days | "Trained/evaluated baseline models, ONNX export" |
| E | Risk engine | After C and after the Phase 3 integrated risk pipeline exists | Inference stub: strategy = the ONNX model consuming H's features; model version pinned in deterministic replay | Inference on a production hot path | 3–4 days | "ONNX inference on hot path, replay-pinned model versions" |
| F | Risk engine | After E and after the Phase 3–4 latency harness exists | Batch-size vs. p50/p99 sweep on E (coordinated-omission-corrected) | Serving tradeoffs | ~1 day | "Quantified batch-size vs. p99 tradeoff" |
| A | Backtester + kernels bench repo | After the CPU Monte Carlo has a deterministic seeded baseline; may run after C or in parallel once gated | CUDA Monte Carlo (path generation on GPU, seeded reproducibility CPU↔GPU) **plus the kernels bench**: GEMM ladder naive→coalesced→tiled→register-blocked, occupancy sweep, roofline figure, one INT8 or fused kernel | CUDA — the screening currency; kernel optimization is the accelerator-company probe | 4–5 wks | "GPU Monte Carlo (CUDA), bit-identical replay across CPU/GPU; GEMM ladder at [A]% of peak DRAM bandwidth and [B]% of peak FP32 FLOPs, roofline-analyzed with Nsight Compute" |
| G | Kernels bench repo | After A (roofline and bandwidth reasoning) and after E/F (serving mechanics, batch economics) | From-scratch C++17 GPT-2-class decoder: weight dump, naive forward pass, KV cache, greedy decode, single-stream benchmarks; correctness by token-for-token match against llama.cpp | Transformer internals on real hardware — converts the know-only LM vocabulary into a measured artifact | 1–1.5 wks | "From-scratch GPT-2-class decoder in C++17 (KV cache, greedy decode): [X] tok/s single-stream, [N] ms TTFT at 2k context; verified token-for-token against llama.cpp" |
| D | Backtester | Anytime after C; can run in parallel with A | Model evaluation harness: walk-forward validation + null ensembles applied to model signals vs. baseline | ML evaluation rigor | 2–3 days | "Walk-forward eval with null ensembles separating model edge from luck" |

**Hardware (cloud route, your choice):** CUDA runs only on NVIDIA — the RX 9070 XT can't, the laptop's RTX 3060 can, but the chosen path is cloud: Colab's free tier (T4) or ~$1/hr instances cover the entire CUDA workload, kernels bench and decoder included — same rent-the-hardware pattern as the NUMA cloud day in [`12-phase-2-microarch.md`](12-phase-2-microarch.md). Use real `nvcc` + Nsight in the cloud, never a translation layer (ZLUDA / ROCm-on-consumer): the profiling toolchain is part of what you're learning, and the resume currency is *CUDA* specifically.

**Optional extension (~1 evening, after step E exists):** *third decision backend — API structured-decision model (e.g., Jev, TypeSafe AI's "System One" classifier: typed probabilistic decisions, ~70–500ms, schema-guaranteed) vs. local ONNX.* Run the same dataset/state through both; measure decision latency, cost-per-decision, calibration (predicted vs. realized frequencies), and determinism (bit-identical replay vs. an opaque versioned API — your ethos vs. theirs, with numbers). Frame the claim generically ("evaluated API-based structured-decision models against local ONNX"); cite the specific vendor only in the design doc. Also one free narrative paragraph: agent guardrails (classifying tool calls before execution) are the same pre-execution-guardrail pattern as your risk engine, one domain over. Measured comparison only — "I called a new API" is the anti-claim. If the comparison numbers don't materialize, the item doesn't ship.

**Two rules (non-negotiable):**

1. **Models are plumbing, not alpha.** Every model is tiny and honest; all claims are about the pipeline and its measurement, never model performance. Scripted probe answer: "my work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."
2. **Bracket rule applies.** No AI bullet ships without its measurement; the ONNX stub reaches the resume only when replay-pinned determinism is demonstrated.

**Deliberately skipped (know the vocabulary, can't claim the build):** distributed training / NCCL, continuous batching, RAG/vector DBs, MLOps-at-scale. (KV caches left this list — step G builds one and measures it. Everything else here stays vocabulary-only.) If probed: the boundary answer above. **Resurrected by the AI extension spine** (previously cut in this spec, now covered): COD 6.6 + Appendix C and 6.7 (slotted into step A), COD 3.5/3.6–3.7 upgraded to real reads (see thrive layer), DDIA ch. 10–11 figures skim. Still skipped: CiA ch. 9–10 (one thread-pool paragraph only, see thrive layer), OSTEP persistence half, Top-Down ch. 7–8.

**End-state narrative:** data → features → training → export → inference → measurement, executed across three existing projects — "the AI loop wired through the quant stack at microsecond latency." Small bonus: note "HIP-portable kernel design" in the CUDA writeup (knowing where the portability boundary sits is itself a signal).

---

### AI Extension Spine: Learning Curriculum (concepts → resources → interview drills)

Keyed to the pipeline steps so each concept is applied within days of learning it. Your systems background is a shortcut throughout: GPU memory hierarchy, tail latency, and throughput-vs-latency are the same physics as Drepper/COD/Gil Tene, re-skinned — that's the interview edge; the drills below are what "thriving" looks like.

**Step H — feature layer (Orderbook).** *Concepts:* order-flow imbalance, microprice, queue-position estimation; why these predict short-horizon prices (adverse selection, informed flow). *Resources:* Cont, Kukanov & Stoikov, *The Price Impact of Order Book Events* (2014 — short paper, the OFI original); Harris, *Trading and Exchanges*, queue/limit-order chapters (skim). *Drill:* "why does order imbalance predict short-horizon moves?"

**Step B — dataset pipeline (Backtester).** *Concepts:* point-in-time correctness; label leakage / lookahead bias (the #1 probed topic); time-series splits — never shuffle; purged CV + embargo periods; class imbalance; rolling z-scores. *Resources:* López de Prado, *Advances in Financial Machine Learning*, ch. 3–7 only (dense but the reference); scikit-learn docs for mechanics. *Drills:* "why does random k-fold fail on time series?" "where does leakage hide in feature engineering?"

**Step C — training literacy (Backtester, Python).** *Concepts:* the floor — linear/logistic regression, gradient boosting (LightGBM/XGBoost), loss functions, overfitting vs. regularization, bias-variance; PyTorch floor — tensors, autograd, one training loop (learn to *talk to* ML engineers, not become one); ONNX as exchange format. *Resources:* PyTorch *60-Minute Blitz* (free, an evening); LightGBM/XGBoost docs; `onnx`/`onnxruntime` docs. **Skip ML textbooks** — a time sink for plumbing-level literacy. *Drills:* "why GBM over a neural net here?" "what's actually inside an ONNX file?"

**Steps E+F — inference on the hot path (Risk engine).** *Concepts:* serving mechanics — sessions, tensors, warmup; **numeric determinism** — FP non-associativity, why CPU vs. GPU inference can differ bitwise (the intellectual bridge to step A's replay story; learn it here); model versioning; batch economics — Little's Law, batch size trading p50 against throughput; **vocabulary here, built in step G:** KV cache, prefill/decode (learn the concept here, expect to write it there); **stays know-only:** continuous batching. *Resources:* ONNX Runtime C++ API docs; **plus the ONNX Runtime determinism and execution-provider documentation** — the plan says "learn numeric determinism here" and this is the missing resource: determinism is not a global switch but a per-provider property (kernel selection, reduction order, and provider fallbacks all vary), so the reading is "which knob pins which provider, and where the guarantee stops"; vLLM blog/paper (vocabulary only, two evenings); TensorRT docs skim (why INT8/FP16 quantization buys throughput). *Drills:* "CPU vs. GPU inference at batch=1 — walk the tradeoff" (answer from your own F-sweep — the strongest answer in the set), "what is quantization and what does it cost?", "why can inference be non-deterministic?"

**Step A — CUDA (Backtester + kernels bench repo, the big one).** Five sub-steps. Every read below unlocks the build under it, and A5 comes last because quantization is the only part that needs the precision ladder. The kernels bench is its own small repo, not a directory inside the Backtester — a reviewer opening the Backtester should find a simulation engine, and a reviewer opening a kernel bench should find measured tables. Keeping them apart is what makes each legible.

**A1. 📖 *GPU architecture.* COD 6.6 (GPU intro) + Appendix C skim, then 6.7 (DSAs/TPU) skim** — SIMT, warps, thread blocks, and the GPU memory hierarchy (registers → shared → L2 → HBM) at textbook altitude before touching code; this is also the accelerator-landscape vocabulary (TPU as archetypal DSA) for AI-chip company interviews. Drepper §1–3 and COD ch. 5, re-skinned.
🔨 **Build:** NVIDIA DLI free intro course (few evenings, hands-on).
✅ **Verify:** why is a GPU bandwidth-rich but latency-poor?

**A2. 📖 *Programming model and coalescing.* CUDA C Programming Guide** as desk reference — kernel launches, grid/block/thread geometry, `__global__`, memory **coalescing** — plus *streams and async*: `cudaMemcpyAsync`, overlapping H2D/compute/D2H, directly analogous to your SPSC pipelining.
🔨 **Build:** *GPU Monte Carlo in the Backtester.* Path generation on GPU, coalesced, seeded CPU↔GPU bit-identical replay, profiled with Nsight Systems on the cloud T4. Do this sub-step first among the CUDA work: it is the "make CUDA run at all" step and it reuses your already-shipped CPU Monte Carlo, so the correctness story is a replay comparison rather than a new invariant. **AI hook preserved:** per-path seeding and the deterministic partition scheme carry over unchanged from the CPU version; the CPU↔GPU equivalence test is the same harness you use for thread-count invariance.
✅ **Verify:** your CPU↔GPU seeded replay story — same path hashes, same terminal equities, across the device boundary.

**A3. 📖 *Tiling and shared memory.* Kirk & Hwu, *Programming Massively Parallel Processors* (4th ed., ch. 1–4)** — thread/block geometry, memory coalescing, shared memory, tiling, and bank conflicts.
🔨 **Build:** *the GEMM ladder, part 1.* Naive → coalesced → shared-memory tiled. Measure each rung as you add it; the ladder table is the artifact, and a rung that buys nothing is a finding worth keeping.
🎯 **Drill:** "walk me through coalescing."

**A4. 📖 *Occupancy and roofline.* Kirk & Hwu ch. 5–6** — occupancy, performance limits, and the matrix-multiplication case study — plus the **Nsight Compute** tutorials (the kernel-level profiler: occupancy achieved, memory throughput, stall reasons) and the roofline model in its primary source — **Williams, Waterman & Patterson, *Roofline: An Insightful Visual Performance Model* (2009)**, short and citable, and the origin of the arithmetic-intensity framing you will be asked to reproduce on a whiteboard.
🔨 **Build:** *the GEMM ladder, part 2.* Register-blocked kernel, occupancy sweep, and the **roofline figure** — achieved FLOPs and achieved DRAM bandwidth plotted against the machine's ceilings. Document where the ladder stops scaling and what the limiter is. Profiling is part of the build, not a postscript: every rung's number gets Nsight Compute output linked to it.
✅ **Verify:** roofline a kernel out loud; explain how occupancy interacts with shared memory; say which rung hit which ceiling and why.

**A5. 📖 *Precision and quantization.* COD 3.5 (IEEE-754) as a real read** — you cannot answer "what does quantization cost" without knowing what a mantissa is — **plus quantization material: TensorRT's quantization docs upgraded from skim to read, and one INT8 GEMM reference.**
🔨 **Build:** *one fused or quantized kernel.* INT8 dequant-GEMM or fused attention — pick one, not both. Measure throughput gain and accuracy cost against the FP32 baseline, and write both numbers down.
🎯 **Drill:** "what is quantization and what does it cost?", answered from your own accuracy delta, not from a blog.

*RISC-V note:* if the accelerator-company path (Tenstorrent/Openchip-style) firms up, skim COD's RISC-V "Real Stuff" sections — "RISC-V vs. x86" is a known probe.

**Step G — tiny decoder (transformer internals, ~1–1.5 weeks).** *Gate:* after A (the roofline and bandwidth reasoning are what make decode legible) and after E/F (serving mechanics and batch economics). This step exists to convert the know-only LM vocabulary into a measured artifact. It is deliberately *not* a serving engine, and the cut list below is what keeps it from becoming one.

*Cut list, fixed in advance — each item here is a separate project's worth of work:* no tokenizer beyond a naive BPE over a dumped vocab; no HTTP server or OpenAI-compatible API; no continuous batching; no quantization of the decoder; no GPU path in v1; nothing larger than GPT-2 124M.

*Concepts:* scaled dot-product attention, multi-head attention, causal masking, layernorm, the residual stream, prefill vs. decode, the KV cache and what it costs in memory, why decode is bandwidth-bound and prefill is compute-bound, sampling (greedy, temperature, top-k), TTFT vs. inter-token latency.

*Resources:* Jay Alammar, *The Illustrated Transformer* (~1 hr, free — attention mechanics without the math overhead) → Karpathy, *Let's build GPT: from scratch, in code, spelled out* (~2 hrs — mechanism only, the Python is incidental; this is the one ML video worth the time) → llama.cpp's architecture docs and a read of its KV-cache and sampling source as a **C++ reference implementation** — reading a working C++ implementation of the thing you are about to write is the highest-leverage hour in this step → *Attention Is All You Need*, skim for vocabulary only. The bandwidth reasoning comes from A4 and the precision ladder from A5; nothing new is needed for either.

*Build, in this order:* (1) a one-file Python weight-dump script — GPT-2 124M weights and vocab to flat binaries; Python stops there. (2) Naive decoder: embeddings → single-head attention → MLP → layernorm → logits → greedy decode, CPU, fp32. (3) **Correctness gate before any optimization:** greedy output must match llama.cpp (or HF) token-for-token on a fixed prompt for the first N tokens. This is your bit-identical-replay habit applied to a new domain, and it is the single most credible thing about the artifact — most from-scratch decoders in portfolios have no correctness claim at all. (4) KV cache, re-run the correctness gate, then re-benchmark. (5) Multi-head, then sampling if cheap. (6) Measure: tok/s single-stream; TTFT at 512/1024/2048 context; achieved DRAM bandwidth during decode against the A4 roofline number; KV-cache footprint vs. context length. (7) *Stretch only, after everything above ships:* batch-size sweep (tok/s rises, per-token latency rises, and the bandwidth ceiling is the explanation), then a GPU path.

*Drills:* "walk me through attention and the KV cache"; "why is decode bandwidth-bound and prefill compute-bound?" (answer from your own achieved-bandwidth figure); "what does the KV cache cost at 2k context and why does that cap batch size?"; "how did you verify your implementation?" — token-for-token against llama.cpp.

*Resume line shape:* "From-scratch GPT-2-class decoder in C++17 (multi-head attention, KV cache, greedy decode): [X] tok/s single-stream, [N] ms TTFT at 2k context; output verified token-for-token against llama.cpp."

**Step D — evaluation rigor (Backtester).** *Concepts:* walk-forward (known); purged/embargoed CV applied; **multiple testing and the deflated Sharpe ratio** (your null ensembles were always a homemade version of this); information coefficient. *Resources:* Bailey & López de Prado, *The Deflated Sharpe Ratio* (short paper, very readable); ALFM ch. 11–12 skim. *Drills:* "how many backtests did you run before finding this?" "why does Sharpe overstate edge after selection?"

**Cross-cutting "thrive" layer:** FP precision ladder (FP32/TF32/FP16/INT8 — what quantization buys and costs; **COD 3.5 is now a real read**, not a skim — you can't answer "what does quantization cost" without knowing what a mantissa is; 3.6–3.7 SIMD likewise read properly — the CPU-side ancestor of GPU vector lanes); GPU-vs-CPU physics (bandwidth vs. latency, why batch amortizes); tail latency under serving load (Gil Tene — done); batch vs. stream vocabulary (DDIA ch. 10–11 figures-only skim, ~1 hour, ten Anki cards — step B/H are batch and streaming respectively); **thread pools, one paragraph** — what they are, why serving frameworks use them, and how your SPSC-spinning model differs (the contrast is the interview answer); the know-only list (NCCL/all-reduce vocabulary, distributed training concepts, MLOps). Every one is answered with either a measurement from your build or the boundary script — that pairing is the signal.

**Sequencing note:** the order above mirrors H→B→C→E→F→A→G→D so concepts land days before their build. One exception: CUDA's curriculum is self-contained and the heaviest — if motivation dips between pipeline steps, run DLI + Kirk & Hwu *in parallel* as the "second book"; cloud GPUs are on-demand either way.

---

**🎯 AI-extension drills** (answer from your own measurements wherever possible; run them through [`03-explain-loop.md`](03-explain-loop.md) like any other drill — cold, spoken, probed, colored, logged):

- *Inference (E/F):* CPU vs GPU inference at batch=1 — walk the tradeoff (your answer: your own F-sweep histograms); what is quantization and what does it cost; why can inference be non-deterministic (FP non-associativity); transformer vocabulary: attention, KV cache, prefill/decode (built in G), continuous batching (still know-only: you can explain the scheduler's tradeoff, not cite a build).
- *CUDA (A):* walk through coalescing; warp divergence; occupancy vs shared memory; roofline a kernel; the GEMM ladder and where it stops scaling; what quantization costs (your measured accuracy delta); why a GPU is bandwidth-rich but latency-poor; your CPU↔GPU seeded-replay determinism story.
- *Decoder (G):* walk me through attention and the KV cache; why is decode bandwidth-bound and prefill compute-bound; what does the KV cache cost at 2k context and what caps batch size; how did you verify the implementation (token-for-token against llama.cpp).
- *Data/eval (B/D):* where does leakage hide in feature engineering; why never shuffle time series; purged CV + embargo; "how many backtests did you run before finding this" (deflated Sharpe — null ensembles are your homemade version).
- *Boundary script (all):* "My work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."
