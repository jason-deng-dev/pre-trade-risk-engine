# Inference Engine v4 — Master Spec

### Read → Build → Measure → Explain: every reading assignment arrives exactly when the project needs it

**What it is:** A low-latency inference serving engine in C++17: streaming clients → admission control → continuous-batching scheduler → a from-scratch transformer decoder → responses, all under measured TTFT and TPOT budgets. Every request passes through a sub-microsecond NUMA-aware admission check — rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch — before it reaches the scheduler; the scheduler admits on a projected KV budget and batches at iteration level. The engine is driven by an **open-loop load generator** (loopback) with a sequenced request stream, so latency is measured under realistic concurrent load, coordinated-omission-correct. The CUDA kernel ladder (GEMM naive → register-blocked, roofline-analyzed) and the decoder's achieved-bandwidth claim share one machine, one harness, one ceiling table. Every optimization claim is backed by `perf stat` and an ablation study.

**What changed in v4:** the third project's *domain* pivots from a pre-trade risk engine to an inference serving gateway. The curriculum does not pivot — Phases 0–5, the never-cut list, the correctness machinery, and the perf playbook survive as a re-skin. The decision record is [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md); the component-for-component mapping is in its §4. New: Phase 6, the continuous-batching scheduler, which was previously "know-only, deliberately skipped" and is now the centerpiece. The AI spine is no longer an extension — the decoder and the kernel bench build *inside* this repo, and the old two-track ordering amendment is superseded.

**This file is canonical.** It holds the operating rules, the timeline, and the cut order — the things that arbitrate disagreements. Phase docs say what to read and build; if a phase doc and this file disagree, this file wins. Weeks and deadlines live only in the table below, never restated in a phase doc, so there is nothing to drift.

---

## Doc index

| Doc | What it holds |
| --- | --- |
| [`00-master.md`](00-master.md) | **This file.** Scope, operating rules, spines, cut order, non-goals |
| [`01-reading-map.md`](01-reading-map.md) | Your reading list → verdict → where it lands; foundation supplements; post-project queue |
| [`02-foundations.md`](02-foundations.md) | The five foundations and where each one is built |
| [`03-explain-loop.md`](03-explain-loop.md) | The retention mechanism: weekly cold recall, monthly walkthrough, the prediction ledger |
| [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) | **Decision record (v4).** Domain pivot, component mapping, scheduler decisions, repo mechanics |
| [`10-phase-0-reading.md`](10-phase-0-reading.md) | OSTEP/CiA execution order, Drepper, supplements, harness, false-sharing toy, SPSC ring capstone |
| [`11-phase-1-core.md`](11-phase-1-core.md) | Admission-control core, circuit breaker, accounting ledger, fuzz/TSan/differential/replay gate |
| [`12-phase-2-microarch.md`](12-phase-2-microarch.md) | Cache/topology experiments, branchless, huge pages, NUMA cloud day |
| [`13-phase-3-loadgen.md`](13-phase-3-loadgen.md) | Open-loop load generator, request stream, overload/shedding policy, latency methodology |
| [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md) | `new`/`delete` interposition and the ablation study |
| [`15-phase-5-docs.md`](15-phase-5-docs.md) | README, design doc, proof pack, post-project reading queue |
| [`16-phase-6-scheduler.md`](16-phase-6-scheduler.md) | **Phase 6.** Continuous batching, KV-budget admission, chunked prefill, the batching benchmark tables |
| [`20-ai-extension.md`](20-ai-extension.md) | AI/ML curriculum: steps H→B→C→D (Orderbook/Backtester) plus A and G, which now build inside this repo |
| [`30-perf-playbook.md`](30-perf-playbook.md) | `perf stat` from zero to claim discipline — referenced by Phases 0, 2, and 4 |
| [`31-verification-and-observability.md`](31-verification-and-observability.md) | The sanitizer CI matrix (all three repos) and the observation-cost rule — referenced by Phases 1, 4, and 5 |
| [`40-deepening-queue.md`](40-deepening-queue.md) | Cross-project additions to the Orderbook and Backtester, gated on core progress |
| [`41-network-ingestion.md`](41-network-ingestion.md) | **Orderbook.** The AF_XDP ingestion ladder, wire timestamps, and the real ITCH message mix — the largest anchor workstream |
| [`90-resumes.md`](90-resumes.md) | House style, distilled bullets, keyword discipline, Appendix A (quant) and B (AI-infra) |
| [`91-interview-territory.md`](91-interview-territory.md) | The full interview map (Appendix E) — the master checklist for interview season |
| [`92-algorithms-and-career.md`](92-algorithms-and-career.md) | Algorithms track (Appendix C) and career strategy (Appendix D) |

