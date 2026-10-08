# Resumes — house style, discipline, and the two variants

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Rule 4 applies here:** the resume describes work that exists. Until every bracket is filled, Appendix A and B are *targets*, not documents.

## House style

Match the Orderbook/Backtester entries: verb-led, mechanism stated, measured number attached. One idea per bullet — the constraint that binds is total page lines, not bullets per project; correctness second (the differentiator — almost no candidate claims fuzzed invariants + deterministic replay); ablation numbers are probe-bait for "how did you measure that?" questions; "loopback" keeps the feed claim honest.

**The inference-engine bullets live in Appendix B below** — that is their single home, so the wording can't drift between a draft block and the shipped page. Appendix A re-spines the same facts for quant roles: Orderbook, Backtester, and the engine framed as low-latency systems evidence. The per-phase resume lines in the phase docs are the *raw material* those bullets are cut from.

**Space-constrained fallback:** cut whole lines, never merge two claims into one bullet. Stacking ideas into a line is what makes a project block unreadable, and it saves less space than dropping a weak line does.

**Bracket rule:** any bullet with an unfilled placeholder ships as the fallback — an `[X]` on a resume is worse than a shorter line.

**Keyword discipline — what NOT to add:** industry terms you haven't built (FIX/ITCH/OUCH parsing, kernel bypass, DPDK, RDMA, FPGA) are probe-bait, not signal — every one converts a defensible bullet into a question you can't survive. They live in the design doc's industry-context Q&A and in interview answers attached to your measurements, never on the page. **One honest exception:** the feed bullet may say "ITCH-style sequenced binary protocol" — defensible, and "no, I modeled rather than parsed ITCH, and here's the tradeoff" is itself a good interview answer.

---

## Claim ↔ code audit (2026-09-29)

Run against the repos on this machine by grepping the code each claim names. **Shipped** = the code exists and the sentence is defensible as written. **Target** = does not ship until the work lands. **Unverifiable** = no code on this machine to check against.

### Backtester

| Claim | Status | Evidence |
| --- | --- | --- |
| Serial and parallel samplers agree bit-for-bit | **shipped** | `tests/montecarlo_test.cc:385`, `serialAndParallelAgree` |
| Index-partitioned sampling, per-run seeding | **shipped** | `src/montecarlo.cpp:148-155`, `:177` (`std::mt19937 gen(seed + i)`) |
| Trade-level Monte Carlo, Markov regime switching | **shipped** | `include/montecarlo.h:14-31` (`Regime`, `RegimeData`, transition matrix) |
| Sharpe, max drawdown | **shipped** | `src/analytics.cpp:48`, `:137` |
| CAGR | **shipped** | `src/analytics.cpp:335`, `include/analytics.h:134` |
| `lock-free SPSC queues` | **removed** | zero `spsc`/`lock-free`/`ring`/`queue` hits, and no `mutex`/`atomic` anywhere in `include/` or `src/`. Corrected to index partitioning |
| `null ensembles separating signal from luck` | **target** | zero `null`/`permutation` hits |
| `point-in-time datasets → PyTorch → ONNX → walk-forward` | **target** | zero `torch`/`onnx`/`point-in-time`/`walk-forward`/`leakage` hits |
| GPU Monte Carlo (CUDA, T4), CPU↔GPU bit-identical | **target — no fallback** | zero `cuda`/`hip`/`__global__`/`nvcc`/`rocm` hits across `include src tests CMakeLists.txt CMakePresets.json` |

### Orderbook

| Claim | Status | Evidence |
| --- | --- | --- |
| p50 add 142ns, cancel 125ns, 3-level sweep 357ns | **shipped** | `README.md` results table and `docs/benchmark/build 1/benchmark_results.md`, Google Benchmark `repeats:100` |
| Array-indexed price ladder, O(1) best-quote, zero hot-path allocations | **target** | the book is `std::map<int, OrdersAtPrice>` — `include/orderbook.h:24-25` |
| `3.9×`, `1.18M → 4.55M msgs/sec` | **target** | no such numbers anywhere in the repo |
| Generational slot map for the order-ID index | **target** | the order index is `std::map<int, Order>` — `include/orders.h:33` |
| PMR-pooled intrusive FIFO per level | **target** | zero `pmr`/`intrusive`/pool hits |
| OFI / microprice / queue-position feature layer | **target** | zero `microprice`/`imbalance`/`queuePosition` hits |

