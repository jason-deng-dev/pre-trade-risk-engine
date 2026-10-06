# Phase 5: Docs + Post-Project Reading Queue

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`90-resumes.md`](90-resumes.md) (the bullets this phase makes true) · [`91-interview-territory.md`](91-interview-territory.md) (what the proof pack is defending against)

**Calibrate the genre first (added 2026-09-28).** Read one or two respected public design documents before writing yours — a RocksDB or LLVM design doc, or a mature RFC. One hour. The failure mode for a first design doc is writing a README with more words: the genre answers *why this design and what it cost*, never *what the code does*.

**Build:** README (every claim linked to `perf stat` output or Google Benchmark JSON); design doc (2–3 pages): dense-array choice + crossover reasoning, atomic flag vs. mutex, branchless — when and why not, huge pages — when they hurt, ledger consistency model, **why UDP for market data / TCP for order entry**, gap-and-snapshot rationale, plus three additions:

- **The latency budget arithmetic — the denominator for "sub-200ns."** A latency claim without the budget it lives inside is a benchmark, not an argument. State what tick-to-trade costs at this scale (roughly 1–10µs, and why — feed handling, decision, gateway, serialization), state what fraction of it your risk check consumes, and state what would have to be true for that fraction to matter. This single paragraph is what turns the headline number into a design decision, and it is the first thing a sharp interviewer will ask for.
- **The overload policy.** From the Phase 3 overload run: which stage saturates first, whether the kernel or the application drops, whether the ring's producer blocks or the oldest entry is sacrificed, and what the kill switch does under sustained pressure. A system's failure mode is part of its specification.
- **Final-versus-reference delta.** The headline comparison against the straightforward implementation (mutex, `std::unordered_map`, allocating), with the optimization ladder from Phase 4 underneath it as the evidence. Reviewer question answered: why yours instead of the obvious thing. The README, design doc, raw benchmark outputs, fuzz/TSan evidence, and replay hashes together form the **core proof pack** referenced by the career timeline in [`92-algorithms-and-career.md`](92-algorithms-and-career.md).

**Design doc appendix — "Industry context" Q&A** (the home for the keywords that must NOT go on the resume, each answered from your own measured system):

- **FIX vs. ITCH/OUCH**: FIX = session-based order entry over TCP (heartbeats, sequence recovery); ITCH = sequenced market data — the gap-and-snapshot model you implemented; OUCH = exchange-native low-latency order entry. One paragraph each, from your system's perspective.
- **Kernel bypass / DPDK / RDMA**: the mechanism is your Phase 2 measurement (syscall + copy overhead); DPDK = userspace poll-mode drivers, RDMA = zero-copy remote memory. Frame: "I know the measured cost these eliminate; I didn't demo them because the point is the physics, and the physics I have."
- **FPGA**: why the last microseconds go to hardware — deterministic latency, no OoO/branch surprises. Know the boundary; don't claim the skill.
- **Colocation**: propagation delay ≈ 5µs/km of fiber — do this math on a whiteboard; it's the entire colocation conversation (why strategy location is infrastructure, not code).

**📖 Post-project queue — only after shipping, sized for interview prep:**

- **DDIA Ch. 1–2, 5–6 skim** (consistency models, replication) — for distributed-flavored system design rounds.
- **Alex Xu Vol. 1 as Q&A flashcards** (caching, rate limiting, load balancing chapters). **Vol. 2: skip.**
- **OSTEP persistence skim (ch. 36, 39, 42 only — ~2 hours)** — I/O-device model, file API, journaling vocabulary. The rest of the persistence half is cut, not deferred.
