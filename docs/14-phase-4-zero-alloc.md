# Phase 4: Zero-Alloc Verification + Ablation

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`30-perf-playbook.md`](30-perf-playbook.md) §7 (claim discipline) · [`03-explain-loop.md`](03-explain-loop.md) (predictions) · [`40-deepening-queue.md`](40-deepening-queue.md) (the Orderbook unlock below)

**Shape of this phase:** one proof, one table. Prove the hot path never allocates, then account for every optimization you've made.

---

**Step 1 — 📖 Read:** cppreference on `operator new`/`operator delete`, **plus the object-model pages: object lifetime, strict aliasing / type punning, and placement new** (~1 hour). This is Foundation #5 made concrete: interposition hooks the allocation functions, and any future pool/arena work (including the PMR deepening on your Orderbook) leans on lifetime and aliasing rules. Plus **COD 2.12** (the compile→assemble→link→load pipeline behind symbol interposition; ~30 min — connects the OS view to the ELF mechanism) and one article on ELF symbol interposition (~20 min). That's all this phase needs — the concepts are already built.

**🔓 Unlocked — Orderbook:** this material is exactly what a pooled, intrusive per-level FIFO needs — nodes placed into a pre-committed arena rather than allocated, with lifetime and aliasing rules respected. Two artifacts from it: (1) re-run the 1M-message benchmark and cite the p99 tightening — the zero-allocation story then spans two projects and reads as an engineering principle, not a project artifact; (2) **the determinism check**: fills must be bit-identical before/after the rewrite on the same golden replay — same fill sequence, same partial-fill behavior. That turns the rewrite into a correctness story, not just a perf one. Queued in [`40-deepening-queue.md`](40-deepening-queue.md).

**Step 2 — 🔨 Build:**

- **Zero-allocation proof.**
  - *What:* every allocation in the process routed through your own accounting, with the benchmark failing if a hot path allocates after warmup.
  - *Why:* an allocation on the hot path is an unbounded latency event wearing a disguise — a lock, a page fault, a syscall, a cache eviction, all at once, at a moment you don't choose.
  - *Reach for:* global operator interposition rather than a custom allocator — interposition catches the hidden allocations inside third-party code, which is exactly where they hide.
  - *Learn:* that "we don't allocate" is a claim requiring an instrument, and that the interesting failures are in code you didn't write.
  - **Boundary of the instrument (added 2026-09-28): interposing `operator new`/`delete` does not see raw `malloc`/`free`.** Any C library, and Google Benchmark itself, allocates through C functions that never touch the C++ operators — so a hidden allocation on the measured path would pass the gate while the gate reports clean. Close it rather than document it: interpose `malloc`/`free`/`realloc` as well, in the same preloaded library as the operators, and run the check as an `LD_PRELOAD` build so dynamically-linked callers are covered too (a `--wrap` at link time only catches calls resolved inside your own binary). Cross-check at least once with a heap profiler on the full run. The claim you are buying is "the process did not allocate on the hot path," and that claim is only as strong as the widest hook you installed.

  - **What the proof does not cover:** it shows nothing *new* was allocated — not what happened to the pages you already hold. Pair it with `page-faults` and RSS under live load, and answer this for the design doc: after warmup, why doesn't RSS shrink when the process frees memory? (Pages go back to the allocator and become page cache; the OS reclaims them only under pressure.) Twenty minutes of `vmstat 1` alongside the running engine answers it on your own machine. That's the honest boundary on the zero-alloc claim — *no allocation* is not the same as *no page traffic*, and a reviewer who knows the difference will ask.
- **AI hook:** keep the benchmark harness able to swap the synthetic workload for the real decoder behind the model-backend interface, and to record batch size as a first-class dimension. Do not build the backend here — the interface is defined in phase 1 and the decoder lands in step G ([`20-ai-extension.md`](20-ai-extension.md)).

- **Ablation study — the centerpiece.** One table, every "what did X buy you?" answered:

| Configuration | p50 | p99 | p99.9 | IPC | Branch-miss | dTLB-miss |
| --- | --- | --- | --- | --- | --- | --- |
| Reference (straightforward implementation) |  |  |  |  |  |  |
| Baseline (this architecture, optimizations removed) |  |  |  |  |  |  |
| + cacheline isolation |  |  |  |  |  |  |
| + branchless |  |  |  |  |  |  |
| + huge pages |  |  |  |  |  |  |
| + NUMA pinning |  |  |  |  |  |  |

Ablation discipline: one fixed workload and one focused `perf stat` event set across all rows; A/B/A or interleaved repetitions; every raw output and environment log under `bench/results/` per the [playbook](30-perf-playbook.md). IPC is explanatory, not the primary claim. Rows that measure as null stay in the table — a null with a mechanism is a finding, and removing it is how a table becomes marketing.

**Two baselines, and the difference matters.** The **reference** row is the implementation a competent engineer writes in a week: a mutex around shared state, `std::unordered_map` for the ledger, heap allocation per decision, a chain of early-exit branches. It is not a strawman — it is what the code looks like before you know what you know. The **baseline** row is *this* architecture with the optimizations removed one at a time, which is what isolates each optimization's contribution. The ablation table answers "what did each optimization buy"; the reference row answers the question a reviewer actually asks, which is "why would I want yours instead of the obvious thing." Report the final-vs-reference delta as the headline sentence and keep the ladder underneath it as the evidence.

The reference implementation is built **once**, in Phase 1, and serves two purposes: it is the differential-testing oracle for the fuzzer (see [`11-phase-1-core.md`](11-phase-1-core.md)) and row 0 of this table. Building it here instead means writing it twice.

**Never cut:** correctness (Phase 1), ablation, zero-alloc.

**Artifact:** `bench/` with ablation table + histograms.
**Claims this phase should earn:** *"Verified zero hot-path allocations via global new/delete interposition; ablation study quantifying each micro-arch optimization's p99.9 contribution; measured on the integrated serving pipeline under open-loop load."*

**Next:** [`15-phase-5-docs.md`](15-phase-5-docs.md).
