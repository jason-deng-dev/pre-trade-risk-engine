# Appendix E — Quant Dev Interview Territory (the full map)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **When to open this:** interview season, and whenever a phase's 🎯 drill block references a territory. Each per-phase 🎯 drill block in the phase docs drills the subset relevant to that phase; this appendix is the master checklist.

Weighting: ~70% coding/systems, ~20% design + math, ~10% behavioral/domain. Coverage notes reference the plan; gaps marked **[ADD]** are cheap post-ship/interview-season items.

### A. C++ language depth (~25–30%)

- **Types & objects:** trivial/POD/standard-layout, object layout & padding, alignment/`alignof`/`alignas`, sizeof quirks
- **Functions:** overload resolution, ADL, `noexcept`, `constexpr`/`consteval`, trailing return types
- **Classes:** special members (rule of 0/3/5), virtual dispatch (mechanism + cost), inheritance access, virtual inheritance, CRTP, friendship
- **Templates:** deduction, deduction guides, partial specialization/ordering, concepts & requires, fold expressions, variadics, `if constexpr`, instantiation/ODR, `extern template`, tag dispatch, template bloat
- **Value semantics:** rvalue/forwarding references & reference collapsing, `std::move` vs `std::forward`, when moves don't happen, guaranteed copy elision, copy-and-swap
- **Smart pointers & ownership:** `unique_ptr` (deleters, arrays), `shared_ptr` (control block, aliasing ctor, `make_shared` tradeoffs), `weak_ptr` (cycles, caches), `enable_shared_from_this`
- **Type erasure — the probe:** three mechanisms (virtuals, `std::function`, hand-rolled vtable) + performance comparison
- **UB minefield:** strict aliasing & common violations, signed overflow, OOB, use-after-free/double-free, uninitialized reads, **iterator invalidation per container**, dangling temporaries, evaluation order, **data races = UB**, static-init-order fiasco
- **Exceptions:** mechanism & when it costs, exception guarantees, `noexcept` as an optimization lever, `terminate` paths
- **Std internals:** vector growth/invalidation, SSO, deque chunking, `unordered_map` (buckets, load factor, iteration guarantees), `std::function` internals, `std::variant` + `visit`
- **Allocators:** allocator traits, `std::pmr` (monotonic buffer, pools), stateful allocators
- **C++20/23:** concepts, ranges, coroutines (what/why/when-not), modules, spaceship, `jthread`, latch/barrier/semaphore, atomic `wait`/`notify`, `stop_token`
- **Compile & link:** translation units, ODR, linkage, static vs. shared libs (PIC, PLT/GOT), symbol visibility, **interposition (`LD_PRELOAD`, their Phase 4)**, ABI (mangling, layout, pImpl rationale)
- **Tooling:** ASan/TSan/UBSan/LSan (what each catches), gdb (breakpoints, watchpoints, core dumps, attach), `perf`, clang-tidy vocabulary

*Coverage: 4 C++ books + Phase 4. **[ADD]:** deliberate UB whiteboard reps; "explain vtable/`std::function`/SSO" drills.*

### B. Concurrency & memory model (~20%)

- Thread lifecycle, `jthread`, arg passing, `hardware_concurrency`, oversubscription
- Mutex taxonomy (timed/recursive/shared), guards, `std::lock`, `call_once`, upgrade locking
- **Deadlock:** four conditions, prevention (ordering, hierarchical mutex, try-lock), livelock/starvation
- **Condition variables:** predicate waits, spurious/lost wakeups, `notify_one` vs `notify_all`, bounded buffer, covering condition
- Futures/promises/`packaged_task`/`shared_future`, `async` launch policies, cross-thread exceptions
- **Atomics:** all specializations, `is_lock_free`, CAS loops, ABA, the **six orderings with x86 mappings**, happens-before / synchronizes-with / release sequences, fences, atomic wait/notify
- **Memory model:** data race = UB, SC-DRF, compiler vs. hardware reordering, store buffers, TSO vs. ARM weak model
- **Lock-free:** SPSC/MPMC rings (their artifact), reclamation (hazard pointers, epochs, RCU concept), seq_cst-prototyping discipline, busy-wait tradeoffs, backoff
- **Sync cost model:** uncontended/contended mutex (futex path), atomic RMW costs, cache-line ping-pong, convoying, priority inversion, spin-then-block, two-phase locks
- Thread pools & work stealing; why refuse `std::execution`

*Coverage: CiA full + OSTEP 28 + the ring. Strongest axis — resume-probed twice; expect the deepest drilling here.*

### C. OS / systems (~15%)

- Processes/threads, creation, states, context-switch mechanics & cost, user vs. kernel mode, **syscall path & cost**
- **Virtual memory:** VA→PA walk, multi-level page tables (x86-64), **TLB (reach, associativity, huge-page interplay)**, page faults (minor/major), `mmap` family (anon/file, `mprotect`, `madvise`, `mlock`), `brk`, CoW, swap policies, thrashing, overcommit, OOM killer, fragmentation, **NUMA + first-touch + interleave**
- Scheduling: MLFQ, **CFS (vruntime, min_granularity)**, EEVDF, real-time classes, affinity (`taskset`, `cpu_set`), `isolcpus`
- IPC & signals: pipes, unix sockets, shared memory, signal delivery/masking, async-signal-safety, eventfd/timerfd/signalfd
- **IO:** blocking/non-blocking, `select`/`poll`/`epoll` (LT/ET), reactor, `io_uring` (SQ/CQ), `SO_BUSY_POLL`, zero-copy (`sendfile`, `splice`)
- Allocation: malloc internals (bins/tcache), mmap threshold, fragmentation, pools/arenas/slabs, jemalloc/tcmalloc
- Observability: `strace`/`ltrace`, `/proc` (maps, smaps, status), `perf stat/record/top`, `vmstat`, `numastat`
- Kernel-bypass concepts (DPDK, AF_XDP, ef_vi — know why, not how)

