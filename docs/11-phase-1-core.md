# Phase 1: Admission-Control Core + Correctness

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`10-phase-0-reading.md`](10-phase-0-reading.md) (the ring you're carrying in) · [`03-explain-loop.md`](03-explain-loop.md) · [`40-deepening-queue.md`](40-deepening-queue.md) · [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) (what became what)

**Shape of this phase:** build the decision path, then attack it. Everything here is judged by the correctness gate at the end, not by whether it runs.

---

**Step 1 — 📖 Read:** **Re-read CiA 5.3** — the heart of Ch. 5 lands a second time now, with the circuit breaker and ledger on your desk (first full pass was Phase 0 step 17; re-reading beats first-reading for retention). Plus re-skim **CiA 4.3** as you write the rate limiter (`steady_clock`, not `system_clock`).

**Step 1b — 📖 Read (domain reading, not systems reading):** admission control as a discipline. The systems half of this phase is well covered; the *policy* half is not covered anywhere in the plan, and it is the half that makes each check a defended decision instead of a guess. Read, in order: (1) **Nygard's circuit-breaker / stability patterns** (the *Release It!* stability-patterns chapter, or Fowler's article as the short version) — the breaker, bulkheads, and load shedding, which is the vocabulary the gateway's control set is built from. (2) **Rate-limiting algorithms** — token bucket vs. leaky bucket vs. sliding window vs. GCRA; be able to say what each one bounds and what it lets through. (3) **Little's Law** and the queueing relationship between concurrency, arrival rate, and latency — the arithmetic behind every admission decision you will make, and the vocabulary for phase 6's batching curves. *(Optional, only if the control list still feels guessed: skim one production gateway's documented policy — nginx's `limit_req`/`limit_conn` or Envoy's circuit breakers and load shedding — to see the real control list. One hour, no more; this is a boundary reference, not a reading assignment.)*
**Why it is worth a few hours:** every design question in this phase has a policy answer a serving interviewer already knows. Why reject rather than queue, why concurrency caps and not just rate limits, what a breaker protects that a kill switch does not, and what happens to in-flight requests when the breaker trips. **Boundary to state in the design doc:** this is a technical implementation of the patterns those systems describe, at loopback scale, not a production gateway.

**Step 2 — 🔨 Build: the admission core — per-tenant state and the checks over it.**

- **What:** the per-tenant state an admission decision reads, and the O(1) checks that turn it into accept/reject — rate limits (token bucket), concurrency caps, quota/notional-style accounting, request-size limits, duplicate-request detection.
- **Why:** this is the code that runs on every single request. Its data layout sets the latency floor before a single micro-optimization is applied.
- **Reach for:** cache-dense state — one tenant's decision inputs living in one cache line (Drepper §3.3.2 finally pays off); allocation-free check paths; a token bucket for rate limiting; the branch-elimination habit you'll test properly in Phase 2.
- **Learn:** that "O(1)" is meaningless until you say *where the data lives*; that a latency floor is a layout decision, not a codegen decision.
- **Decide and document the rejection policy:** reject-at-capacity vs. queue-and-delay. Pick reject (fail-fast), state what it costs the refused client, and note where queueing would be the right call instead. This is the same question phase 6 asks about the KV budget, one level down.
- *Tests first, implementation second.*