Supporting docs stay behind the master. The phase docs carry their own 📖 Read / 🔨 Build / ✅ Verify blocks and 🎯 drills; nothing here restates them.

---

## How to use this document (operating rules)

1. **Bracket rule** — any `[N]` placeholder must be replaced with a measured value before the claim ships anywhere (resume, interview answer). Unfillable bracket = delete the clause.
2. **Honesty rule** — every claim needs mechanism, measurement, boundary. Never claim what you can't defend under cross-examination.
3. **Weekly review (10 min) + cold recall (20 min)** — review: planned vs. actual hours per Toggl category; a stalled category is next week's first priority. Categories: Algorithms / Systems / Concurrency / Projects / Anki (≤30% of reading time) / Interview Prep (the 🎯 drill blocks). Then run the week's cold-recall session per [`03-explain-loop.md`](03-explain-loop.md) — it is the leg of the loop that otherwise quietly disappears.
4. **Resume is output, not input** — the resume appendices describe work that exists; they are targets until brackets are filled.
5. **Hardware**: 9800X3D desktop (single NUMA node, Omarchy/Arch), RTX 3060 laptop; cloud for CUDA/NUMA measurement days. **Pace calibration**: ~100–140 dense pages/week with Anki; keep units small and phase-gated.
6. **Session setup**: new chat → attach **this file + the phase doc you're working in** → brief: *"You are my AI-infra / systems prep assistant. This spec is canonical. Operating rules as above. Check plan-vs-actual against Toggl when I report progress."* No other doc needs attaching unless you're working inside it.

---

## Curriculum Design (why the reading load is intentional)

This is **not** a fastest-path project plan. It is a learning curriculum embedded in a production-style build. Reading is not separate from the project; each reading block exists to unlock a concrete build, measurement, or design decision. The desired output is not only a finished engine, but durable systems knowledge that can be explained, measured, and transferred to adjacent domains — and the quant anchors keep the market-microstructure and statistical side alive without competing for the engine's hours.

The curriculum has two spines:

| Spine | Purpose | Completion rule |
| --- | --- | --- |
| **Inference Engine Spine** | Build the systems, concurrency, correctness, and measurement foundation through the serving engine — phases 0–6, with the CUDA kernel bench (step A) and the decoder (step G) built inside the same repo | Must stay green; nothing else activates at the cost of a skipped gate here |
| **Quant Anchors** | Keep Orderbook and Backtester sharp: the deepening queue, the ingestion ladder, plus AI steps H (feature layer), B (point-in-time datasets), C (training literacy), D (evaluation rigor) | Gated on each project's own prerequisites; pauses whenever the engine spine needs the hours |

The learning loop for every phase is:

1. **Read** the minimum material needed to understand the mechanism.
2. **Build** the artifact that forces the concept to become concrete.
3. **Measure** the result and identify the boundary.
4. **Explain** the concept without notes — in the design doc, an interview drill, or a short recorded walkthrough.

**Learning-transfer checkpoint (end of every phase):** answer these five prompts in the design doc or a short note:

- What did I learn?
- Where did I apply it?
- What measurement or test confirmed it?
- What is the boundary or exception?
- Where else does this concept transfer?

