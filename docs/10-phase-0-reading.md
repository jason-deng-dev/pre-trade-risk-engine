# Phase 0: Pre-Project Reading — *do not skip*

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`01-reading-map.md`](01-reading-map.md) · [`02-foundations.md`](02-foundations.md) · [`30-perf-playbook.md`](30-perf-playbook.md) · [`03-explain-loop.md`](03-explain-loop.md)

**Baseline assumption: you arrive with zero concurrency background and no SPSC queue in your backtester.** Everything is built here from nothing. This block is front-loaded because atomics/memory-ordering bugs are correctness bugs in a risk engine — you can't ablate your way out of a data race.

The shape: one linear sequence below — OSTEP spine with per-chapter verdicts, CiA interleaved at the parallel marker, supplements in their slots. Your Anki deck handles retention.

**📖 Read — in execution order:**

1. **OSTEP 1–2** — read fully
2. **OSTEP 3** (dialogue) — optional (test the device; keep if it refreshes you)
3. **OSTEP 4–6** — read fully · **ch. 6 is the anchor**: LDE (context switches, traps) is the mechanism behind every data race
4. **OSTEP 7** — summary level (~15 min: intro + summary)
5. **OSTEP 8** — partial (intro + summary + §8.1; skip proofs/tuning)
6. **OSTEP 9–10** — summary level (~15 min each: intro + summary)
7. **OSTEP 11** — skip (summary dialogue)
8. **START CiA ch. 1–3** — read fully, **∥ in parallel with OSTEP 13–23 below** (orthogonal content, deliberate change of pace): what concurrency is, launching threads, mutexes/races/deadlock — your vocabulary, first tools, and the "spotting race conditions in interfaces" eye the correctness gate needs
9. **OSTEP 13–15** — read fully
10. **OSTEP 16** — narrative pace (no cards, no re-reads; exists to explain why paging won)
11. **OSTEP 17–21** — read fully · ch. 21's page-fault mechanics are the whole zero-alloc story
12. **OSTEP 22–23** — skim
13. **CiA ch. 4** — full read (part of the full-book pass): CVs/futures at comprehension pace; **§4.3 with full attention** (`steady_clock` vs `system_clock` — correctness for latency measurement and the rate limiter)
14. **OSTEP 28** — read properly, NON-NEGOTIABLE (test-and-set, CAS, LL/SC, fetch-and-add, spinlock costs, futex/two-phase locks — the hardware foundation under `std::atomic`, the ring, and the deep interview chains; CiA has no equivalent)
15. **Preshing's two memory-ordering posts** — optional on-ramp (~1 hr): *The Synchronizes-With Relation*, *Acquire and Release Fences*
16. **McKenney, *Memory Barriers* §1–4** — read (the store-buffer model underneath Ch. 5)
17. **CiA ch. 5 — FULL read incl. 5.3** (synchronizes-with, happens-before, orderings, release sequences, fences). Budget 2–3× normal time for 5.3 — it's the densest material in the book and the one real wall; re-reading at Phase 1 (kill switch/ledger on the desk) still wins
18. **SPSC ring v1→v2** — now unlocked: build it here (v1 all-`seq_cst`, TSan; v2 relax to acquire/release with a written justification per change). Phase 0's capstone, arrived at with Ch. 5 fresh
19. **CiA ch. 6** — full read: design *reasoning* with attention (§6.1 guidelines); the lock-based implementations are reference material — comprehend, don't type
20. **CiA ch. 7** — full read: **7.1 + 7.3 properly** (definitions; the guidelines: seq_cst prototyping, reclamation, ABA, busy-wait); **7.2 at literacy depth** (hazard pointers, reference counting — know them, you'll cite why your bounded ring needs neither)
21. **CiA ch. 8** — full read: **8.2 read twice** (cache ping-pong, false sharing, NUMA distance, oversubscription — the C++-side view of your Phase 2 experiments; first pass now, second pass when Phase 2 starts); 8.3–8.4 normal; **8.5 at comprehension depth** (know exactly what parallel `for_each` does — your refusal answer)
22. **CiA ch. 9** — full read: thread pools; weight the *concepts* (queue contention, work stealing) over the implementations — you'll contrast pools with your SPSC model in interviews
23. **CiA ch. 10** — full read (short): `std::execution` policies — know precisely what you're refusing and why
24. **CiA ch. 11** — full read: designing for testability — read right before the correctness gate (Phase 1 Step 5), where you'll live its material
25. **OSTEP 33** — skim (event-loop contrast — vocabulary for why your feed handler spins)
26. **OSTEP extras** (~45 min): §29.1 approximate counters · one semaphore card (ch. 31)
27. **Stokes, *Inside the Machine*** — optional but recommended (~3–4 evenings; before Drepper): concept chapters fully, skim the Pentium 4/PowerPC case studies, never quote its (2006) implementation details
28. **Drepper §1–3** — read fully (~60 pp; Foundation #1 first half — Phase 2's experiments land on this)
29. **COD structured read, part 1 — alongside Drepper §1–3** (read *after* each matching Drepper section; second sources reinforce, they don't build): Ch. 1 (performance/Amdahl); 5.1–5.4 (rigorous caches); 5.10 (MESI — the protocol behind false sharing and atomic costs). Rest of the COD skip-list lives in [`01-reading-map.md`](01-reading-map.md); Phase 2 and Phase 4 slots below; GPU chapters went to the AI extension spine (step A)
30. **Tools** (~2 hr): work through the [`perf stat` Playbook](30-perf-playbook.md) (Sections 1–3 first, then the learning layers), `man perf-stat`, and the Google Benchmark README → then the Phase 0 builds below (harness + false-sharing toy, if not already done)

**Recorded for completeness — do not schedule:** OSTEP 24–25 (dialogues) · 26–27, 29–32 (covered by CiA in C++ form; extras in step 26 only) · 34 (dialogue) · persistence 36–51 (optional 2-hr skim of 36/39/42; rest cut) · security 52–57 (skipped entirely).

**Anki discipline:** card what you'll use within weeks, not everything.

**🔨 Build (interleave — start the harness and the false-sharing toy once you finish the memory chapters; the SPSC ring stays the capstone, after the reads):**

Three artifacts, in increasing order of difficulty. The first two exist to make the third measurable.

- **The measurement harness.**
  - *What:* a `perf stat` + Google Benchmark wrapper that produces machine-readable output with reproducible, seeded workloads.
  - *Why:* every claim you make for the next five phases depends on this being trustworthy. Build it before you need it, not while you're also trying to interpret a result.
  - *Reach for:* the [`perf stat` Playbook](30-perf-playbook.md)'s Section 3 method — fixed event sets, seeded input, environment metadata logged alongside every run.
  - *Learn:* that measurement is infrastructure, and that an untrustworthy harness makes every later number worthless regardless of how good the code is.

- **The false-sharing toy** — your first interview prop. *What:* two threads updating separate variables that happen to share a cache line, measured before and after you separate them. *Reach for:* `perf stat` for cycles and cache events, `perf c2c` where available, the controlled padding ablation as the actual evidence when the generic counters stay ambiguous. *Learn:* that coherence traffic is invisible to the counters you'd expect to show it — and why the ablation, not the counter, carries the claim.

- **The SPSC ring** — your first real concurrency artifact, carried into Phase 1.
  - *What:* a bounded, pre-allocated single-producer/single-consumer queue.
  - *Why:* it's the transport under the ledger, and the place where memory ordering stops being theory. CiA Ch. 5 arrives just before it, deliberately.
  - *Reach for:* **v1 all-`seq_cst`** — obviously correct, and CiA 7.3's own prototyping guideline; then **v2 relaxes to acquire/release one change at a time**, re-running TSan and the tests after each, with a written justification for every relaxation in the commit message. That justification document is interview gold.
  - *Learn:* which orderings are load-bearing and which are superstition. **Note what you dodged:** a bounded pre-allocated ring has no node reclamation, so hazard pointers and ABA (CiA 7.2) don't apply — know *why*, because you'll be asked why you didn't need them.

**Transferable benchmark hygiene** (banked from the OSTEP ch. 19/14 homeworks — apply to every number you ever produce):
- **Observable side effects:** the compiler deletes loops whose results nobody reads — print/accumulate results, or use Google Benchmark's `DoNotOptimize` (ch. 19 hw Q5).
- **`steady_clock`, never `system_clock`;** warmup iterations separated from measured ones; enough repetitions that timer precision (ch. 19 hw Q1) can't distort the mean.
- **First-touch:** freshly allocated pages cost demand-zeroing on first access (ch. 19 hw Q7) — pre-touch buffers before timing, or your first iteration lies.
- **Pin before measuring** (`sched_setaffinity` — ch. 19 hw Q6): unpinned threads bounce cores and inherit foreign TLB/cache state.

**🔓 Unlocked — Backtester (do this after Phase 1, while the ring is fresh):** port this exact ring into the backtester as the Monte Carlo job/result transport and reproduce the bit-identical-hash property. Your resume claims lock-free SPSC queues with specific numbers — this makes the claim true, backed by the same per-memory-order justifications you just wrote down. Protects the credibility of every other number on the page. Queued in [`40-deepening-queue.md`](40-deepening-queue.md).

**✅ Verify:** Can you explain why x86 makes acquire/release nearly free but `seq_cst` needs a full fence — now from *your own ring's* disassembly, not just the books? Can you explain the toy's cache-miss spike mechanism in two sentences? Is your ring v2 TSan-clean with a written justification for every memory order?

**Then:** run the transfer checkpoint in [`00-master.md`](00-master.md) out loud first, then move to [`11-phase-1-core.md`](11-phase-1-core.md).
