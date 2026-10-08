# 04 — Inference Engine Pivot (v4)

> **Status: ADOPTED 2026-10-09.** This file is a decision record, not a plan. `00-master.md` is canonical and rewritten to v4; where this file and the master disagree, the master wins.
> **Supersedes** the "Execution order amendment (proposed 2026-09-28)" in `00-master.md`. The two workstreams it proposed are collapsed in v4 — see §7.
> Companion docs: [`20-ai-extension.md`](20-ai-extension.md) (steps A and G now build inside this project) · [`16-phase-6-scheduler.md`](16-phase-6-scheduler.md) (the new centerpiece) · [`40-deepening-queue.md`](40-deepening-queue.md) (unchanged).

## 1. The decision

The third project's **domain** pivots from a pre-trade risk engine to a **low-latency inference serving gateway** in C++17: streaming clients → admission control → continuous-batching scheduler → model backend (the step-G decoder) → responses, all under a measured per-request latency budget, with every optimization claim backed by `perf` and ablation.

The **curriculum does not pivot.** Phases 0–5, the never-cut list, the honesty and bracket rules, the perf playbook, and the deepening queue survive. This is a re-skin of the skeleton, not a new plan.

## 2. Why (decision record)

| Consideration | Verdict |
| --- | --- |
| Quant-dev track already anchored by Orderbook + Backtester; a third finance project adds no new signal there | Pivot |
| AI-infra screeners parse "pre-trade risk / SEC 15c3-5" as fintech C++, regardless of systems quality | Pivot |
| The skeleton is domain-agnostic: ~80% of the spec maps component-for-component (§4) | Pivot is cheap |
| AI spine becomes *more* coherent: decoder = backend, batch sweep = core benchmark, continuous batching = centerpiece instead of know-only vocabulary | Pivot strengthens the spine |
| What is lost: the 15c3-5 compliance narrative and tight Orderbook coupling | Accepted; Orderbook replay survives as one workload flavor |
| What is lost: UDP gap/snapshot recovery is a smaller learning item than it protects | Accepted; recovery drops to stretch status |
| What is lost: the quant resume drops from three projects to two | Accepted and stated plainly: Orderbook + Backtester carry that page, and the gateway still reads as low-latency systems evidence on it |

**Framing rule:** the lead project must be domain-legible to both audiences. Low-latency systems discipline reads the same in any domain; financial regulation does not.

## 3. What survives unchanged

- Phase 0 (OSTEP/CiA/Drepper/McKenney reading spine, harness, false-sharing toy, SPSC ring v1/v2) — the dominant block, untouched
- [`30-perf-playbook.md`](30-perf-playbook.md) — claim discipline transfers verbatim
- [`03-explain-loop.md`](03-explain-loop.md) — cold recall, prediction ledger; *more* important now
- Operating rules 1–6 (bracket rule, honesty rule, weekly review, resume-is-output, hardware, session setup)
- Never-cut list (Phase 0, correctness gate, ablation, zero-alloc verification, replay determinism) — with the scheduler correctness gate added
- [`40-deepening-queue.md`](40-deepening-queue.md) — cache/NUMA work on Orderbook and Backtester is domain-independent and stays
- AI spine steps B, C, D (dataset pipeline, training literacy, evaluation rigor) — untouched, still Backtester work
- Step A (CUDA + kernels bench) — content untouched, now built inside the inference engine repo; roofline and bandwidth reasoning become the decoder's and scheduler's vocabulary

## 4. Component mapping (what becomes what)

| Risk engine component | Gateway equivalent |
| --- | --- |
| UDP multicast feed simulator | Open-loop load generator: recorded request streams replayed at configurable arrival rates |
| SPSC order-entry queue | SPSC request dispatch queues |
| Position/notional/price-band checks | Admission control: token-bucket rate limits, per-tenant concurrency caps — O(1), sub-microsecond, ablated vs. a mutex baseline |
| Kill switch (restart-safe) | Circuit breaker + load shedding (restart-safe) |
| Lock-free position ledger | Lock-free request/accounting ledger, restart-safe metrics |
| 5µs tick-to-trade budget | **Two** budgets, not one: TTFT and TPOT, each with a stated SLO under a stated offered rate. The admission decision is the knob between them — admitting a request grows the batch, raising TPOT for every resident request and setting TTFT for the newcomer. See §6 |
| Fuzz / differential / TSan / replay gates | Identical — the correctness machinery is domain-blind |
| Phase 4 zero-alloc ablation | Identical, including `LD_PRELOAD` verification |

