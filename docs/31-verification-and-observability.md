# 31 — Verification & Observation (sanitizers, CI, and the cost of watching)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Companion docs:** [`30-perf-playbook.md`](30-perf-playbook.md) (whose rule "sanitizers are correctness, never performance" this operationalizes) · [`11-phase-1-core.md`](11-phase-1-core.md) (the fuzz/TSan/differential gate this runs continuously) · [`41-network-ingestion.md`](41-network-ingestion.md) (the one place untrusted bytes enter the portfolio) · [`15-phase-5-docs.md`](15-phase-5-docs.md) (where the results land in the proof pack)

**Why this doc exists:** the plan declares its correctness gates never-cut in three separate places, and then never says how they run. A gate that only exists on the machine where it was written is a claim about a moment, not about the code. This doc turns the gates into a matrix that runs on every push, and then makes the second, sharper point: **every tool that watches a running system costs something**, and on a sub-microsecond path that cost is a design constraint, not a footnote.

---

## Part 1 — The sanitizer matrix (all three repos)

Same matrix for Orderbook, Backtester, and the inference engine. It is cheap, it is the single highest signal-per-hour item available outside the engine spine, and none of the three repos has any CI at all today.

| Tool | Catches | Blind to | Cost |
| --- | --- | --- | --- |
| **ASan** (+LSan) | heap/stack/global buffer overflow, use-after-free, double-free, invalid free, leaks; use-after-return with `ASAN_OPTIONS=detect_stack_use_after_return=1` | uninitialized reads, data races | ~2× slowdown, large shadow-memory footprint |
| **UBSan** | signed overflow, bad shifts, misaligned access, invalid casts, VLA misuse, null deref | anything that is merely *wrong*, not undefined | ~free (a few %) |
| **TSan** | data races, lock-order inversion, use of destroyed synchronization | races that the run never schedules; races in hand-rolled synchronization it cannot recognize | 5–15× slowdown, heavy memory |
| **MSan** | uninitialized-memory reads | **everything your dependencies do** — see below | ~3× |
| **libFuzzer** | whatever the sanitizer underneath can see, over generated inputs | inputs your mutator cannot produce | the sanitiser's cost, times the corpus |

**Rules that make the matrix real:**

1. **UBSan must abort in CI.** By default it warns and continues, which means a green build that printed an undefined-behaviour line nobody read. `-fno-sanitize-recover=all` or it does not count.
2. **ASan and TSan are separate builds, never combined** — they are mutually exclusive, and TSan's memory overhead means you run it on the concurrency-focused targets, not the whole world.
3. **MSan only if you build the world.** It requires *every* dependency — including libc++ — instrumented; against a stock libstdc++ it reports false positives inside library code and burns a day of your life. Practical rule: ASan+UBSan everywhere, TSan on the concurrent targets, and MSan either in a purpose-built container or not at all. Write the limitation down; knowing why MSan lies is worth more than a green MSan badge you cannot trust.
4. **Never benchmark under a sanitizer.** Different allocator, different timing, different memory layout. ASan numbers are not latency numbers; the perf playbook's rule already says this, and CI is where it gets violated.
5. **TSan and the SPSC ring.** Hand-rolled lock-free code may be flagged or may be invisible to TSan depending on how the synchronization is expressed; annotations (`ANNOTATE_HAPPENS_BEFORE`/`_ACQUIRE`) exist for exactly this. If annotation is not worth it, say so explicitly in the design doc and lean on the other two legs — the deterministic replay claim and the fuzz machinery — rather than quietly accepting a TSan exclusion.

**Fuzzing under the sanitizer is the point.** The corpus already exists in spirit (the [10M]-sequence admission fuzz, the request-sequence fuzzer, and — new with [`41-network-ingestion.md`](41-network-ingestion.md) — a frame deserializer that eats untrusted bytes). Wire it as: short fuzz run on every PR, long run nightly, corpus committed, every crash kept as a regression case. That converts "fuzzed with zero divergences" from a past event into a property that holds on every commit, which is a strictly stronger sentence on the page.