**Adversarial lens (banked from the OSTEP ch. 8 homework audit — the one transferable idea):** any allocation mechanism whose penalty rules trigger on naive behavior gets gamed by whoever understands the rules — the MLFQ gaming attack (fake I/O to avoid demotion) is the canonical example. Apply it here: when you build the token bucket, ask *can a tenant pattern its requests to stay under the rate limit while maximizing consumed capacity?* (The MLFQ designers' answer was accounting-based: demote on allotment regardless of behavior.) Write the question and your answer into the design doc — it is also the fairness question phase 6 inherits.

**🔓 Unlocked — Orderbook (after the Phase 1 gate):** the dense, index-addressed state you just built is the pattern behind a generational slot map — the rewrite queued for your orderbook's order-ID index. You'll understand it well enough to replace your own cancel-by-ID container and justify it from cache behavior rather than from fashion. (The generational counter is the same stale-access idea as the slot map's ABA defense — CiA 7.3 connects.) Queued in [`40-deepening-queue.md`](40-deepening-queue.md).

**Step 3 — 📖 Read:** libFuzzer getting-started doc (~30 min), **plus the custom-mutator documentation (~30 min) — the second one is not optional here.** Your input is a structured request sequence, not a byte blob. A mutation fuzzer over raw bytes spends nearly all of its cycles exploring the deserializer and generating sequences that a real client could never emit, which means the 10M-sequence number measures the parser rather than the admission logic. Two fixes, use both: a **custom mutator** that emits structurally valid requests and sequence-level edits (cancel a live request mid-generation, complete a request, duplicate an ID, reuse a tenant, arrive exactly at a quota boundary), and **differential testing** against a deliberately naive reference implementation — same input, both engines, assert identical accept/reject decisions. Differential testing is the strongest correctness claim in this project because it needs no one to enumerate the invariants: divergence *is* the bug report.

**Step 4 — 🔨 Build: circuit breaker + ledger + replay.**

- **What:** three components — a circuit breaker that stops admitting when the system is unhealthy and a restart-safe kill switch that refuses everything on command; an accounting ledger fed by completions (tokens served, bytes, per-tenant usage, in-flight counts); a replay log of the accept/reject stream.
- **Why:** the breaker is the engine's own overload protection; the ledger is the state every check reads; replay is the only way to prove the system is *deterministic* rather than merely *passing*.
- **Reach for:** a lock-free breaker signal readable on the hot path without a syscall or a lock — this is Phase 0's memory-ordering work, cashed in; your SPSC ring as the transport between the acceptor and the serving threads; `steady_clock` for anything time-based.
- **Learn:** how to design an invariant that survives concurrency instead of a lock that hides the question; that determinism is a design constraint you build in, not a test you run at the end.
- **Decide and document the consistency model:** who owns the ledger, what the acceptor thread reads, and what happens if it changes between check and admit. Pick fail-safe (reject) and defend it in code.
- **Before designing the replay format, read one writeup on deterministic record-replay / deterministic simulation** — Wilson's *Testing Distributed Systems w/ Deterministic Simulation* (talk) or Antithesis's deterministic-simulation posts. What you are buying is the taxonomy of nondeterminism sources (thread interleaving, time, addresses, FP) — it tells you which ones your replay must pin, and which ones you are quietly assuming away. Skipping it is how a "bit-identical" claim turns out to hold only on one machine.
- **Breaker state must survive a restart.** A trip flag held only in process memory means a crash, a config reload, or a supervisor restart silently re-admits traffic — a real incident class, not a hypothetical. Decide where the flag is persisted, what must be durable *before* the next request can be accepted, and how a restarting process learns it is still tripped. Cheap now, awkward to retrofit, and a good answer to "what happens if your process dies?".
- **Limits change while the engine is running — the config-publication problem.** New quotas arrive from an operator or a config service while requests are in flight. Publish them to the hot path with no lock and no torn read. This is Phase 0's memory-ordering work applied to a more realistic problem than the breaker signal. The mechanism has a name and a literature — read the seqlock and RCU chapters of McKenney's *Is Parallel Programming Hard, And, If So, What Can You Do About It?* (free, targeted chapters) before choosing; then one design-doc paragraph on which mechanism you picked and why a plain `double` store would be wrong.
- **AI hook:** version every decision input and configuration in the replay format now, so a model version can be pinned without redesigning replay — and define the **model backend interface** here (a request in, tokens out), even though the decoder it will front does not exist yet. It is the seam phase 6 and step G both build against; see [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) §6.

**Step 5 — 📖 Read + ✅ Correctness gate (phase is not done until all pass):** Skim **CiA Ch. 11** first (~40 min — designing for testability, stress-testing techniques); then:

- Unit tests at boundaries: exactly at limit, one over, negative, zero, max int.
- **Ledger edge cases, as unit tests and as fuzz invariants:** a duplicate completion for an already-recorded request; a completion arriving out of order; a completion for an unknown or cancelled request; in-flight count going negative (the bug that means the cap leaked); a cancel-acknowledgement arriving after the request already completed; and the reconciliation question underneath all of them — what the ledger does when the client's view and its own view disagree. These are the probes an interviewer uses to find out whether the ledger was designed or merely written.
- libFuzzer: structured request sequences via the custom mutator (not raw bytes); invariants — in-flight ≤ cap after any accept, rate ≤ burst, breaker rejects everything post-trip, every admitted request is accounted exactly once. Run [10M] sequences.
- **Differential gate:** the same generated sequences through the reference implementation built alongside this gate and through the real engine; decisions must match exactly. Any divergence is a find, not a nuisance — log it either way.
- **Reference implementation — build it once, here.** A deliberately straightforward version of the same admission path: a mutex around shared state, `std::unordered_map` for the ledger, heap allocation per decision, early-exit branches, no cache work at all. Around a day. It is not throwaway: it is the differential-testing oracle above, it is row 0 of the Phase 4 ablation table, and it is the honest answer to "why would I want yours instead of the obvious implementation." Building it in Phase 4 instead means writing it twice — see [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md).
- TSan clean under concurrent load.
- Known-answer accounting tests (hand-computed usage totals over a fixed sequence).

**Artifact:** `gateway/` library + `gateway/test/` + fuzz harness.
**Claims this phase should earn:** *"Fuzzed admission path with [10M] random request sequences; zero invariant violations across rate-limit, concurrency-cap, breaker, and accounting guarantees; bit-identical decision replay across runs and thread counts."*

---

**🎯 Interview drills (Phase 1 ships → you answer these):**

- 5-level chain: why no mutex → contended mutex → futex → syscall cost → would you spin? why not on one core → how does the OS know you're descheduled?
- acquire/release vs seq_cst — what x86 actually emits (store buffers; McKenney).
- Walk through your SPSC ring; justify EVERY memory order. Why no hazard pointers/ABA (bounded pre-allocated ring)?
- Branchless: when is it WORSE? (your forced always-taken-branch benchmark story.)
- Why reject instead of queue? What does Little's Law say about the alternative?
- What does a circuit breaker protect that a rate limit does not?
- Thread pools: what they are, why serving frameworks use them, how your SPSC-spin model differs — the contrast is the answer.

**Next:** run the transfer checkpoint in [`00-master.md`](00-master.md) out loud first, then [`12-phase-2-microarch.md`](12-phase-2-microarch.md).
