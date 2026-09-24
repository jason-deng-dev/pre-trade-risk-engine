# Deepening Queue (for your other two projects)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Each row's **🔓 Unlocked** note appears inline in the phase doc where the prerequisite is built.

Your Orderbook and Backtester are already strong; these are the remaining high-signal additions, gated on the relevant core prerequisite being complete and the current core gate staying green. The inline **🔓 Unlocked** notes in the phase docs mark where each piece of knowledge lands — several items below are nearly free because of that. Marginal signal per hour is still smaller than the risk engine itself; if the core slips, this queue pauses (mid-project pivots are your demonstrated stall pattern).

| When | Project | Addition | Effort | Why |
| --- | --- | --- | --- | --- |
| After Phase 1 | Backtester | Port your SPSC ring in as the MC job/result transport; reproduce bit-identical hashes | ~3 days | Makes the existing resume claim true (currently a liability — see Phase 0 unlock note) |
| After Phase 2 | Backtester | `alignas(64)` on MC queue head/tail counters | ~1 day | Extends the false-sharing story to a second project |
| After Phase 1 | Orderbook | Generational slot map for the order-ID index (dense arena, IDs = (index, generation)); **fills must be bit-identical vs. the old index on the 1M-message replay** | ~2–3 days | Dense-vs-hash + cache reasoning from Phases 0–1 turned into a measured rewrite; the natural sequel to the map→ladder bullet |
| After Phase 4 | Orderbook | Per-level FIFO as PMR-pooled **intrusive list** (placement-new'd nodes from a pre-committed monotonic buffer); same bit-identical replay check; re-benchmark | ~3–5 days | Zero-alloc hot path on two projects; reuses Foundation #5 |
| After shipping | Backtester | TWAP child-order slicing + depth-aware fills (fill at best bid/ask, consume visible depth, slippage as first-class output) | ~2–3 weeks | The biggest conceptual upgrade available — makes the backtester honest about execution; keep it to TWAP + depth, don't balloon into VWAP/POV |
| After Phase 3 if green; otherwise after shipping | Orderbook | Microstructure features: order-flow imbalance, microprice, queue-position estimate — feedable as backtester signal inputs; this is AI extension step H | ~1–2 weeks | Closes the loop: orderbook → features → backtester; first AI extension module (see [`20-ai-extension.md`](20-ai-extension.md)) |

Resume discipline: each deepening adds at most one bullet with a before/after measured number — never a claim without the number. **Cap the internal-container rewrites at two or three total across all projects** — the signal saturates fast, and a fourth starts reading as compulsive rather than principled.