## 5. What changes, phase by phase

| Phase doc | Change |
| --- | --- |
| [`10-phase-0-reading.md`](10-phase-0-reading.md) | None to the reading. The SPSC capstone's framing note points at the dispatch queue it becomes |
| [`11-phase-1-core.md`](11-phase-1-core.md) | Risk checks → admission control (§4); ledger becomes the accounting ledger; domain reading (15c3-5, RTS 6) → serving-side policy reading. Fuzz/TSan/differential/replay gate kept verbatim |
| [`12-phase-2-microarch.md`](12-phase-2-microarch.md) | None — the experiments are on the admission path, which has the same shape |
| [`13-phase-3-loadgen.md`](13-phase-3-loadgen.md) | Multicast feed handler → open-loop load generator + TCP request stream. Sequencing keeps a staleness-detection story; **snapshot recovery drops to stretch**. Overload run gains the shedding policy |
| [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md) | None |
| [`15-phase-5-docs.md`](15-phase-5-docs.md) | README/design/proof pack reframed for serving; industry-context Q&A becomes serving context; boundary script updated |
| **NEW [`16-phase-6-scheduler.md`](16-phase-6-scheduler.md)** | The centerpiece (§6) |

**Sequencing (v4).** The old order amendment existed to run AI work ahead of the risk engine's grind; with the tracks collapsed there is nothing to run ahead of, so the amendment is superseded rather than adopted. The v4 order is: **Phase 0 → A (kernels) → G (decoder) → Phases 1–5 (gateway) → Phase 6 (scheduler)**, with steps H, B, C, D continuing to run as Orderbook/Backtester work whenever their own gates are green. This puts two finished artifacts — the kernels bench and the decoder — in the repo before the gateway needs them, and makes the decoder's correctness gate the last cheap moment to stop.

## 6. New material: the scheduler (phase 6)

Continuous batching was previously "know-only, deliberately skipped." It is now the heart of the project, and it is harder than the component it replaces — a concurrent system with admission, eviction, and fairness, not a per-message filter. Budget accordingly; the full build spec is [`16-phase-6-scheduler.md`](16-phase-6-scheduler.md).

Decisions taken at adoption, so phase 6 does not silently choose them:

- **Ladder, not monolith.** Rung 0: the naive reference scheduler (one request at a time, no batching) — the differential oracle. Rung 1: iteration-level continuous batching with KV-budget admission. Rung 2: chunked prefill. Each rung is measured before the next is written; a rung that buys nothing is a finding worth keeping. Same idiom as the GEMM ladder.
- **Chunked prefill: yes, as rung 2.** It is the modern design (vLLM V1, TensorRT-LLM, SGLang) and the only way the TTFT/TPOT tradeoff is real rather than nominal — without it a long prompt stalls the whole batch. It is also the strongest "why is this hard" answer the project produces.
- **KV allocation goes behind an interface from day one**, even while the implementation is contiguous. Paging can then replace the allocator in phase 6 without touching attention. Retrofitting this later means rewriting the decoder's memory layout under load.
- **The differential invariant is output equality**, not merely "zero divergences": batching is a scheduling change, not a numerical one, so token outputs must be identical across batch compositions and across thread counts. Deterministic replay is the same claim under the replay harness.
- **Admission projects the worst case** — prompt length + `max_new_tokens` — and rejects at capacity rather than preempting. Preemption (recompute or swap) is a stretch rung, only if the ladder goes that far. Reject-at-capacity is the direct analog of the risk engine's fail-safe reject, and it is defensible in one sentence.
- **Scheduler overhead per admission decision is measured and reported**, because "admission control adds [N]ns to a decision that costs [M]µs" is the sentence that makes the gateway a systems project rather than a wrapper.

