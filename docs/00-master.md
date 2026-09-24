# Real-Time Pre-Trade Risk Engine v3 — Master Spec

### Read → Build → Measure → Explain: every reading assignment arrives exactly when the project needs it

**What it is:** A production-grade pre-trade risk engine that sits between your strategy and your existing Orderbook matching engine. Every order passes through a sub-200ns NUMA-aware risk check — position limits, notional exposure, rate limiting, kill switch — before it reaches the gateway. Fills flow back into a lock-free position ledger. The engine is driven by a **mock UDP multicast feed** (loopback) with sequence-gap detection and snapshot recovery, so latency is measured under realistic market-data load. Every optimization claim is backed by `perf stat` and an ablation study.

**What changed in v3:** v2 scoped the project; v3 turns it into a learning curriculum embedded in the build. Every phase is an explicit loop of **📖 Read → 🔨 Build → ✅ Verify → 🗣️ Explain**, so reading and building reinforce each other instead of competing for time. v3 also adds Phase 0 (foundational reading and the SPSC capstone), a pre-interview reading queue mapped to your existing list, a **Foundational Core map**, and an **AI Infrastructure Extension Spine** that grows naturally out of the same projects rather than becoming a separate profile detour.

**This file is canonical.** It holds the operating rules, the timeline, and the cut order — the things that arbitrate disagreements. Phase docs say what to read and build; if a phase doc and this file disagree, this file wins. Weeks and deadlines live only in the table below, never restated in a phase doc, so there is nothing to drift.

---

## Doc index

| Doc | What it holds |
| --- | --- |
| [`00-master.md`](00-master.md) | **This file.** Scope, operating rules, two spines, timeline, cut order, non-goals |
| [`01-reading-map.md`](01-reading-map.md) | Your reading list → verdict → where it lands; foundation supplements; post-project queue |
| [`02-foundations.md`](02-foundations.md) | The five foundations and where each one is built |
| [`03-explain-loop.md`](03-explain-loop.md) | The retention mechanism: weekly cold recall, monthly walkthrough, the prediction ledger |
| [`10-phase-0-reading.md`](10-phase-0-reading.md) | OSTEP/CiA execution order, Drepper, supplements, harness, false-sharing toy, SPSC ring capstone |
| [`11-phase-1-core.md`](11-phase-1-core.md) | Dense instrument table, risk checks, kill switch, ledger, fuzz/TSan/replay gate |
| [`12-phase-2-microarch.md`](12-phase-2-microarch.md) | Cache/topology experiments, branchless, huge pages, NUMA cloud day |
| [`13-phase-3-feed.md`](13-phase-3-feed.md) | Multicast simulator, feed handler, gap/snapshot recovery, latency methodology |
| [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md) | `new`/`delete` interposition and the ablation study |
| [`15-phase-5-docs.md`](15-phase-5-docs.md) | README, design doc, proof pack, post-project reading queue |
| [`20-ai-extension.md`](20-ai-extension.md) | The AI Infrastructure Extension Spine: steps H→B→C→E→F→A→D, its curriculum and drills |
| [`30-perf-playbook.md`](30-perf-playbook.md) | `perf stat` from zero to claim discipline — referenced by Phases 0, 2, and 4 |
| [`40-deepening-queue.md`](40-deepening-queue.md) | Cross-project additions to the Orderbook and Backtester, gated on core progress |
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
6. **Session setup**: new chat → attach **this file + the phase doc you're working in** → brief: *"You are my quant-dev / AI-infra prep assistant. This spec is canonical. Operating rules as above. Check plan-vs-actual against Toggl when I report progress."* No other doc needs attaching unless you're working inside it.

---

## Curriculum Design (why the reading load is intentional)

This is **not** a fastest-path project plan. It is a learning curriculum embedded in a production-style build. Reading is not separate from the project; each reading block exists to unlock a concrete build, measurement, or design decision. The desired output is not only a finished risk engine, but durable systems knowledge that can be explained, measured, and transferred to adjacent domains such as inference serving and GPU computing.

The curriculum has two spines:

| Spine | Purpose | Completion rule |
| --- | --- | --- |
| **Core Quant Spine** | Build the systems, concurrency, networking, correctness, and measurement foundation through the risk engine | Must stay green; AI modules activate only at prerequisite gates and pause if the core slips |
| **AI Infrastructure Extension Spine** | Reuse the same projects to learn feature pipelines, point-in-time data, model export, inference serving, CUDA, and evaluation rigor | Dependency-gated; integrated at natural slots but never allowed to destabilize the core spine |

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
| 3 | UDP semantics, gap/snapshot recovery, tail-latency methodology, coordinated omission |
| 4 | Allocation behavior, object lifetime, interposition, ablation discipline |
| AI extension | Feature pipelines, leakage control, model export, inference serving, CUDA, evaluation rigor |

**AI integration rule:** during the core build, make only small design accommodations for future AI modules — versioned replay inputs, a clean feature-computation boundary, reusable latency harnesses, deterministic seeding. The actual AI builds activate at their natural prerequisite gates when the core is green. This keeps AI infrastructure inside the learning pipeline without letting it cannibalize the core quant-dev ship. The inline hooks are marked **AI hook** in the phase docs; the pipeline itself is in [`20-ai-extension.md`](20-ai-extension.md).

---

## Core Quant Spine scope: relative size

| Phase | Content | Size |
| --- | --- | --- |
| 0 | **OSTEP ch. 1–34 sequential w/ carve-outs** + **CiA full (ch. 1–11, depth-noted)** + McKenney + Drepper §1–3 + harness + false-sharing toy + **SPSC ring v1/v2** | the dominant block |
| 1 | Core + correctness (fuzz/TSan/replay) | ~1.5 units |
| 2 | Micro-arch: false sharing, NUMA, huge pages, branchless | ~1 unit |
| 3 | Mock multicast feed + pipeline integration + latency methodology | ~1 unit |
| 4 | Zero-alloc + ablation | ~1 unit |
| 5 | Docs + post-project reading queue | ~0.5 unit |

**On the clock — there isn't one.** The sizes above are *ratios*, not a schedule. Read them as: Phase 0 is the bulk of the commitment, roughly equal to everything after it combined; each build phase is about one unit of work. The table exists so you know the shape of what you're signing up for and what to cut first when life compresses it. Track **hours per category** (rule 3), not weeks elapsed. A phase that takes twice as long and lands green is not behind schedule; a phase that lands on time with a skipped gate is.

The **AI Infrastructure Extension Spine** is additional on top: H→B→C→E→F are small items, A is the large one, D is a couple of days. AI modules activate only at their natural prerequisite gates and pause whenever the core slips — the pause is built into the design, not a failure of it.

**Cut order if time runs short:**

1. SIMD batch checks (already a stretch — the design-doc Q&A bullets survive the cut)
2. Half of Phase 2's experiments (keep NUMA + false sharing; branchless and huge pages are the cut candidates)
3. Snapshot-recovery *robustness* polish (keep gap detection + one working recovery path)

**Never cut:** Phase 0 reading and the SPSC ring (it's the longest block, and that's the point — the OSTEP/CiA spine is the foundation everything else stacks on), correctness gate, ablation, zero-alloc verification, the feed handler's gap-detection path.

---

## What this project is *not*

- Not a full ITCH/FIX parser — a simplified binary format carries the signal (gap mechanics, non-blocking I/O, snapshot recovery) without protocol plumbing.
- Not an execution algo suite — discuss TWAP/VWAP conceptually in interviews.
- Not kernel bypass — explain *why* it exists (syscall/context-switch overhead); don't demo it.
- Not a TCP distributed system — one UDP hop + in-process everything else.
- Not Almgren-Chriss, not slippage attribution — uncalibrated models invite unanswerable questions.
- Not an AVX2 showcase — the design-doc Q&A bullets prove literacy; the build adds little.

**Shipped > ambitious.** The build block after Phase 0 is small relative to the reading block that precedes it. Doing it at 100% over a longer stretch beats doing it at 60% on a schedule — the ratio between the two blocks is the design, and the ratio is what the sizing table is for.

---

## Why v3 maximizes ROI

Reading, building, measuring, and explaining are now the same schedule, not competing ones. Every reading block is immediately cashed in by a build step that *requires* it, every build step surfaces the exact question the next reading block answers, and every phase ends with a transfer check that converts the work into interview-ready understanding. The AI extension spine then reuses the same artifacts — replay, latency measurement, deterministic simulation, and the Orderbook/Backtester — instead of forcing a separate project. You finish with the core risk engine **and** the interview-relevant 80% of your reading list consumed at maximal retention, with AI infrastructure treated as a natural continuation of the same systems education.