### Inference engine (v4 pivot, adopted 2026-10-09)

| Claim | Status | Evidence |
| --- | --- | --- |
| All engine bullets — gateway, scheduler, decoder, kernels | **target** | the repo holds `TODO.md` and `docs/` only — there is no `src/`. The pre-trade risk-engine claims audited here previously are superseded wholesale; the pre-pivot spec is in git history before the v4 commit |

### GoodSoft work bullet 1

**Unverifiable from this machine.** No C++ Monte Carlo source exists under `/home/jason/repos`, and `github.com` is blocked from this session, so the pybind11/FastAPI engine cannot be checked. Two things to reconcile before it ships: the linked `automation-ecosystem` README documents **five** Docker services while the bullet claims **six** (the sixth is presumably this engine — make the README show it, or drop the count), and no C++ service appears in that README at all.

### Consequences

The Orderbook block currently ships one defensible bullet of four, and the GPU bullet ships none. Appendix A is a TARGET document and rule 1 covers `[N]` placeholders — but it does not cover an *unbracketed* assertion. `Cacheline-aligned, NUMA-pinned hot path` has no bracket, no number, and no code, and it reads as shipped. Extend rule 1: **an unbracketed assertion needs code behind it, not just a bracketed number.**

Separately: `Orderbook/README.md` titles the project `C++23` while `CMakeLists.txt` sets `CMAKE_CXX_STANDARD 17`. The resume says C++17, which matches the build — fix the README, not the resume.

The pipeline lead-in under **Projects** is itself one of these unbracketed assertions: it names four steps and all four are targets. It ships when the first link ships — step H, the Orderbook feature layer — not before.

---

## Appendix A — Resume, Quant Variant (TARGET — brackets must be filled)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen | Fluent in English and Mandarin

**Education**
B.Sc. (Honours), Economics, Minor in Computer Science & Mathematics — University of Toronto, June 2025

**Work History**
**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++17 Monte Carlo engine (bootstrap resampling, EWMA decay, Sharpe-ratio optimization) deriving content-type allocation weights from 60k+ observed outcomes; shipped to production via pybind11/FastAPI.
- Sole engineer on 6 services in production (AWS Lightsail, Docker, PostgreSQL, per-service DB isolation, GitHub Actions CI/CD): owned schema, deploys, and incident response for a year; hardened the public APIs with Redis rate limiting and a proxy layer.
- Built semaphore-bounded concurrent ingestion of 2,000+ products and a weekly race calendar — DeepL-translated, zero manual steps.

**Projects**

*Three systems, one standard: a matching engine, a statistical simulation engine, and a low-latency serving engine — nanosecond-scale latency measured with controls, determinism verified bit-for-bit.*

**Order Book Matching Engine — C++17** (github.com/jason-deng-dev/Orderbook)

- Array-indexed price ladder with PMR-pooled intrusive FIFO queues per level (price-time priority): O(1) best-quote access and insert/cancel-by-ID; zero hot-path allocations.
- Replaced std::map prototype with the flat ladder (3.9× throughput, 1.18M → 4.55M msgs/sec) and the order index with a generational slot map (cancel p50 [125→N]ns) — fills bit-identical across rewrites on the 1M-message replay.
- Benchmarked 12 op paths (Google Benchmark): p50 add 142ns, cancel 125ns, 3-level sweep 357ns; p99 within ~3% of p50.
- Zero-alloc feature layer (order-flow imbalance, microprice, queue-position estimate) at [X] msgs/sec, feeding the backtester's signal inputs.

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- C++17 backtester with orthogonal Signal/Sizing/Risk components and zero-alloc CSV ingest; FIFO lot accounting with per-position statistics and equity-curve analytics (Sharpe, max drawdown, CAGR).
- Validated edge with trade-level Monte Carlo (CIs on EV, drawdown, terminal equity), Markov regime switching, and null ensembles separating signal from luck.
- GPU Monte Carlo at [N]× the 16-thread CPU path (CUDA, cloud T4), seeded replay bit-identical across CPU and GPU.
- Parallelized across [C] cores by index-partitioned writes with per-run seeding: serial and parallel samplers agree bit-for-bit under test, [S]× speedup on [C] cores.
- Model pipeline: leakage-free point-in-time datasets → PyTorch training → ONNX export → walk-forward evaluation with null-ensemble validation.