*Coverage: OSTEP spine. Second-strongest axis.*

### D. Architecture / micro-architecture (~10%)

- Pipeline: stages, hazards, forwarding, **branch prediction & misprediction cost**, speculation, OoO (ROB, reservation stations), SMT (shared resources)
- Memory hierarchy latency table; cache organization (sets/ways); **MESI/MOESI** & false-sharing mechanism; hardware memory ordering
- ISA-level atomics (`lock xadd`, `cmpxchg`, LL/SC), fences (`mfence`/`lfence`/`sfence`)
- SIMD: SSE/AVX2/AVX-512, intrinsics, auto-vectorization, gather/scatter, alignment, when SIMD loses
- Prefetching (hardware stride/pattern; software `prefetch`; when harmful)
- **NUMA quantified:** local vs. remote atomics (~5×), their cloud measurement
- Perf analysis: counters (IPC, branch-misses, cache-misses, dTLB-misses), top-down vocabulary, `rdtsc` vs `clock_gettime`
- FP: IEEE-754, rounding modes, denormal cliffs, FMA contraction, fast-math dangers, cross-vendor determinism (their CUDA/HIP claim)

*Coverage: Drepper + COD targeted + Phase 2.*

### E. Networking (~10%)

- Stack vocabulary: L2/L3/L4 roles (frames/MAC, IP/routing, TCP/UDP)
- **TCP:** segments, handshake/teardown states, seq/ack, retransmission (RTT/RTO), flow control (rwnd), congestion control (slow start, AIMD, CUBIC/BBR), **head-of-line blocking**, Nagle + delayed-ACK interaction, keepalive, SACK/timestamps
- **UDP:** header, checksum weakness, MTU/fragmentation (why avoid), **multicast (class D, IGMP, joins, loopback)**
- **The HFT split:** UDP market data / TCP order entry — defend it
- API depth: non-blocking + `EAGAIN`, partial send/recv, `MSG_*` flags, latency-relevant `SO_*` (RCVBUF/SNDBUF/REUSEADDR/REUSEPORT/BUSY_POLL/TCP_NODELAY), `shutdown` vs `close`, `SO_ERROR`
- Kernel path: interrupt → NAPI → softirq → socket buffer → wakeup; kernel bypass & `io_uring` for networking
- Protocols: FIX shape (tag=value, session seqnums, heartbeats, gap fill/resend), ITCH/OUCH/PITCH/SBE vocabulary, binary framing (length-prefix vs delimiters)
- Measurement: RTT anatomy, bandwidth-delay product, bufferbloat, **propagation ≈ 5µs/km**, serialization delay

*Coverage: Top-Down structured + Phase 3 + Beej's.*

### F. Algorithms / LeetCode (~10–15%, screen gate)

- Full LC taxonomy: arrays/two-pointer/sliding window; hashing; lists; **monotonic stack**; trees/tries; heaps; intervals; greedy; binary search (incl. on-answer); graphs (BFS/DFS/topo/Dijkstra/union-find); DP (linear/2D/knapsack/interval); backtracking; bit manipulation; math (gcd, binary exponentiation, matrices)
- Bar: medium fluent, hard reachable, **contest rating ~1900**; bug-free-first-submit discipline

*Coverage: NeetCode 150 + weekly contests (ongoing). See [`92-algorithms-and-career.md`](92-algorithms-and-career.md).*

### G. System design — low-level flavored (~5–10%)

- Repertoire: market data handler, order gateway/OMS, matching engine, risk gateway, logger (seq writes, batching, MPSC), thread pool, memory pool/arena, rate limiter (token bucket vs. sliding window), LRU cache, pub/sub, heartbeat/failover, time sync (PTP), config distribution
- Scoring axes: correctness under concurrency, latency-budget breakdown, failure handling, observability, capacity math
- Generic SD (AI-infra roles): Xu Vol. 1 flashcard territory

*Coverage: the projects ARE the designs; **[ADD]** whiteboard reps of the repertoire list.*

### H. Math / probability (~5–10% QD)

- EV, conditional probability, Bayes, combinatorics, expected-value games; distributions, CLT, CIs (owned via Monte Carlo); Fermi estimation; fast mental math; Markov chains (their domain); basic linear algebra
- *(QT shots only: full puzzle gauntlet)*

### I. Behavioral (~5–10% — the "pod trust" filter)

- Six story bank (conflict, failure, ambiguity, deadline, deep-debug, disagreement) from real projects; why-trading/firm narrative coherence; ownership signals (sole engineer, retained consultant)

### J. Market microstructure domain (~5%)

- Order types (limit/market/stop/IOC/FOK/peg), price-time priority, book mechanics, spread, adverse selection basics; latency economics (tick-to-trade, jitter, colocation); exchange anatomy (gateways, feeds, matching); light regulatory vocabulary (Reg NMS, market-maker obligations)
- Their industry-context appendix covers the vocabulary layer

### K. Resume cross-examination (runs under everything)

Every number: provenance, why-not-X alternatives, what-broke stories. Served by README ↔ perf output linkage, the ablation table, and the design doc.

*Format note: each per-phase 🎯 drill block in the phase docs drills the subset relevant to that phase; this appendix is the master checklist for interview season.*
