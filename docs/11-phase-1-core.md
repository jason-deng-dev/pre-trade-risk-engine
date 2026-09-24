# Phase 1: Core + Correctness

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`10-phase-0-reading.md`](10-phase-0-reading.md) (the ring you're carrying in) · [`03-explain-loop.md`](03-explain-loop.md) · [`40-deepening-queue.md`](40-deepening-queue.md)

**Shape of this phase:** build the decision path, then attack it. Everything here is judged by the correctness gate at the end, not by whether it runs.

---

**Step 1 — 📖 Read:** **Re-read CiA 5.3** — the heart of Ch. 5 lands a second time now, with the kill switch and ledger on your desk (first full pass was Phase 0 step 17; re-reading beats first-reading for retention). Plus re-skim **CiA 4.3** as you write the rate limiter (`steady_clock`, not `system_clock`).

**Step 2 — 🔨 Build: the risk core — instrument state and the checks over it.**

- **What:** the per-instrument state a pre-trade decision reads, and the O(1) checks that turn it into accept/reject — position and notional limits (per-instrument and aggregate), order size, self-trade prevention, rate limiting.
- **Why:** this is the code that runs on every single order. Its data layout sets the latency floor before a single micro-optimization is applied.
- **Reach for:** cache-dense state — one instrument's decision inputs living in one cache line (Drepper §3.3.2 finally pays off); allocation-free check paths; a token bucket for rate limiting; the branch-elimination habit you'll test properly in Phase 2.
- **Learn:** that "O(1)" is meaningless until you say *where the data lives*; that a latency floor is a layout decision, not a codegen decision.
- *Tests first, implementation second.*

**Adversarial lens (banked from the OSTEP ch. 8 homework audit — the one transferable idea):** any allocation mechanism whose penalty rules trigger on naive behavior gets gamed by whoever understands the rules — the MLFQ gaming attack (fake I/O to avoid demotion) is the canonical example. Apply it here: when you build the token bucket, ask *can a strategy pattern its submissions to stay under the rate limit while maximizing flow?* (The MLFQ designers' answer was accounting-based: demote on allotment regardless of behavior.) Write the question and your answer into the design doc — it's a better artifact than the simulator exercise would have been.

**🔓 Unlocked — Orderbook (after the Phase 1 gate):** the dense, index-addressed state you just built is the pattern behind a generational slot map — the rewrite queued for your orderbook's order-ID index. You'll understand it well enough to replace your own cancel-by-ID container and justify it from cache behavior rather than from fashion. (The generational counter is the same stale-access idea as the slot map's ABA defense — CiA 7.3 connects.) Queued in [`40-deepening-queue.md`](40-deepening-queue.md).

**Step 3 — 📖 Read:** libFuzzer getting-started doc (~30 min) — timed for when you're about to need it.

**Step 4 — 🔨 Build: kill switch + ledger + replay.**

- **What:** three components — a kill switch that rejects new orders and cancels working ones; a position ledger fed by fills (realized P&L, mark-to-market, notional exposure); a replay log of the accept/reject stream.
- **Why:** the kill switch is the engine's own circuit breaker; the ledger is the state every check reads; replay is the only way to prove the system is *deterministic* rather than merely *passing*.
- **Reach for:** a lock-free kill signal readable on the hot path without a syscall or a lock — this is Phase 0's memory-ordering work, cashed in; your SPSC ring as the transport between strategy and risk threads; `steady_clock` for anything time-based.
- **Learn:** how to design an invariant that survives concurrency instead of a lock that hides the question; that determinism is a design constraint you build in, not a test you run at the end.
- **Decide and document the consistency model:** who owns the ledger, what the risk thread reads, and what happens if it changes between check and submit. Pick fail-safe (reject) and defend it in code.
- **AI hook:** version every decision input and configuration in the replay format now, so a future model version can be pinned without redesigning replay (used by step E in [`20-ai-extension.md`](20-ai-extension.md)).

**Step 5 — 📖 Read + ✅ Correctness gate (phase is not done until all pass):** Skim **CiA Ch. 11** first (~40 min — designing for testability, stress-testing techniques); then:

- Unit tests at boundaries: exactly at limit, one over, negative, zero, max int.
- libFuzzer: random order sequences; invariants — position ≤ limit after any accept, notional ≤ cap, kill switch rejects everything post-trigger, rate ≤ burst. Run [10M] sequences.
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