**Low-Latency Inference Engine — C++17** (repo) — *framed for this page as systems evidence; the serving-layer bullets live in Appendix B*

- Sub-microsecond O(1) admission control on the request path (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch): [A]× under a mutex + unordered_map baseline measured on the same harness.
- Cacheline-aligned, NUMA-pinned hot path: zero hot-path allocations under malloc interposition, cross-node atomics [D]× a local atomic.
- Ablation on that layout: false-sharing elimination cut p99 [B]×, branchless checks cut branch misses [C]%.
- Verification: [10M] fuzzed request sequences differential-tested against a reference implementation with zero divergences; TSan-clean lock-free SPSC ledger with decisions bit-identical across runs and thread counts.
- Driven end-to-end from an open-loop load generator over a loopback TCP request stream; per-stage latency measured coordinated-omission-correct.

**Editing rules:** cut adjectives, tool-drops, redundant tails — never numbers, never "bit-identical", never method names. Cut order under pressure: engine bullet 5 (load generator) → engine bullet 3 (ablation) → backtester parallel bullet. One page, always.

**Bracket fallbacks (current, per rule 1 — an unfilled `[X]` ships as the shorter line):** engine bullet 1 drops the `[A]× … same harness` parenthetical, keeping "sub-microsecond O(1) admission control". Engine bullet 2 drops the cross-node clause if `[D]` is unfilled, keeping the layout and interposition clauses. Engine bullet 3 drops the two ablation factors and the line goes with them. Engine bullet 4 drops the fuzz/differential clause if `[10M]` is unfilled, keeping the TSan/bit-identical clauses. Engine bullet 5 drops entirely until the load generator exists. Orderbook bullet 3 drops the `[125 → N]ns` parenthetical. Orderbook bullet 4 drops entirely if `[X]` is unfilled. Backtester bullet 3 (GPU) does not ship at all until step A exists in the repo: as of 2026-09-29 `grep -rniE "cuda|hip/|__global__|nvcc|rocm"` over `include src tests CMakeLists.txt` returns zero hits, so there is no fallback for that bullet, only deletion. Backtester bullet 4 drops entirely if `[S]×` and `[C]` are unfilled.

---

## Appendix B — Resume, AI-Infra Variant (same facts, re-spined)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen

C++ systems engineer — low-latency inference serving, GPU computing, and performance engineering.
Skills: C++17/20 · CUDA · Python/PyTorch · ONNX Runtime · Linux (perf, NUMA, GDB, TSan) · TCP/UDP · CMake

**Education** — identical to Appendix A.

**Work History**

**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++ Monte Carlo engine (bootstrap resampling, EWMA, Sharpe optimization) deriving allocation weights from 60k+ observations; shipped to production via pybind11/FastAPI.
- Sole engineer on a 6-service production platform (AWS, Docker, PostgreSQL, CI/CD): automated content and catalog operations end-to-end — eliminated routine manual ops and gave non-technical operators self-serve control without SSH; hardened public APIs with Redis rate limiting and a proxy layer.
- Built and operated a Claude-API content pipeline in production — prompt strategy and content weights tuned from historical performance data across 4 content types.

**Projects** (reordered: the engine leads — it is the flagship)

**Inference Engine — C++17/CUDA** (repo)