| Phase | Transfer you should be able to explain without notes |
| --- | --- |
| 0 | Memory ordering, SPSC mechanics, context switches, virtual memory, cache basics |
| 1 | Invariant design, fail-safe consistency, fuzzing, TSan, deterministic replay |
| 2 | False sharing, branch prediction, TLB pressure, SMT/NUMA topology, experimental control |
| 3 | Open-loop vs. closed-loop load, backpressure, tail-latency methodology, coordinated omission |
| 4 | Allocation behavior, object lifetime, interposition, ablation discipline |
| 6 | Iteration-level scheduling policy, KV memory management, admission under two SLOs, batch economics |
| Kernels (A) | CUDA execution model, coalescing, tiling, occupancy, roofline, quantization cost |
| Decoder (G) | Attention, KV cache, prefill vs. decode, transformer internals, correctness against a reference implementation |
| Quant anchors (H/B/C/D) | Feature pipelines, leakage control, training literacy, model export, evaluation rigor |

**Interface rule:** the two spines meet at exactly two interfaces, both designed in phase 1 before either consumer exists — the **model backend interface** (decoder today, ONNX Runtime optionally) and the **KV allocator interface** (contiguous today, paged in phase 6). Those two seams are the only forward-looking design the engine carries; everything else is built for the phase that needs it. Inline hooks in the phase docs are marked **AI hook**; the ML curriculum is in [`20-ai-extension.md`](20-ai-extension.md).

---

## Inference Engine Spine scope: relative size

| Phase | Content | Size |
| --- | --- | --- |
| 0 | **OSTEP ch. 1–34 sequential w/ carve-outs** + **CiA full (ch. 1–11, depth-noted)** + McKenney + Drepper §1–3 + harness + false-sharing toy + **SPSC ring v1/v2** | the dominant block |
| 1 | Admission control core + correctness (fuzz/TSan/differential/replay) | ~1.5 units |
| 2 | Micro-arch: false sharing, NUMA, huge pages, branchless | ~1 unit |
| 3 | Open-loop load generator + pipeline integration + latency methodology | ~1 unit |
| 4 | Zero-alloc + ablation | ~1 unit |
| 5 | Docs + post-project reading queue | ~0.5 unit |
| 6 | **Continuous batching + KV-budget admission (+ chunked prefill)** — the centerpiece | ~2 units, honestly sized; a concurrent policy system, not a filter |
| A | CUDA + kernels bench (GEMM ladder, roofline, one quantized kernel) | 4–5 weeks |
| G | From-scratch decoder (token-for-token gate) | 1–1.5 weeks |

**On the clock — there isn't one.** The sizes above are *ratios*, not a schedule. Read them as: Phase 0 is the bulk of the commitment, roughly equal to everything after it combined; each build phase is about one unit of work. The table exists so you know the shape of what you're signing up for and what to cut first when life compresses it. Track **hours per category** (rule 3), not weeks elapsed. A phase that takes twice as long and lands green is not behind schedule; a phase that lands on time with a skipped gate is.

The **Quant Anchors** run alongside: H→B→C→D are small items on Orderbook and Backtester, gated on their own prerequisites, and they pause whenever the engine spine needs the hours — the pause is built into the design, not a failure of it. The one anchor item that is not small is the Orderbook's network-ingestion ladder ([`41-network-ingestion.md`](41-network-ingestion.md)), and it is sized in rungs precisely so it can pause without stranding anything: rungs 0–1 plus wire timestamps and the sanitizer CI ([`31-verification-and-observability.md`](31-verification-and-observability.md)) need no special hardware and are the always-available floor; rungs 2–3 need a NIC that supports them and are the first anchor work to pause.

**Cut order if time runs short:**

1. SIMD batch checks (already a stretch — the design-doc Q&A bullets survive the cut)
2. **Second backend (ONNX Runtime) — cut before any core item**; the decoder backend alone carries the story
3. Half of Phase 2's experiments (keep NUMA + false sharing; branchless and huge pages are the cut candidates)
4. Phase 6 rung 2 (chunked prefill) — the scheduler ships at rung 1 with the ladder documented and rung 2 named as designed-but-unbuilt
5. Phase 6 preemption rungs (recompute/swap)
6. Load-generator realism (keep one recorded stream + one synthetic flavor)
7. Snapshot-recovery stretch (already demoted)
8. Step G's stretch items (sampling variants, GPU path) — and cut A5 (the quantized kernel) before cutting the GEMM ladder

