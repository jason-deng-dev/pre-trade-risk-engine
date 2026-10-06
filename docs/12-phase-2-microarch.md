# Phase 2: Micro-Architecture

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`30-perf-playbook.md`](30-perf-playbook.md) (Sections 5–6 before every experiment) · [`01-reading-map.md`](01-reading-map.md) (the COD skip list) · [`03-explain-loop.md`](03-explain-loop.md) (the prediction ledger) · [`40-deepening-queue.md`](40-deepening-queue.md)

**Shape of this phase:** controlled experiments on a machine you understand. Every claim here is a before/after with a prediction made *before* the run.

---

**Step 1 — 📖 Read:** Three sources, two foundations (#1 and #3, see [`02-foundations.md`](02-foundations.md)):

- **Drepper §4–6.4.1** — virtual memory from the hardware view, NUMA systems (§5), and the programmer-facing sections (§6.4.1 false sharing, §6.5). This completes the Drepper backbone started in Phase 0 — after this you've read the paper that replaces COD.
- **OSTEP re-skim (30 min + your Anki cards):** the full spine read is done — before the experiments, re-skim the address-translation/paging/TLB and `malloc`-internals chapters so the OS-altitude view sits fresh alongside Drepper's hardware-altitude view. Nothing here is new; any shaky card → re-read that section. (The syscall/context-switch cost model from ch. 6 — already in your deck — is the boundary condition for every "is this optimization worth it" judgment in this phase.)

Read COD as the structured second source (see the [Reading Map](01-reading-map.md) for the full skip list), sequenced *after* the corresponding Drepper sections — second sources reinforce a model; they don't build it first. The highest-value pairings: Drepper §1–3 → COD 5.1–5.4 + 5.10 (**done in Phase 0, step 29** — re-skim your highlights only); Drepper §4–5 → COD 5.7–5.8 + 6.4–6.5 (virtual memory, hyperthreading — read 6.4 *before* making pinning decisions, since physical-vs-logical core placement depends on it); Agner Fog → COD 4.6/4.8–4.11 (branch-miss mechanics); Phase 4 preview → COD 2.12 (link/load model behind symbol interposition).

**Hardware reality (9800X3D, Omarchy/Arch):** 8 Zen 5 cores / 16 threads, **single CCD → one NUMA node**, SMT 2, 96MB L3. Consequences: (a) no cross-node memory exists at home — the real cross-node number comes from one cloud measurement day (below); (b) SMT topology and the huge L3 make your machine a *better* instrument for most of this phase than a server. Everything here assumes native Linux (`perf`, `numactl`, `libnuma`, `madvise` all first-class on Arch).

**Step 2 — 🔨 Build: the cache and topology experiments.** Four questions, four before/afters in your Phase 0 harness:

- **Does isolating hot shared state pay?** Apply cacheline separation to the state two threads actually touch — the ledger, the kill signal, the rate limiter. *Reach for:* `alignas` on the shared lines, `perf c2c` when the generic cache counters stay ambiguous. *Learn:* false sharing is coherence traffic, not a cache miss — and why the ablation, not the counter, is the evidence. (The Phase 0 toy, now applied to real code.)
- **🔓 Unlocked — Backtester:** the same fix applies to the Monte Carlo queue head/tail counters — a small piece of work once you've measured the effect here, and it extends your false-sharing story across two projects.
- **What does SMT placement actually cost?** Pin the hot threads to sibling threads vs. distinct physical cores and measure the difference. *Learn:* why co-locating hot threads on siblings is a production mistake, and why your machine's 96MB L3 makes it *less* punishing than a server would. This is the at-home replacement for NUMA, and the COD 6.4 material applied.
- **Where do the cache-level crossovers sit on *your* die?** Dense-vs-hash and cache-pressure experiments have different thresholds on a 96MB-L3 consumer part than on a Xeon. Note the provenance honestly in the writeup — "measured on a 96MB-L3 consumer die" is an interesting result, not a caveat. (Reasoning exercise, not a benchmark — see step 4.)
- **What is the real cross-node penalty?** Develop and unit-test the NUMA code path at home; it will correctly discover one node and pin everything there. Then rent a 2-socket bare-metal instance for one afternoon and measure same-node vs. cross-node atomics with the *same harness*. *Learn:* the ~5× remote-atomic cost your hardware physically cannot demonstrate. Annotate provenance in the writeup ("developed on 9800X3D; cross-node numbers measured on 2-socket bare metal"). This number goes on your resume.
  - Rent the machine twice if the budget allows — a cold harness on a rented box is the classic way to come home with nothing. Validate the harness against a trivial workload first, then measure.
  - (Free, for code-path validation only: boot with one NUMA node faked into two to exercise per-node allocation and pinning. Emulated latency deltas are structural, not physical — never let those numbers into a claim.)

**Step 3 — 📖 Read:** Agner Fog's optimization manual, branch-prediction section + two Daniel Lemire posts on branchless programming (~2 hours total). Plus **CiA 8.2** (data contention/cache ping-pong, false sharing, NUMA distance, oversubscription) — the C++-code-side companion to Drepper's hardware view; read it back-to-back with the experiments above. Re-read the [`perf stat` Playbook](30-perf-playbook.md)'s Sections 5–6 before running them. Also `man numa`, `numactl`.

**Plus one paper, and it is the most important read in this phase for the ablation's credibility: Mytkowicz, Diwan, Hauswirth & Sweeney, *Producing Wrong Data Without Doing Anything Obviously Wrong* (ASPLOS 2009, ~20 min).** The argument: innocuous setup choices — memory layout, environment size, alignment, the length of a path string, the order of link — can shift measured performance by 5–10% in either direction, in otherwise methodologically correct experiments. That is larger than most of the effects this phase is trying to measure. Read it before you trust any single-digit before/after, then apply it: the ablation compares *separately built binaries*, which is exactly the situation the paper describes. The playbook's A/B/A interleaving and "don't claim 2% inside 10% noise" rule are the operational defence; this paper is the reason the rule exists, and citing it when you defend the method is a strong answer to "how do you know that delta is real?"

**Step 4 — 🔨 Build: branchless, the memory-hierarchy sweep, and huge pages.**

- **Branchless decision path.** *What:* the combined risk check rewritten as straight-line arithmetic instead of a chain of early-exit branches. *Reach for:* prior prediction of `branches` and `branch-misses` before you run it — this is the experiment most likely to disappoint if you skip the prediction. *Learn:* when removing branches *loses* — build one benchmark with a forced, perfectly-predictable branch where predication is slower, and write the paragraph explaining why. That paragraph is the interview answer.
- **The memory-hierarchy sweep.** *What:* sweep the working set from L1-resident up past RAM into swap, recording per-access time (or achieved bandwidth) against footprint. *Why:* the resulting curve — L1 → L2 → L3 → RAM → the swap cliff — is the single best figure this project produces, and it's the same experiment as the TLB sweep with a longer x-axis. *Reach for:* `vmstat 1` in a second terminal for the swap regime (`si`/`so`/`free`/`swpd`); the OSTEP ch. 21 `mem` homework is this experiment already written — `mem.c` is in your `ostep-homework` checkout and it prints bandwidth per loop, so the harness is mostly "sweep the size, collect, plot." *Learn:* the hierarchy as *your machine's* numbers rather than a textbook table — and that loop 0 pays first-touch faults while loops 1+ are warm, until past RAM, where loop 1 re-faults everything loop 0 evicted. **Predict both shapes before running** ([`03-explain-loop.md`](03-explain-loop.md)).
  - **Boundary, honestly:** a beyond-RAM run thrashes the NVMe for minutes and can trip the OOM killer (`overcommit_memory=0` means the allocation *succeeds* and the process dies later — a good war story, a bad surprise). Run it from a TTY, capped; or measure the curve to RAM and label the swap regime *"read from OSTEP ch. 21, not measured."* Either is defensible. An unlabelled one is not.
- **Huge pages and TLB characterization.** *What:* apply huge pages to the hot data once the sweep above has told you where TLB pressure actually bites. *Reach for:* `madvise`, `smaps` (`AnonHugePages`) for verification, `dTLB-load-misses` for the mechanism, and the three THP modes as your controlled variable. *Learn:* why a page walk is mostly misses — the upper levels are tiny and shared across every translation, so they stay cached; leaf entries are scattered across the whole address space, so they don't. Huge pages attack both halves of that: fewer levels to walk, and each leaf entry covering more of the footprint. The elbows in the sweep curve are *your* machine's L1/L2 dTLB reach, not trivia.
  - **Pre-register the likely null:** if the hot working set already lives inside 96MB of L3, huge pages may buy nothing at the ledger. A measured null with a mechanism ("the footprint never leaves L3, so TLB pressure was never the bottleneck") is a *better* result than an invented win — and a better interview answer. Record it either way.
- **Dense-vs-hash:** half-day reasoning exercise (footprint vs. pointer chasing, cache-level crossover) written into the design doc. Not a benchmark.

**✅ Verify:** Can you explain why an always-taken branch beats predication? Why huge pages can hurt small/sparse working sets? Where the swap cliff sits on your curve — and why loop 1 costs more than loop 0 past it? Why remote atomics cost ~5× local — and why your 9800X3D can't demonstrate it (single CCD) but a 2-socket machine can?

**Skip:** prefetching (noise), AVX2 (stretch only), per-node kill-switch replication (understand it, cite it, don't build it).

**Artifact:** bench report — topology deltas, false-sharing before/after, branch-miss delta, memory-hierarchy sweep (L1 → swap cliff) + TLB curve, cross-node penalty (cloud-measured, provenance annotated), all perf-verified, all with predictions logged.
**Claims this phase should earn:** *"NUMA-aware risk path: [X]× p99 degradation on cross-node atomics (measured on 2-socket bare metal); cacheline-aligned ledger (false-sharing p99 improved [Y]×); branchless checks ([Z]% fewer branch misses, perf-verified, with documented case where branchless loses)."*

---

**🎯 Interview drills (Phase 2 ships → you answer these):**

- **Super-linear speedup**: N CPUs exceed N× when a working set fits the aggregate cache but not one core's — warm beats cold; migration re-colds. (Banked from OSTEP ch. 10 hw Q7 — concept only.) Same physics as your pinning experiment.
- **Guarantee math**: starving job needs ≥ f of CPU with quantum q → boost/rebalance interval ≤ q/f (10ms, 5% → every 200ms). One-liner, not a derivation. (OSTEP ch. 8 hw Q5.)
- Linux scheduling vocabulary: CFS = vruntime, slice = sched_latency/n with min_granularity floor, nice→weights, sleepers reset to tree-min on wake; CFS was default until Linux 6.6 → now EEVDF. (Lottery/stride = theory behind fair share; MLFQ = the other paradigm.)
- Kernel bypass: why does it exist? (your ch. 6 syscall-cost knowledge.) Huge pages: when do they HURT? Why your 9800X3D can't show remote-atomic cost but a 2-socket machine can.
- Event loop vs spinning receiver; what io_uring solved (post-dates OSTEP ch. 33).
- MESI at the protocol level (false-sharing mechanism; remote atomic ~5× local). Dense array vs hash: footprint vs pointer chasing, cache-level crossover. SIMD: gather/scatter, AVX2 vs AVX-512 downclocking, lane semantics.

**Next:** [`13-phase-3-feed.md`](13-phase-3-feed.md).