- End-to-end low-latency serving stack built as one system: from-scratch transformer decoder, continuous-batching scheduler, admission-control gateway, and a measured CUDA kernel study — single CMake build, every claim regenerable from `bench/`.
- Decoder: from-scratch GPT-2-class transformer (multi-head attention, KV cache behind a swappable allocator, greedy decode); output verified token-for-token against llama.cpp on fixed prompts; [X] tok/s single-stream, [N] ms TTFT at 2k context; decode at [B]% of peak DRAM bandwidth, roofline-analyzed against the kernel ladder's measured ceiling.
- Scheduler: iteration-level continuous batching with KV-budget admission (reject-at-capacity); outputs token-identical to the single-stream reference across batch compositions and thread counts; [N]× throughput at [B]× TPOT, explained by achieved decode bandwidth; TTFT/TPOT under concurrent load, coordinated-omission-correct.
- Gateway: sub-microsecond O(1) admission control (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch) over SPSC dispatch; cacheline-aligned, NUMA-pinned, zero hot-path allocations (LD_PRELOAD-verified); [A]× over a mutex baseline; perf c2c-verified false-sharing elimination cut p99 [B]×.
- Kernels: GEMM ladder naive→coalesced→shared-memory-tiled→register-blocked at [A]% of peak FP32 FLOPs and [B]% of peak DRAM bandwidth; occupancy sweep; one INT8 kernel with measured throughput gain and accuracy cost.
- Correctness: [10M]+ fuzzed request sequences differential-tested against a naive reference scheduler with zero divergences; deterministic replay bit-identical across thread counts (FP non-associativity analyzed, not assumed); TSan-clean.

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- GPU-accelerated Monte Carlo (CUDA): [N]× path throughput vs. CPU, bit-identical seeded replay across CPU/GPU, Nsight-profiled on cloud T4.
- End-to-end ML pipeline: leakage-free point-in-time datasets → PyTorch training → ONNX export → walk-forward evaluation with null-ensemble validation.
- Parallelized across [C] cores by index-partitioned writes with per-run seeding: serial and parallel samplers agree bit-for-bit under test, [S]× speedup on [C] cores.
- C++17 simulation engine: zero-alloc CSV ingest, pluggable strategy components, lot-level accounting with per-position statistics.

**Order Book Matching Engine — C++17** (github.com/jason-deng-dev/Orderbook)

- Array-indexed price ladder with PMR-pooled intrusive FIFO queues: O(1) best-quote access and insert/cancel-by-ID; zero hot-path allocations.
- Replaced std::map with the flat ladder (3.9× throughput, 1.18M → 4.55M msgs/sec) and the order index with a generational slot map (cancel p50 [125→N]ns) — fills bit-identical across rewrites on the 1M-message replay.
- Benchmarked 12 op paths: p50 add 142ns, cancel 125ns, 3-level sweep 357ns; p99 within ~3% of p50.
- Zero-alloc feature-computation layer (order-flow imbalance, microprice, queue position) at [X] msgs/sec — feeds the backtester's signal inputs.

**Editing rules:** the skills strip is the FIRST cut under space pressure (bullets carry the keywords). Never submit as "ML engineer" — boundary discipline applies to applications too. Engine-block cut order: correctness line → kernels line → gateway ablation clause. The decoder and scheduler bullets are the block's reason to exist; they go last.

**Engine bracket fallbacks:** every engine bullet is a target until the code exists. An engine bullet with an unfilled bracket ships as its shorter form or not at all — and the block's lead-in line ships only when the first two subsystems (kernels and decoder) are measurable.

**Underused asset (added 2026-09-28): the GoodSoft bullets are framed as automation work, and for AI-infra and platform roles they should be framed as distributed-systems operations work** — six services, the interfaces and failure modes between them, rate limiting and a proxy layer as defence in depth, and incident ownership as the retained consultant. It is the only entry on the page that proves you have operated something in production and been accountable for it when it broke, which is the thing most junior candidates cannot claim. It currently reads as a side note. One rewriting pass at application time: same facts, distributed-systems vocabulary, no invented specifics.

### Addendum — artifact placement (resolved by v4, 2026-10-09)

The v3 question — where the kernels bench and the decoder live once they exist — is answered by the pivot: they are subsystems of the **Inference Engine** project block, not separate blocks and not fold-into-Backtester clauses. The Backtester keeps its own block (GPU Monte Carlo + ML pipeline); the kernels ladder's numbers, the decoder's tok/s and TTFT, and the scheduler's batching tables are all engine bullets.

- **Skills strip:** "Nsight" and a kernel-optimization phrase may be added once A ships, and "KV cache / attention" once G ships. Not before. The strip stays the first cut under pressure.
- **Bracket discipline** applies to every engine bullet: the ladder's bandwidth and FLOPs percentages, the quantized kernel's accuracy delta, the decoder's tok/s and TTFT, the scheduler's throughput multiple, and every gateway ablation factor all ship as fallbacks until measured. A blocked module with no measured number is worse than omitting the bullet, because "roofline-analyzed" or "continuous batching" with nothing behind it is the easiest claim in this project to cross-examine.