## 7. AI spine edits (`20-ai-extension.md`)

| Item | Edit |
| --- | --- |
| Order amendment (H→B→C→D→A→G ahead of the core grind) | **Superseded** — the tracks collapsed into one sequence (§5). Delete the section; do not adopt it |
| Steps E, F (inference stub, batch sweep) | **Absorbed into the gateway.** E's *build* (ONNX Runtime on the hot path) is cuttable — cut item 2 — and E's *learning* (per-execution-provider determinism) survives as a phase-6 writeup topic. F's sweep becomes a phase-6 benchmark |
| Step G (decoder) | Decoder becomes the gateway's in-process backend behind a pluggable interface. Cut-list change: remove "no continuous batching" from G — batching lives in the gateway; the decoder stays single-stream-correct first (token-for-token vs. llama.cpp), the gateway adds concurrency |
| Continuous batching (was "deliberately skipped") | **Built**, in phase 6 |
| Step A (CUDA/kernels bench) | Content unchanged; built inside the inference engine repo, where the decoder's achieved-bandwidth claim sits against the ladder's measured ceiling on the same machine |
| Steps B, C, D | Unchanged |
| New reads (§8) | Added to the extension spine's curriculum |

## 8. Reading additions (the real new learning, ~2–3 weeks)

The pivot adds little net study — the systems spine already covers the mechanics. The gap is scheduling *policy*, previously and deliberately vocabulary-only:

| Resource | Status | Covers |
| --- | --- | --- |
| **Orca** (Yu et al., OSDI 2022, iteration-level scheduling) | NEW — real read | Origin of continuous batching; the scheduler's reference design |
| **vLLM / PagedAttention** (Kwon et al., SOSP 2023) | Upgraded: skim → real read | KV paging; the admission-budget mechanism |
| **Pope et al., "Efficiently Scaling Transformer Inference"** (Google, 2022) | NEW — real read | Batch economics, KV-memory math; short and quotable |
| Circuit breaker / load shedding (Fowler's article; Nygard, *Release It!* stability patterns) | NEW — evening-level | Breaker and shedding patterns |
| Little's Law | NEW — one section, applied immediately in the batching curves | Queueing vocabulary for the sweep writeup |
| ONNX Runtime determinism / execution-provider docs | Retained from old step E | Per-provider determinism; the phase-6 writeup source |
| **Sarathi-Serve** (Agrawal et al., OSDI 2024) | NEW — read before phase 6 rung 2 | Chunked prefill: the mechanism, and what it costs |
| **vLLM scheduler + block-manager source** | NEW — reference implementation | The design's actual data structures and preemption policy; read the way step G reads llama.cpp |
| **HdrHistogram library docs** | NEW — ~30 min | Percentile mechanics: recording, rescaling, merging — whether p99.9 is actually right |
| **Deterministic replay / simulation-testing writeup** (Wilson's talk; Antithesis posts) | NEW — ~1 hour | Taxonomy of nondeterminism sources; what a "bit-identical" claim must pin |
| **McKenney, *Is Parallel Programming Hard* — seqlock + RCU chapters** | NEW — targeted chapters | Phase 1's lock-free config publication, named and chosen rather than improvised |
| **One fair-queueing treatment** (WFQ / deficit round robin) | NEW — ~1 hour | Phase 6's cross-tenant fairness policy, named with a starvation boundary instead of improvised |

**Explain-loop rule (mandatory before phase 6 ships):** cold, spoken, no notes — explain Orca's iteration-level scheduling and vLLM's paging, *why* each beats the naive design, and reconcile both with your own scheduler's choices. This is the feedback substitute for having no mentor; run it seriously.

## 9. Resume edits (`90-resumes.md`)

- **Appendix A (quant):** risk-engine bullets deleted; the pre-pivot spec is preserved in git history (everything before the v4 commit) as a future cycle. Quant page = Orderbook + Backtester, plus the gateway only if it is framed as low-latency systems evidence rather than serving work. The quant track was never dependent on the third project.
- **Appendix B (AI-infra):** the inference engine is the lead project block — gateway, scheduler, decoder, and kernels under one heading, all bullets bracketed until measured. Narrative line: *"Built a low-latency inference serving gateway from the metal up — SPSC dispatch, NUMA-pinned, zero hot-path allocations — and served a from-scratch C++17 decoder through it; TTFT/TPOT measured under concurrent load with coordinated-omission-correct methodology."*
- Boundary script updated: "my work is the serving and performance layer; the model is a placeholder — a deliberate boundary, not a gap."

## 10. Revised cut order (if time compresses)

The master's list is canonical; reproduced here so the two cannot drift.

1. SIMD batch checks (still a stretch; the design-doc Q&A bullets survive the cut)
2. **Second backend (ONNX Runtime) — cut before any core item**; the decoder backend alone carries the story
3. Half of Phase 2's experiments (keep NUMA + false sharing; branchless and huge pages are the cut candidates)
4. Phase 6 rung 2 (chunked prefill) — the scheduler ships at rung 1 with the ladder documented and rung 2 named as designed-but-unbuilt
5. Phase 6 preemption rungs (recompute/swap)
6. Load-generator realism (keep one recorded stream + one synthetic flavor)
7. Snapshot-recovery stretch (already demoted)
8. Step G's stretch items (sampling variants, GPU path) — and cut A5 (the quantized kernel) before cutting the GEMM ladder

**Never cut:** Phase 0 reading and the SPSC ring; correctness gates (fuzz/TSan/differential/replay); the scheduler correctness gate; ablation; zero-alloc verification.

## 11. Risk register

| Risk | Mitigation |
| --- | --- |
| Scheduler is harder than the risk checks it replaces | Orca/vLLM as reference designs; rung-0 naive reference for differential testing; phase 6 gated on phase 1 being green; chunked prefill is rung 2, not rung 1 |
| No external feedback on new material | §8 explain-loop rule is a shipping gate, not a suggestion |
| Timeline slips while a hiring window opens | Kernels bench and decoder both ship before the gateway does — two finished artifacts stand alone. Never present planned work as built (bracket rule) |
| Scope creep back toward a product | The non-goals list in `00-master.md` is load-bearing: no OpenAI-compatible API, no multi-model serving, no HTTP/2 stack, no auth |

## 12. Repo mechanics (monorepo, as built)

Nothing is built (6 commits, docs only), so the pivot is cheap.

1. **Rename** `pre-trade-risk-engine` → `inference-engine`; GitHub keeps redirects. The rename is a separate step from the doc adoption — it changes the working directory and the remote.
2. **Layout** — one repo, one CMake build, one CI, because the decoder's bandwidth claim needs the kernels ladder's measured ceiling on the same machine and the same harness, and because one README can present the claims as one system:

```
inference-engine/
├── README.md            ← the proof pack: architecture diagram + claims table,
│                          every number linked to the bench that produced it
├── src/
│   ├── gateway/         ← admission control, SPSC dispatch, breaker, ledger
│   ├── scheduler/       ← continuous batching, KV-budget admission  (phase 6)
│   ├── decoder/         ← transformer decoder, KV cache behind an allocator interface
│   └── kernels/         ← GEMM ladder, roofline figures, the quantized kernel
├── bench/               ← all benchmark harnesses + measured tables
├── docs/                ← this plan (rename to design docs as they ship)
└── tests/               ← fuzz, differential, replay gates
```

3. **This file** lives as `docs/04-inference-engine-pivot.md`; the master's scope section references it.
4. Phase docs 11, 13, and 15 are edited per §5; `16-phase-6-scheduler.md` exists before phase 1 starts, so the scheduler's hooks (KV allocator interface, backend interface) are in the phase-1 design.
5. `20-ai-extension.md` is edited per §7. The pre-pivot risk-engine spec is not deleted — it lives in git history before the v4 commit, and the 15c3-5 angle remains a quant-only differentiator worth a future cycle.
6. Commit history carries the arcs: kernels ladder rung-by-rung, then the decoder with its correctness gate, then gateway, then scheduler. Nothing lands before the thing it depends on.
