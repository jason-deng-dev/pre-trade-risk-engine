# `perf stat` Playbook (what it is, how to learn it, and how professionals use it)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Referenced by:** Phase 0 (step 30, learning the tool), Phase 2 (Sections 5–6 before the experiments), Phase 4 (the ablation study).

## The Honesty Rule (the reason this doc exists)

For every optimization: **mechanism** (why), **measurement** (`perf stat` before/after), **boundary** (when it doesn't help). All bracketed numbers are placeholders — replace with your measured values. §7 below is this rule in operational form.

---

`perf stat` is the Linux `perf` tool's **event-counting front end**. It runs a command — or attaches to an already-running process — and counts hardware PMU events and selected OS events over the measurement interval: cycles, instructions, branches, branch misses, cache references/misses, TLB misses, page faults, context switches, and more.

Use it to answer: **“What was the CPU doing while this workload ran?”** It gives aggregate counts and ratios. It does **not** tell you which source line was slow, and it does **not** give p99 latency. The professional toolchain is:

| Tool | What it answers | Use in this project |
| --- | --- | --- |
| Google Benchmark / HDR histogram | How fast, including p50/p99/p99.9 | Primary latency and throughput claims |
| `perf stat` | How much work and what kind of stalls/events occurred | Explains *why* an optimization helped or hurt |
| `perf record` + `perf report` | Where samples landed | Hotspot localization when the slow region is unknown |
| `perf annotate` | Instruction/source-level profile | Deep dive after `record` finds the function |
| `perf c2c` | Cache-line sharing between threads | Better evidence for false sharing than generic cache counters |
| `numastat` / `/proc` / `smaps` | OS-level memory placement and huge-page state | NUMA and huge-page verification |
| ASan/TSan/UBSan | Correctness | Never replaced by performance tools |

**Rule:** `perf stat` supports a claim; it does not create one. The claim comes from a controlled experiment with a primary metric (usually cycles/order or p99 latency) and explanatory counters.

### 1. What the core counters mean

| Counter / ratio | Meaning | How to interpret it |
| --- | --- | --- |
| `task-clock` | CPU time consumed by the measured task | Use with wall time to detect descheduling or interference |
| `context-switches` | Times the task left the CPU | Should be near zero on a pinned, isolated hot-path benchmark |
| `cpu-migrations` | Times the task moved cores | Nonzero means pinning failed or the scheduler interfered |
| `page-faults` | Demand paging and memory-map faults | Should be near zero after warmup on a zero-alloc hot path |
| `cycles` | CPU cycles consumed | Primary hardware-cost metric for same-workload comparisons |
| `instructions` | Retired instructions | Measures work done; branchless code may increase this while reducing cycles |
| `IPC` | `instructions / cycles` | Efficiency clue, **not** a universal score; higher is not automatically better |
| `branches` | Retired branch instructions | Denominator for branch-miss rate |
| `branch-misses` | Mispredicted branches | High rate suggests unpredictable control flow; low rate means branchless may not help |
| `cache-references` / `cache-misses` | Generic cache activity, often mapped to LLC events | Verify the mapping with `perf list`; do not treat as universal “all cache misses” |
| `dTLB-loads` / `dTLB-load-misses` | Data TLB activity | Key evidence for huge-page experiments |
| `stalled-cycles-frontend/backend` | Pipeline stall counts, when supported | Useful hint; exact meaning is CPU-specific |
| `--topdown` metrics | Frontend/backend/bad-speculation/retiring classification | Good high-level triage on CPUs that expose top-down PMU support |

Useful derived metrics:

```text
IPC                    = instructions / cycles
branch-miss rate       = branch-misses / branches
cache-miss rate        = cache-misses / cache-references
dTLB-load-miss rate    = dTLB-load-misses / dTLB-loads
cycles per operation   = cycles / benchmark operations
instructions per op    = instructions / benchmark operations
```

Always compare counters on the **same workload, same input size, same seed, same thread count, same machine, and same CPU placement**. A 20% cycle reduction with a 3× instruction increase can still be a win; an IPC increase with higher total cycles is not.

### 2. First commands to learn

On Omarchy/Arch, install and smoke-test the tool first:

```bash
sudo pacman -S perf
perf --version
perf stat /bin/true
cat /proc/sys/kernel/perf_event_paranoid
```

If the smoke test fails because of permissions, fix the environment before building the harness. Do not work around it by silently dropping events.

Start with the default event set:

```bash
perf stat ./bench/risk_bench --seed=123 --orders=10000000
```

Then use an explicit, small event set chosen for the question:

```bash
perf stat -r 7 -x, -o results/branchless-baseline.csv \
  -e cycles:u,instructions:u,branches:u,branch-misses:u \
  ./bench/risk_bench --seed=123 --orders=10000000
```

Flags you actually need:

| Flag | Meaning | When to use it |
| --- | --- | --- |
| `-e EVENT,...` | Select events | Every real experiment; don't rely on defaults |
| `-r N` | Repeat the whole command N times | Noise control; report median and spread |
| `-o FILE` | Save output | Mandatory for every resume-linked number |
| `-x,` | CSV-style output | Easier to parse into the ablation table |
| `-p PID` | Attach to a running process | Measure the integrated serving pipeline under load |
| `-t TID` | Attach to one thread | Measure a specific pinned thread |
| `-a` | System-wide measurement | Rare for this project; needs care and permissions |
| `-C CPU` | Restrict CPU set, normally with `-a` | Topology/system experiments, not ordinary benchmark runs |
| `-- sleep 10` | Count for a fixed interval | Attach to a live process without stopping it |
| `--topdown` | Top-down PMU breakdown | First-pass stall classification where supported |
| `-v` / `--no-big-num` | Verbose or machine-friendly formatting | Debugging output or scripting |

Attach to a live pipeline for ten seconds:

```bash
perf stat -p <engine-pid> -- sleep 10
```

Discover what your CPU exposes:

```bash
perf list
perf list | grep -i dtlb
man perf-stat
```

Use `:u` when you want **user-space** cycles/instructions and want to reduce kernel noise in a hot-path benchmark. Omit `:u` when the kernel path is part of the thing being measured (for example, syscall-heavy networking experiments).

### 3. The industry-standard workflow

There is no formal industry standard that says “run exactly these `perf stat` flags.” The standard is the **performance-engineering method** around the tool:

1. **State the question before running.** “Does `alignas(64)` reduce cycles/order for two hot threads?” is testable. “Make it faster” is not.
2. **Pick one primary metric.** Usually cycles/order for micro-architecture experiments and p99/p99.9 latency for user-visible behavior. Counters explain the movement.
3. **Freeze the workload.** Same binary, compiler flags, input size, seed, thread count, and duration. Deterministic inputs first; randomized inputs only with fixed seeds.
4. **Freeze the machine state as much as practical.** Pin threads, separate warmup, avoid co-locating hot threads on SMT siblings unless that is the experiment, record CPU model/governor/SMT state, and keep background load low.
5. **Measure a baseline.** Save raw output, not just a copied number.
6. **Change one variable.** One code/configuration change per experiment. If two things changed, the ablation is invalid.
7. **Repeat and alternate.** Run baseline/variant/baseline (A/B/A) or several interleaved repetitions to expose drift, thermals, and frequency changes.
8. **Report effect size and noise.** Median plus range or IQR is enough for this project. Do not claim a 2% “win” inside 10% run-to-run noise.
9. **Explain the mechanism.** The counter movement must match the story: branchless should move branch misses; huge pages should move dTLB misses; false-sharing fixes should reduce coherence cost even if generic cache misses barely move.
10. **Archive provenance.** Save the command, git commit, compiler/flags, kernel, CPU, event list, workload seed, thread pinning, and raw output. Resume numbers without this archive are placeholders.

Minimum experiment log:

```text
date:
git commit:
compiler + flags:
kernel + perf version:
CPU model + topology + SMT state:
governor/frequency notes:
workload + seed + duration:
pinning:
perf stat command:
primary metric:
result (median + spread):
interpretation:
boundary/when this should not generalize:
```

### 4. How to learn it from zero

Learn it in layers. Do not start with raw CPU-specific PMU events.

**Authoritative resources, in order:** `man perf-stat` and `perf list` first; the kernel's `perf` documentation for permissions and event semantics; Brendan Gregg's `perf` examples for command patterns; Intel's Top-Down Microarchitecture Analysis material only after the basic counters make sense. Vendor PMU manuals are reference material, not reading assignments.

**Layer 1 — operate the tool (~1 evening).**
- Run `perf stat /bin/true`, `perf stat ./your_existing_benchmark`, and `perf list`.
- Identify every default field in the output.
- Learn `-e`, `-r`, `-o`, `-x,`, `-p`, and `-- sleep`.
- If events show `<not supported>`, read it as a hardware/permissions/virtualization fact, not as a command failure you should hide.

**Layer 2 — build counter intuition (~2–3 small experiments).**
- Tight predictable loop vs. random branch loop: predict `branches`, `branch-misses`, cycles, and IPC before running.
- Sequential array traversal vs. random pointer chasing: predict cache behavior before running.
- Small working set vs. larger-than-cache working set: watch the crossover.
- The goal is to predict direction first, then measure. That is how counters become a model instead of trivia.

**Layer 3 — apply it to this project.**
- False-sharing toy: same logical work, different cache-line placement.
- Branchless admission check: same decisions, different control flow.
- Huge pages: same traversal, different TLB pressure.
- SMT placement: same code, different topology.
- Integrated pipeline: attach to the live process and compare load levels.

**Layer 4 — advanced awareness, not a Phase 0 requirement.**
- CPU-specific raw PMU events from `perf list` and vendor manuals.
- `--topdown` stall classification.
- Uncore events for memory-controller/NUMA traffic.
- `perf c2c` for cache-line ownership transfer.
- Multiplexing behavior when you request more events than the PMU can count simultaneously.

### 5. Event sets for this project's experiments

| Experiment | Primary question | Suggested counters | What would support the claim |
| --- | --- | --- | --- |
| False sharing / `alignas(64)` | Does separating hot lines reduce cycles for the same work? | `cycles:u`, `instructions:u`, `cache-references:u`, `cache-misses:u`, `context-switches`, `cpu-migrations` | Similar instructions, lower cycles/order; ideally corroborated by `perf c2c` because generic cache counters may not expose coherence traffic |
| Branchless checks | Does removing unpredictable branches reduce cycles? | `cycles:u`, `instructions:u`, `branches:u`, `branch-misses:u` | Lower cycles/order and lower branch-miss rate; instructions may rise. If misses were already near zero, branchless can lose |
| Huge pages | Does reducing TLB pressure improve traversal? | `cycles:u`, `instructions:u`, `dTLB-loads:u`, `dTLB-load-misses:u`, `page-faults` | Lower dTLB miss rate and lower cycles/order on a TLB-bound working set; verify `AnonHugePages` in `smaps` |
| SMT sibling vs. physical core | Do sibling threads contend for core resources? | `cycles:u`, `instructions:u`, `task-clock`, `context-switches`, `cpu-migrations` | Same instructions, worse cycles/latency on siblings; topology logged |
| NUMA local vs. remote | What is the cross-node penalty? | Cycles/latency from the harness plus CPU-specific local/remote memory events if exposed; `numastat` for placement | Reproducible same-node vs. cross-node delta on bare metal, with provenance; generic cache counters alone do not prove remote traffic |
| Integrated pipeline under load | Does admission behavior change while the serving threads are active? | Default events plus branches/cache/TLB set, attached with `-p` | Compare equal message rates and pinned topologies; separate pipeline latency from aggregate CPU events |
| Zero-allocation verification | Are hot paths allocation-free? | Interposition counters are the proof; `page-faults` and `context-switches` are supporting signals | Zero intercepted allocations after warmup; do not claim zero-alloc from `perf stat` alone |
| Final ablation | What did each optimization contribute? | One fixed event set across all configurations | Same workload for every row; primary latency/cycle metric plus explanatory counters |

Keep event sets small. PMUs have a limited number of programmable counters; when you request too many, `perf` may multiplex them and report the percentage of time each event was active. Multiplexed counters are still useful, but they are weaker evidence. For a final table, prefer several short runs with focused event groups over one enormous event list.

### 6. Interpretation rules and traps

- **IPC is not a score.** A branchless version can retire more instructions at lower IPC and still win because it avoids mispredictions. Compare total cycles and latency first.
- **A counter must match the mechanism.** Don't use branch misses to justify a cache optimization or generic cache misses to prove NUMA locality.
- **Generic cache event names are aliases.** Their exact hardware mapping varies by CPU. Check `perf list`; prefer explicit LLC events when the question is specifically about the last-level cache.
- **False sharing is coherence traffic, not necessarily an LLC miss.** `perf stat` may show the cost through cycles; `perf c2c` is the more direct specialist tool.
- **Virtual machines may hide or virtualize PMU events.** For the cross-node NUMA number, use bare metal. If a cloud instance reports `<not supported>`, that result cannot be substituted with a guess.
- **SMT siblings share resources.** A “slow core” result may be sibling contention. Record logical-to-physical topology.
- **Frequency changes contaminate cycles less than wall time, but not perfectly.** Record governor/thermal conditions and repeat runs. Never compare across machines without labeling the machine.
- **Warmup matters.** First-touch page faults, lazy allocation, cold caches, and lazy symbol resolution belong in setup, not in the measured interval.
- **Pinning matters.** `cpu-migrations` should be zero. If it is not, fix the harness before interpreting anything else.
- **One run is not a result.** Use repetitions and report spread. Keep the raw outputs.
- **Counters are aggregate.** If you need to know *where* the cycles went, switch to `perf record`/`perf report`. If you need tail latency, use the benchmark/HDR histogram.
- **Permissions are environment-specific.** If `perf` refuses an event or attach mode, check `kernel.perf_event_paranoid` and use the least-permissive configuration that allows your own-process measurements. Do not permanently weaken the workstation's security just to silence an error.

### 7. The exact `perf stat` discipline for this spec

Every `perf stat` number used in the README, design doc, resume, or interview answer must satisfy all five:

1. **Question:** the experiment has a one-sentence hypothesis.
2. **Control:** only one variable changed from baseline.
3. **Primary metric:** cycles/order or measured latency moved in the claimed direction.
4. **Mechanism:** at least one relevant counter moved in the expected direction, or the absence of movement is explained.
5. **Archive:** raw output and environment metadata are committed under `bench/results/`.

A valid claim sounds like:

> “Padding the producer/consumer counters to separate cache lines reduced median cycles/order from [A] to [B] over [N] interleaved runs, with instructions unchanged within [noise]. The mechanism is reduced coherence traffic; generic cache-miss deltas were [small/ambiguous], so the padding ablation — not the cache-miss counter alone — is the evidence.”

An invalid claim sounds like:

> “IPC improved, so the code is faster.”

### 8. Interview answers this section should unlock

- “`perf stat` is aggregate PMU counting; it tells me how much and what kind of work happened, not where the time went.”
- “I use benchmark latency as the primary claim and counters as mechanism evidence.”
- “I keep event sets small to avoid multiplexing, repeat and interleave runs, pin threads, and archive the exact command and environment.”
- “IPC is contextual. I compare cycles/order and tail latency first; a lower-IPC branchless implementation can still be faster.”
- “For false sharing, generic cache counters can be ambiguous because the cost is coherence ownership transfer; the controlled padding ablation and, where available, `perf c2c` are stronger evidence.”
- “If an event is unsupported on a VM, I don't infer it; I either choose a documented alternative or move the measurement to bare metal.”

**Learning gate:** you are done learning `perf stat` for this project when you can choose a small event set for each Phase 2 experiment, predict the direction of each counter before running, explain multiplexing and unsupported events, and defend why the resulting measurement does or does not support the optimization claim.
