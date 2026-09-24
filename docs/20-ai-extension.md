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

**The extension pipeline** — each step feeds the next. H→B→C→E→F is ~4–5 weeks of small items; A is another ~3–4 weeks; D can run anytime after C. H is the microstructure-features row in the [deepening queue](40-deepening-queue.md), reframed as the pipeline's feature layer.

| Step | Project | Natural slot / gate | Build | AI skill it teaches | Effort | Resume line shape |
| --- | --- | --- | --- | --- | --- | --- |
| H | Orderbook | After Phase 3, if the feed/book pipeline is green | Zero-alloc feature layer: OFI, microprice, queue-position estimate (already queued in [`40-deepening-queue.md`](40-deepening-queue.md)) | Feature engineering at latency | 1–2 wks | "Zero-alloc microstructure feature layer at X msgs/sec" |
| B | Backtester | After H produces usable features and the Backtester replay is stable | Point-in-time dataset generation from sims with strict leakage discipline (labels from future data = banned) | Data engineering for ML | ~1 wk | "Leakage-free point-in-time dataset pipeline" |
| C | Backtester | After B produces a validated dataset | Tiny model (logistic/GBM, PyTorch/Python) trained on B's data, exported to ONNX | Training-loop literacy | 2–3 days | "Trained/evaluated baseline models, ONNX export" |
| E | Risk engine | After C and after the Phase 3 integrated risk pipeline exists | Inference stub: strategy = the ONNX model consuming H's features; model version pinned in deterministic replay | Inference on a production hot path | 3–4 days | "ONNX inference on hot path, replay-pinned model versions" |
| F | Risk engine | After E and after the Phase 3–4 latency harness exists | Batch-size vs. p50/p99 sweep on E (coordinated-omission-corrected) | Serving tradeoffs | ~1 day | "Quantified batch-size vs. p99 tradeoff" |
| A | Backtester | After the CPU Monte Carlo has a deterministic seeded baseline; may run after C or in parallel once gated | CUDA Monte Carlo: path generation on GPU; the correctness story is seeded reproducibility CPU↔GPU | CUDA — the screening currency | 3–4 wks | "GPU Monte Carlo (CUDA), bit-identical replay across CPU/GPU" |
| D | Backtester | Anytime after C; can run in parallel with A | Model evaluation harness: walk-forward validation + null ensembles applied to model signals vs. baseline | ML evaluation rigor | 2–3 days | "Walk-forward eval with null ensembles separating model edge from luck" |

**Hardware (cloud route, your choice):** CUDA runs only on NVIDIA — the RX 9070 XT can't, the laptop's RTX 3060 can, but the chosen path is cloud: Colab's free tier (T4) or ~$1/hr instances cover the entire CUDA project — same rent-the-hardware pattern as the NUMA cloud day in [`12-phase-2-microarch.md`](12-phase-2-microarch.md). Use real `nvcc` + Nsight in the cloud, never a translation layer (ZLUDA / ROCm-on-consumer): the profiling toolchain is part of what you're learning, and the resume currency is *CUDA* specifically.

