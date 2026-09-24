# Resumes — house style, discipline, and the two variants

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Rule 4 applies here:** the resume describes work that exists. Until every bracket is filled, Appendix A and B are *targets*, not documents.

## House style

Match the Orderbook/Backtester entries: verb-led, mechanism stated, measured number attached. Four bullets max per project; correctness second (the differentiator — almost no candidate claims fuzzed invariants + deterministic replay); ablation numbers are probe-bait for "how did you measure that?" questions; "loopback" keeps the feed claim honest.

**The risk-engine bullets live in Appendix A below** — that is their single home, so the wording can't drift between a draft block and the shipped page. Appendix B re-spines the same facts for AI-infra roles. The per-phase resume lines in the phase docs are the *raw material* those bullets are cut from.

**Space-constrained fallback (3 bullets):** merge bullet 4's first clause into bullet 1 ("…; ingests a loopback UDP multicast feed with gap detection/snapshot recovery"), drop the pipeline histogram clause.

**Bracket rule:** any bullet with an unfilled placeholder ships as the fallback — an `[X]` on a resume is worse than a shorter line.

**Keyword discipline — what NOT to add:** industry terms you haven't built (FIX/ITCH/OUCH parsing, kernel bypass, DPDK, RDMA, FPGA) are probe-bait, not signal — every one converts a defensible bullet into a question you can't survive. They live in the design doc's industry-context Q&A and in interview answers attached to your measurements, never on the page. **One honest exception:** the feed bullet may say "ITCH-style sequenced binary protocol" — defensible, and "no, I modeled rather than parsed ITCH, and here's the tradeoff" is itself a good interview answer.

---

## Appendix A — Resume, Quant Variant (TARGET — brackets must be filled)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen

**Education**
B.Sc. (Honours), Economics, Minor in Computer Science & Mathematics — University of Toronto, June 2025

**Work History**
**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)

- Built a multithreaded C++ Monte Carlo engine (bootstrap resampling, EWMA, Sharpe optimization) deriving allocation weights from 60k+ observations; shipped to production via pybind11/FastAPI.
- Sole engineer on a 6-service production platform (AWS, Docker, PostgreSQL, CI/CD): automated content and catalog operations end-to-end — eliminated routine manual ops and gave non-technical operators self-serve control without SSH; hardened public APIs with Redis rate limiting and a proxy layer.

**Projects**

**Order Book Matching Engine — C++17** (github.com/jason-deng-dev/Orderbook)

- Array-indexed price ladder with PMR-pooled intrusive FIFO queues per level (price-time priority): O(1) best-quote access and insert/cancel-by-ID; zero hot-path allocations.
- Replaced std::map prototype with the flat ladder (3.9× throughput, 1.18M → 4.55M msgs/sec) and the order index with a generational slot map (cancel p50 [125→N]ns) — fills bit-identical across rewrites on the 1M-message replay.
- Benchmarked 12 op paths (Google Benchmark): p50 add 142ns, cancel 125ns, 3-level sweep 357ns; p99 within ~3% of p50.
- Zero-alloc feature layer (order-flow imbalance, microprice, queue-position estimate) at [X] msgs/sec, feeding downstream model inference.

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- C++17 backtester with orthogonal Signal/Sizing/Risk components and zero-alloc CSV ingest; FIFO lot accounting with per-position statistics and equity-curve analytics (Sharpe, max drawdown, CAGR).
- Validated edge with trade-level Monte Carlo (CIs on EV, drawdown, terminal equity), Markov regime switching, and null ensembles separating signal from luck — extended to GPU (CUDA): [N]× path throughput vs. CPU with bit-identical seeded replay.
- Parallelized with lock-free SPSC queues: per-path seeding, bit-identical hashes across 1–[C] threads, [S]× speedup on [C] cores.
- Model pipeline: leakage-free point-in-time datasets → PyTorch training → ONNX export → walk-forward evaluation with null-ensemble validation.

**Real-Time Pre-Trade Risk Engine — C++17** (repo)

- Pre-trade risk layer between strategy and matching engine: sub-200ns O(1) checks (position/notional, rate limits, kill switch) on a NUMA-pinned, cacheline-aligned hot path; zero hot-path allocations verified via new/delete interposition.
- Fuzzed [10M] order sequences, zero invariant violations; TSan-clean lock-free SPSC ledger, bit-identical decision replay across runs and thread counts.
- Ablation study: [X]× p99 degradation on cross-node atomics, [Y]× from false-sharing elimination, [Z]% fewer branch misses via branchless checks — all perf-verified.
- Driven end-to-end from a loopback UDP multicast feed (gap detection, snapshot recovery) with ONNX inference on the hot path: replay-pinned model versions keep decisions bit-identical; batch-size vs. p99 quantified.

**Editing rules:** cut adjectives, tool-drops, redundant tails — never numbers, never "bit-identical", never method names. Cut order under pressure: work-history secondary bullets → risk-engine fallback (3 bullets). One page, always.

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

**Projects** (reordered: most AI-dense first)

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- GPU-accelerated Monte Carlo (CUDA): [N]× path throughput vs. CPU, bit-identical seeded replay across CPU/GPU, Nsight-profiled on cloud T4.
- End-to-end ML pipeline: leakage-free point-in-time datasets → PyTorch training → ONNX export → walk-forward evaluation with null-ensemble validation.
- Parallelized the simulation with lock-free SPSC queues: per-path seeding, bit-identical hashes across 1–[C] threads, [S]× speedup on [C] cores.
- C++17 simulation engine: zero-alloc CSV ingest, pluggable strategy components, lot-level accounting with per-position statistics.

**Real-Time Inference Gateway — C++17** (repo)

- Low-latency inference serving on a hot path: ONNX Runtime in C++ under live UDP feed load; model versions pinned in replay — decisions bit-identical across runs and thread counts; batch-size vs. p99 sweep quantified.
- Fuzzed [10M] sequences with zero invariant violations; TSan-clean lock-free SPSC ledger; zero hot-path allocations verified via new/delete interposition.
- Micro-architecture ablation study (perf-verified): [X]× p99 degradation on cross-node atomics, [Y]× from false-sharing elimination, [Z]% fewer branch misses via branchless checks.
- O(1) pre-trade guardrails (position/notional limits, kill switch) on a NUMA-pinned, cacheline-aligned path.

**Order Book Matching Engine — C++17** (github.com/jason-deng-dev/Orderbook)

- Array-indexed price ladder with PMR-pooled intrusive FIFO queues: O(1) best-quote access and insert/cancel-by-ID; zero hot-path allocations.
- Replaced std::map with the flat ladder (3.9× throughput, 1.18M → 4.55M msgs/sec) and the order index with a generational slot map (cancel p50 [125→N]ns) — fills bit-identical across rewrites on the 1M-message replay.
- Benchmarked 12 op paths: p50 add 142ns, cancel 125ns, 3-level sweep 357ns; p99 within ~3% of p50.
- Zero-alloc feature-computation layer (order-flow imbalance, microprice, queue position) at [X] msgs/sec — feeds the inference pipeline.

**Editing rules:** the skills strip is the FIRST cut under space pressure (bullets carry the keywords). Never submit as "ML engineer" — boundary discipline applies to applications too.