The decoder itself is no longer cuttable: it is the scheduler's backend, and phase 6 has nothing to serve without it.

**Never cut:** Phase 0 reading and the SPSC ring (it's the longest block, and that's the point — the OSTEP/CiA spine is the foundation everything else stacks on); the correctness gates (fuzz / TSan / differential / replay); **the scheduler correctness gate** (output equality across batch compositions); the decoder's token-for-token gate against llama.cpp; ablation; zero-alloc verification.

---

## Execution order amendment (proposed 2026-09-28) — SUPERSEDED by v4

The amendment proposed running H→B→C→D→A→G ahead of the risk engine's Phase 2–4 grind, to get AI artifacts on the resume before the core shipped. It was never adopted, and v4 retires it: with the pivot, the kernel bench (A) and the decoder (G) are built *inside* the engine repo, so there is no separate track to run them ahead of. The v4 sequence — **Phase 0 → A → G → Phases 1–5 → Phase 6**, with H/B/C/D continuing on Orderbook and Backtester — replaces it. See [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) §5. The amendment's one surviving insight is kept: the kernels bench and the decoder ship as finished artifacts before the gateway needs them.

---

## What this project is *not*

- Not a protocol implementation — no OpenAI-compatible API, no HTTP/2 stack, no gRPC, no SSE streaming layer. A simple binary request protocol on loopback carries the signal (arrival rates, sequencing, backpressure) without protocol plumbing.
- Not a product — no auth, no model routing, no multi-tenancy beyond quota accounting, no admin UI. Tenant quotas exist to give admission control something to enforce, not to build a SaaS.
- Not a model-serving framework — one model (GPT-2 124M), one backend by default. ONNX Runtime is an optional second backend and the first thing cut.
- Not an ML project — the model is a fixed, well-understood checkpoint and a deliberate placeholder. The work is the serving and performance layer; say exactly that when probed.
- Not a training system — no fine-tuning, no distributed training, no NCCL. Know the vocabulary, claim the boundary.
- Not kernel bypass **in this repo** — explain *why* it exists (syscall/context-switch overhead, measured in Phase 2) and know the physics cold. The engine's ingress stays a loopback request stream by design; the build lives where a real feed exists, in the Orderbook's ingestion ladder ([`41-network-ingestion.md`](41-network-ingestion.md)). One repo builds it, one repo explains it, and no claim is made in the wrong one.
- **Not distributed at all, deliberately.** One process, one node, a loopback load generator — no replication, no consensus, no multi-node anything. This is a real boundary for AI-platform roles, where the hiring signal is distributed systems rather than single-node work. State it as a boundary in interviews rather than hiding it, and note the cheap extension if that door becomes primary: DDIA ch. 5–6 plus one small distributed artifact (a replicated log, or a sharded service with a failover path). Do not add it pre-emptively.
- Not an AVX2 showcase — the design-doc Q&A bullets prove literacy; the build adds little.

**Shipped > ambitious.** The build block after Phase 0 is small relative to the reading block that precedes it. Doing it at 100% over a longer stretch beats doing it at 60% on a schedule — the ratio between the two blocks is the design, and the ratio is what the sizing table is for.

---

## Why v4 maximizes ROI

Reading, building, measuring, and explaining are the same schedule, not competing ones. Every reading block is cashed in by a build step that *requires* it, every build step surfaces the exact question the next reading block answers, and every phase ends with a transfer check that converts the work into interview-ready understanding. One repo carries the whole system — kernel ladder, decoder, gateway, scheduler — so the claims reinforce each other instead of living in separate artifacts: the decoder's achieved-bandwidth number is measured against the ladder's ceiling, and the scheduler's throughput curves are explained by the same ceiling again. You finish with a measured inference engine **and** the interview-relevant 80% of your reading list consumed at maximal retention, with Orderbook and Backtester standing as the quant anchors.