**Optional extension (~1 evening, after step E exists):** *third decision backend — API structured-decision model (e.g., Jev, TypeSafe AI's "System One" classifier: typed probabilistic decisions, ~70–500ms, schema-guaranteed) vs. local ONNX.* Run the same dataset/state through both; measure decision latency, cost-per-decision, calibration (predicted vs. realized frequencies), and determinism (bit-identical replay vs. an opaque versioned API — your ethos vs. theirs, with numbers). Frame the claim generically ("evaluated API-based structured-decision models against local ONNX"); cite the specific vendor only in the design doc. Also one free narrative paragraph: agent guardrails (classifying tool calls before execution) are the same pre-execution-guardrail pattern as your risk engine, one domain over. Measured comparison only — "I called a new API" is the anti-claim. If the comparison numbers don't materialize, the item doesn't ship.

**Two rules (non-negotiable):**

1. **Models are plumbing, not alpha.** Every model is tiny and honest; all claims are about the pipeline and its measurement, never model performance. Scripted probe answer: "my work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."
2. **Bracket rule applies.** No AI bullet ships without its measurement; the ONNX stub reaches the resume only when replay-pinned determinism is demonstrated.

**Deliberately skipped (know the vocabulary, can't claim the build):** distributed training / NCCL, LLM internals (KV caches, continuous batching, RAG/vector DBs), MLOps-at-scale. If probed: the boundary answer above. **Resurrected by the AI extension spine** (previously cut in this spec, now covered): COD 6.6 + Appendix C and 6.7 (slotted into step A), COD 3.5/3.6–3.7 upgraded to real reads (see thrive layer), DDIA ch. 10–11 figures skim. Still skipped: CiA ch. 9–10 (one thread-pool paragraph only, see thrive layer), OSTEP persistence half, Top-Down ch. 7–8.

**End-state narrative:** data → features → training → export → inference → measurement, executed across three existing projects — "the AI loop wired through the quant stack at microsecond latency." Small bonus: note "HIP-portable kernel design" in the CUDA writeup (knowing where the portability boundary sits is itself a signal).

---

### AI Extension Spine: Learning Curriculum (concepts → resources → interview drills)

Keyed to the pipeline steps so each concept is applied within days of learning it. Your systems background is a shortcut throughout: GPU memory hierarchy, tail latency, and throughput-vs-latency are the same physics as Drepper/COD/Gil Tene, re-skinned — that's the interview edge; the drills below are what "thriving" looks like.

**Step H — feature layer (Orderbook).** *Concepts:* order-flow imbalance, microprice, queue-position estimation; why these predict short-horizon prices (adverse selection, informed flow). *Resources:* Cont, Kukanov & Stoikov, *The Price Impact of Order Book Events* (2014 — short paper, the OFI original); Harris, *Trading and Exchanges*, queue/limit-order chapters (skim). *Drill:* "why does order imbalance predict short-horizon moves?"

**Step B — dataset pipeline (Backtester).** *Concepts:* point-in-time correctness; label leakage / lookahead bias (the #1 probed topic); time-series splits — never shuffle; purged CV + embargo periods; class imbalance; rolling z-scores. *Resources:* López de Prado, *Advances in Financial Machine Learning*, ch. 3–7 only (dense but the reference); scikit-learn docs for mechanics. *Drills:* "why does random k-fold fail on time series?" "where does leakage hide in feature engineering?"

**Step C — training literacy (Backtester, Python).** *Concepts:* the floor — linear/logistic regression, gradient boosting (LightGBM/XGBoost), loss functions, overfitting vs. regularization, bias-variance; PyTorch floor — tensors, autograd, one training loop (learn to *talk to* ML engineers, not become one); ONNX as exchange format. *Resources:* PyTorch *60-Minute Blitz* (free, an evening); LightGBM/XGBoost docs; `onnx`/`onnxruntime` docs. **Skip ML textbooks** — a time sink for plumbing-level literacy. *Drills:* "why GBM over a neural net here?" "what's actually inside an ONNX file?"

**Steps E+F — inference on the hot path (Risk engine).** *Concepts:* serving mechanics — sessions, tensors, warmup; **numeric determinism** — FP non-associativity, why CPU vs. GPU inference can differ bitwise (the intellectual bridge to step A's replay story; learn it here); model versioning; batch economics — Little's Law, batch size trading p50 against throughput; **know-only vocabulary:** KV cache, prefill/decode, continuous batching. *Resources:* ONNX Runtime C++ API docs; vLLM blog/paper (vocabulary only, two evenings); TensorRT docs skim (why INT8/FP16 quantization buys throughput). *Drills:* "CPU vs. GPU inference at batch=1 — walk the tradeoff" (answer from your own F-sweep — the strongest answer in the set), "what is quantization and what does it cost?", "why can inference be non-deterministic?"

**Step A — CUDA (Backtester, the big one).** Concepts strictly in order — each builds on the last, and 1–2 are accelerated by your systems background:

1. *GPU architecture:* SMs, warps, thread blocks, memory hierarchy (registers → shared → L2 → HBM). Drepper §1–3 and COD ch. 5, re-skinned. Drill: "why is a GPU bandwidth-rich but latency-poor?"
2. *Programming model:* kernel launches, grid/block/thread geometry, `__global__`, memory **coalescing**.
3. *Shared memory & sync:* `__syncthreads`, bank conflicts.
4. *Streams & async:* `cudaMemcpyAsync`, overlapping H2D/compute/D2H — directly analogous to your SPSC pipelining.
5. *Occupancy & roofline:* arithmetic intensity, bandwidth- vs. compute-bound.
6. *RNG & reproducibility:* cuRAND, seeded streams, FP-order non-determinism — the CPU↔GPU replay story.

*Resources (architecture layer first — the AI extension spine resurrects previously-skipped COD material):* **COD 6.6 (GPU intro) + Appendix C skim, then 6.7 (DSAs/TPU) skim** — SIMT, thread blocks, and the GPU memory hierarchy at textbook altitude before touching code; this is also the accelerator-landscape vocabulary (TPU as archetypal DSA) for AI-chip company interviews → NVIDIA DLI free intro course (few evenings, hands-on) → Kirk & Hwu, *Programming Massively Parallel Processors* (4th ed., ch. 1–6) → CUDA C Programming Guide as desk reference → Nsight Systems/Compute tutorials (profiling is part of the build). *Drills:* "walk me through coalescing," "warp divergence?" "how does occupancy interact with shared memory?" "roofline this kernel." *RISC-V note:* if the accelerator-company path (Tenstorrent/Openchip-style) firms up, skim COD's RISC-V "Real Stuff" sections — "RISC-V vs. x86" is a known probe.

**Step D — evaluation rigor (Backtester).** *Concepts:* walk-forward (known); purged/embargoed CV applied; **multiple testing and the deflated Sharpe ratio** (your null ensembles were always a homemade version of this); information coefficient. *Resources:* Bailey & López de Prado, *The Deflated Sharpe Ratio* (short paper, very readable); ALFM ch. 11–12 skim. *Drills:* "how many backtests did you run before finding this?" "why does Sharpe overstate edge after selection?"

**Cross-cutting "thrive" layer:** FP precision ladder (FP32/TF32/FP16/INT8 — what quantization buys and costs; **COD 3.5 is now a real read**, not a skim — you can't answer "what does quantization cost" without knowing what a mantissa is; 3.6–3.7 SIMD likewise read properly — the CPU-side ancestor of GPU vector lanes); GPU-vs-CPU physics (bandwidth vs. latency, why batch amortizes); tail latency under serving load (Gil Tene — done); batch vs. stream vocabulary (DDIA ch. 10–11 figures-only skim, ~1 hour, ten Anki cards — step B/H are batch and streaming respectively); **thread pools, one paragraph** — what they are, why serving frameworks use them, and how your SPSC-spinning model differs (the contrast is the interview answer); the know-only list (NCCL/all-reduce vocabulary, distributed training concepts, MLOps). Every one is answered with either a measurement from your build or the boundary script — that pairing is the signal.

**Sequencing note:** the order above mirrors H→B→C→E→F→A→D so concepts land days before their build. One exception: CUDA's curriculum is self-contained and the heaviest — if motivation dips between pipeline steps, run DLI + Kirk & Hwu *in parallel* as the "second book"; cloud GPUs are on-demand either way.

---

**🎯 AI-extension drills** (answer from your own measurements wherever possible; run them through [`03-explain-loop.md`](03-explain-loop.md) like any other drill — cold, spoken, probed, colored, logged):

- *Inference (E/F):* CPU vs GPU inference at batch=1 — walk the tradeoff (your answer: your own F-sweep histograms); what is quantization and what does it cost; why can inference be non-deterministic (FP non-associativity); transformer vocabulary: attention, KV cache, prefill/decode, continuous batching (know-only).
- *CUDA (A):* walk through coalescing; warp divergence; occupancy vs shared memory; roofline a kernel; why a GPU is bandwidth-rich but latency-poor; your CPU↔GPU seeded-replay determinism story.
- *Data/eval (B/D):* where does leakage hide in feature engineering; why never shuffle time series; purged CV + embargo; "how many backtests did you run before finding this" (deflated Sharpe — null ensembles are your homemade version).
- *Boundary script (all):* "My work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."