**Wiring (per repo, ~a day each):** a GitHub Actions matrix over `{asan+ubsan, tsan}`, a third job for the fuzz corpus, `sccache`/`ccache` plus a CMake preset per sanitizer mode (the repos already have `CMakePresets.json` — add a preset rather than inventing a second build system), and a README line linking the run. No Bazel, no remote execution, no build-system migration: the signal here is that the matrix exists and is enforced, not which tool drives it.

**Claims this section earns (proof-pack sentences, not resume bullets):** *"CI matrix green across ASan/UBSan and TSan on every push; the [N]-sequence fuzz corpus runs under ASan nightly with zero findings since [date]."*

---

## Part 2 — The cost of watching (eBPF, and where it is legitimate)

### The problem this section exists to state

You cannot observe a sub-microsecond hot path without changing it. Instrumentation has a price — a trap, a probe, a map update, a cache line touched — and on a 200ns decision the price is measured in multiples of the thing being measured. So the question is never "how do I trace this?" It is **"how much does this observation cost, and where is observation still legitimate?"**

### What eBPF is genuinely for here

Not the hot path. The things your own code cannot see about itself:

- **Kernel involvement, proven absent**: syscall counts, `sched:sched_switch` (off-CPU time), CPU migrations on a pinned thread, page faults. This is how you *prove* the zero-alloc and zero-syscall claims rather than asserting them — and it corroborates the rung deltas in [`41-network-ingestion.md`](41-network-ingestion.md) from the outside.
- **Lock contention and futex behaviour** on the reference implementation (the mutex-based oracle) — the "what the obvious implementation actually costs" number, observed from outside the process.
- **The observation-cost measurement itself** — see below. It is a legitimate, publishable result.

### The experiment: measure the observer

Design it like any other ablation, because it is one:

1. Pick a function on the hot path with a known cost (`perf stat` cycles/op over [N] iterations, pinned, no probes).
2. Attach a bpftrace uprobe on entry, then entry+exit; re-run the *same* harness.
3. Report the delta in cycles/op and in p99 — the perturbation, in nanoseconds, on a path you already know the length of.
4. State the conclusion in one sentence: at this cost, in-band tracing of this function is or is not viable, and what the production alternative is.

Expect the answer to be "not viable" on a sub-microsecond path, and expect that to be the *interesting* result rather than a failure. **The alternative is in-band:** an append-only event log (the pattern the Orderbook already uses) plus a histogram published to a reader through a seqlock — the exact mechanism Phase 1 already assigns you the McKenney reading for. Read it as one continuous idea: the same seqlock that publishes config to a hot path can publish a latency histogram out of one, and the cost is a store plus a fence. The same snapshot then feeds the metrics exporter ([`15-phase-5-docs.md`](15-phase-5-docs.md)) — the hot path writes counters, the exporter reads a consistent snapshot, and no scraper ever touches the decision path.

**Reads (all free):** bpftrace one-liners and the `bcc` tool list first (an evening, and enough for everything above); then the BPF verifier constraints (bounded loops, 512-byte stack, no arbitrary pointers) and per-CPU vs. hash maps, which explain why the cheap probes are cheap; `ringbuf` vs. perf buffer if you go beyond counters. Half a day total, all of it just-in-time.

**Claim this section earns:** *"Measured uprobe overhead at [N]ns on a [M]ns decision path ([P]% perturbation); replaced in-band tracing with a [K]ns seqlock-published histogram and reserved eBPF for syscall, scheduler, and off-CPU observation, where it cannot perturb the path."*

**Drills:** why can't you just add a timer around the hot path? What does a uprobe cost, and what does that cost *mean* on a 200ns path? What can eBPF see that your process cannot see about itself? Why a per-CPU map instead of a shared one? What does the verifier refuse to let you do, and why does that make the program safe to run in the kernel at all?

**Boundary:** this is observability literacy plus one honest measurement, not a production telemetry stack. No OpenTelemetry, no metrics pipeline, no dashboards. Say so if probed; the finding is the cost, not the infrastructure.

---

**Sequencing:** Part 1 is the "in a gap" work — it needs no hardware and unblocks nothing else, so it fills a half-week between engine phases and should be done **before** the Phase 1 fuzz gate is claimed as shipped. Part 2 is engine-side, runs after the gateway exists (Phase 4's zero-alloc verification is the natural pairing — same subject, two observation methods), and its measurement needs a pinned, quiet machine and the playbook's A/B/A discipline to be believable.
