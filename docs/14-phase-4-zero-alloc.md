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
- **AI hook:** keep the benchmark harness able to swap the synthetic strategy for a future decision backend, and to record batch size as a first-class dimension. Do not build the backend here; preserve the extension point (steps E+F in [`20-ai-extension.md`](20-ai-extension.md)).

- **Ablation study — the centerpiece.** One table, every "what did X buy you?" answered:

| Configuration | p50 | p99 | p99.9 | IPC | Branch-miss | dTLB-miss |
| --- | --- | --- | --- | --- | --- | --- |
| Baseline (naive) |  |  |  |  |  |  |
| + cacheline isolation |  |  |  |  |  |  |
| + branchless |  |  |  |  |  |  |
| + huge pages |  |  |  |  |  |  |
| + NUMA pinning |  |  |  |  |  |  |

Ablation discipline: one fixed workload and one focused `perf stat` event set across all rows; A/B/A or interleaved repetitions; every raw output and environment log under `bench/results/` per the [playbook](30-perf-playbook.md). IPC is explanatory, not the primary claim. Rows that measure as null stay in the table — a null with a mechanism is a finding, and removing it is how a table becomes marketing.

**Never cut:** correctness (Phase 1), ablation, zero-alloc.

**Artifact:** `bench/` with ablation table + histograms.
**Claims this phase should earn:** *"Verified zero hot-path allocations via global new/delete interposition; ablation study quantifying each micro-arch optimization's p99.9 contribution; integrated with custom matching engine driven by multicast feed."*

**Next:** [`15-phase-5-docs.md`](15-phase-5-docs.md).
