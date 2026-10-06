# Phase 1: Core + Correctness

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`10-phase-0-reading.md`](10-phase-0-reading.md) (the ring you're carrying in) · [`03-explain-loop.md`](03-explain-loop.md) · [`40-deepening-queue.md`](40-deepening-queue.md)

**Shape of this phase:** build the decision path, then attack it. Everything here is judged by the correctness gate at the end, not by whether it runs.

---

**Step 1 — 📖 Read:** **Re-read CiA 5.3** — the heart of Ch. 5 lands a second time now, with the kill switch and ledger on your desk (first full pass was Phase 0 step 17; re-reading beats first-reading for retention). Plus re-skim **CiA 4.3** as you write the rate limiter (`steady_clock`, not `system_clock`).

**Step 1b — 📖 Read (domain reading, not systems reading — added 2026-09-28):** the control set and its regulation. The systems half of this phase is well covered; the *risk* half is not covered anywhere in the plan, and it is the half that makes each check a defended decision instead of a guess. Read, in order: (1) **SEC Rule 15c3-5**, the "market access rule" — the US regulation that mandates pre-trade risk controls for market access, and the origin of most of the industry's control vocabulary; then skim MiFID II **RTS 6** for the EU equivalent. (2) **An exchange's risk-control documentation** (CME and Nasdaq both publish readable versions) for the actual control list a real engine runs: price bands/collars, maximum order size, order-count and message-rate limits, duplicate-order detection, self-trade prevention modes, and the per-instrument versus aggregate distinction. His current check list is a reasonable guess; this is what converts it into a specification. (3) **Self-trade prevention semantics** — cancel resting, cancel aggressing, cancel both — and pick one deliberately, with the reason recorded.
**Why it is worth a few hours:** every design question in this phase has a domain answer that a market-maker interviewer already knows. Why notional exposure and not quantity, why gross and not net, why the kill switch rejects new orders *and* cancels working ones, what happens to in-flight orders during a kill. **Boundary to state in the design doc:** this is a technical implementation of the control set those rules describe, not compliance work.

**Step 2 — 🔨 Build: the risk core — instrument state and the checks over it.**

- **What:** the per-instrument state a pre-trade decision reads, and the O(1) checks that turn it into accept/reject — position and notional limits (per-instrument and aggregate), order size, self-trade prevention, rate limiting.
- **Why:** this is the code that runs on every single order. Its data layout sets the latency floor before a single micro-optimization is applied.
- **Reach for:** cache-dense state — one instrument's decision inputs living in one cache line (Drepper §3.3.2 finally pays off); allocation-free check paths; a token bucket for rate limiting; the branch-elimination habit you'll test properly in Phase 2.
- **Learn:** that "O(1)" is meaningless until you say *where the data lives*; that a latency floor is a layout decision, not a codegen decision.
- *Tests first, implementation second.*

**Adversarial lens (banked from the OSTEP ch. 8 homework audit — the one transferable idea):** any allocation mechanism whose penalty rules trigger on naive behavior gets gamed by whoever understands the rules — the MLFQ gaming attack (fake I/O to avoid demotion) is the canonical example. Apply it here: when you build the token bucket, ask *can a strategy pattern its submissions to stay under the rate limit while maximizing flow?* (The MLFQ designers' answer was accounting-based: demote on allotment regardless of behavior.) Write the question and your answer into the design doc — it's a better artifact than the simulator exercise would have been.

**🔓 Unlocked — Orderbook (after the Phase 1 gate):** the dense, index-addressed state you just built is the pattern behind a generational slot map — the rewrite queued for your orderbook's order-ID index. You'll understand it well enough to replace your own cancel-by-ID container and justify it from cache behavior rather than from fashion. (The generational counter is the same stale-access idea as the slot map's ABA defense — CiA 7.3 connects.) Queued in [`40-deepening-queue.md`](40-deepening-queue.md).

**Step 3 — 📖 Read:** libFuzzer getting-started doc (~30 min), **plus the custom-mutator documentation (~30 min) — the second one is not optional here.** Your input is a structured order sequence, not a byte blob. A mutation fuzzer over raw bytes spends nearly all of its cycles exploring the deserializer and generating sequences that a real strategy could never emit, which means the 10M-sequence number measures the parser rather than the risk logic. Two fixes, use both: a **custom mutator** that emits structurally valid orders and sequence-level edits (cancel a live order, replace a fill, flip a side, reuse an ID), and **differential testing** against a deliberately naive reference implementation — same input, both engines, assert identical accept/reject decisions. Differential testing is the strongest correctness claim in this project because it needs no one to enumerate the invariants: divergence *is* the bug report.

**Step 4 — 🔨 Build: kill switch + ledger + replay.**

- **What:** three components — a kill switch that rejects new orders and cancels working ones; a position ledger fed by fills (realized P&L, mark-to-market, notional exposure); a replay log of the accept/reject stream.
- **Why:** the kill switch is the engine's own circuit breaker; the ledger is the state every check reads; replay is the only way to prove the system is *deterministic* rather than merely *passing*.
- **Reach for:** a lock-free kill signal readable on the hot path without a syscall or a lock — this is Phase 0's memory-ordering work, cashed in; your SPSC ring as the transport between strategy and risk threads; `steady_clock` for anything time-based.
- **Learn:** how to design an invariant that survives concurrency instead of a lock that hides the question; that determinism is a design constraint you build in, not a test you run at the end.
- **Decide and document the consistency model:** who owns the ledger, what the risk thread reads, and what happens if it changes between check and submit. Pick fail-safe (reject) and defend it in code.
- **Kill-switch state must survive a restart (added 2026-09-28).** A kill flag held only in process memory means a crash, a config reload, or a supervisor restart silently re-enables trading — a real incident class, not a hypothetical. Decide where the flag is persisted, what must be durable *before* the next order can be accepted, and how a restarting process learns it is still killed. Cheap now, awkward to retrofit, and a good answer to "what happens if your process dies?".
- **Limits change while the engine is running — the config-publication problem.** New position limits arrive from an operator or a config service while orders are in flight. Publish them to the hot path with no lock and no torn read. This is Phase 0's memory-ordering work applied to a more realistic problem than the kill signal, and it is the same shape as the config-distribution entry in Appendix G. One design-doc paragraph on the mechanism, and why a plain `double` store would be wrong.
- **AI hook:** version every decision input and configuration in the replay format now, so a future model version can be pinned without redesigning replay (used by step E in [`20-ai-extension.md`](20-ai-extension.md)).

**Step 5 — 📖 Read + ✅ Correctness gate (phase is not done until all pass):** Skim **CiA Ch. 11** first (~40 min — designing for testability, stress-testing techniques); then:

- Unit tests at boundaries: exactly at limit, one over, negative, zero, max int.
- **Ledger edge cases, as unit tests and as fuzz invariants (added 2026-09-28):** a duplicate fill for an already-applied fill; a fill arriving out of sequence; a fill for an unknown or already-cancelled order; cumulative fills exceeding the order's quantity; a cancel-acknowledgement arriving after the order already filled; and the reconciliation question underneath all of them — what the ledger does when the exchange's view and its own view disagree. These are the probes a market-maker interviewer uses to find out whether the ledger was designed or merely written.
- libFuzzer: structured order sequences via the custom mutator (not raw bytes); invariants — position ≤ limit after any accept, notional ≤ cap, kill switch rejects everything post-trigger, rate ≤ burst. Run [10M] sequences.
- **Differential gate:** the same generated sequences through the reference implementation built alongside this gate and through the real engine; decisions must match exactly. Any divergence is a find, not a nuisance — log it either way.
- **Reference implementation — build it once, here.** A deliberately straightforward version of the same risk path: a mutex around shared state, `std::unordered_map` for the ledger, heap allocation per decision, early-exit branches, no cache work at all. Around a day. It is not throwaway: it is the differential-testing oracle above, it is row 0 of the Phase 4 ablation table, and it is the honest answer to "why would I want yours instead of the obvious implementation." Building it in Phase 4 instead means writing it twice — see [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md).
- TSan clean under concurrent load.
- Known-answer FIFO P&L tests.

**Artifact:** `risk/` library + `risk/test/` + fuzz harness.
**Claims this phase should earn:** *"Fuzzed risk engine with [10M] random order sequences; zero invariant violations across position, notional, kill-switch, and rate-limit guarantees; bit-identical decision replay across runs and thread counts."*

---

**🎯 Interview drills (Phase 1 ships → you answer these):**

- 5-level chain: why no mutex → contended mutex → futex → syscall cost → would you spin? why not on one core → how does the OS know you're descheduled?
- acquire/release vs seq_cst — what x86 actually emits (store buffers; McKenney).
- Walk through your SPSC ring; justify EVERY memory order. Why no hazard pointers/ABA (bounded pre-allocated ring)?
- Branchless: when is it WORSE? (your forced always-taken-branch benchmark story.)
- Thread pools: what they are, why serving frameworks use them, how your SPSC-spin model differs — the contrast is the answer.

**Next:** run the transfer checkpoint in [`00-master.md`](00-master.md) out loud first, then [`12-phase-2-microarch.md`](12-phase-2-microarch.md).
