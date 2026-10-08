# Phase 3: Load Generator + Integration

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`01-reading-map.md`](01-reading-map.md) (the Top-Down structured pass) · [`30-perf-playbook.md`](30-perf-playbook.md) · [`03-explain-loop.md`](03-explain-loop.md) · [`16-phase-6-scheduler.md`](16-phase-6-scheduler.md) (the phase this harness exists to measure)

**Shape of this phase:** stop measuring a component and start measuring a *system*. Everything before this was a microbenchmark; from here the numbers include a network hop, a real arrival process, and concurrency.

---

**Step 1 — 📖 Read:** **Top-Down structured pass** (see the [Reading Map](01-reading-map.md)): read 1.3–1.5 first — 1.4's delay/loss/throughput math is the back-of-envelope material interviews probe — then 2.1 (transport services frame) and 2.7 (socket concepts), then **Ch. 3 in full**: TCP vs. UDP, reliability, flow control, congestion control, head-of-line blocking. Pay special attention to **3.4 (reliable data transfer principles — sequence numbers, ACKs, timeouts)**: your request stream's sequencing and staleness design reimplements 3.4's ideas at application layer, and you should say exactly that in the design doc. **3.7 (TCP congestion control)** and flow control matter more here than they would have in the market-data framing: this is the first project where the transport pushes back on you.

**Step 2 — 🔨 Build: the load generator — an open-loop request source.**

- **What:** a generator that replays recorded request streams (prompt lengths and generation lengths drawn from a real distribution) at a **configurable offered arrival rate**, independent of how fast the server responds.
- **Why:** closed-loop generators — send a request, wait for the response, send the next — measure a system that is never actually loaded: when the server slows down, a closed-loop client politely slows down with it, and the latency number is a lie. Open-loop arrival is the only way TTFT and TPOT mean anything under load, and it is the prerequisite for coordinated-omission correction in step 6.
- **Reach for:** a target rate as the single controlled variable; Poisson or bursty arrivals as a second flavor; a recorded trace as the realistic flavor; request IDs and application-level sequence numbers from the start.
- **Learn:** that the load model is part of the measurement instrument, and an instrument you did not calibrate produces numbers you cannot defend.

**Step 3 — 📖 Read:** socket API specifics — `man 7 tcp`, `man 7 ip`, **Beej's Guide to Network Programming (targeted: stream sockets + non-blocking + `poll`/`epoll` sections)**, and Top-Down's socket-programming section (~1.5 hours, reference-style, look up as you code). Plus **one hour on load-testing practice** — how wrk2 differs from wrk and why (open-loop + coordinated-omission correction is the whole difference; Tene's talk in step 5 is the theory, this is the tool that implements it).

**Step 4 — 🔨 Build: the request path, staleness detection, then the full pipeline.**

- **What:** (a) a loopback **TCP** request stream carrying a simple sequenced binary protocol, with the gateway accepting, admitting or refusing, and responding; (b) staleness detection on the input side — a generator that silently stops is more dangerous than one that is obviously broken, so the gateway needs a heartbeat/timeout path to notice, plus the sequence-edge cases: **32-bit wraparound**, where `next != prev + 1` is wrong at the boundary and the correct test is a distance computation on the unsigned difference, and **stale-without-gap**, where the sequence advances perfectly but the workload is old; (c) the whole chain wired end to end — generator → gateway admission → **decoder behind the model-backend interface** (built in step G; a deterministic stub is acceptable here only if G has not landed) → responses → accounting ledger.
- **Why:** (a) is the transport decision made explicitly — TCP, because serving traffic wants backpressure and reliability, and because **head-of-line blocking is now a real design constraint to reason about rather than a market-data footnote**: one slow response on a shared connection delays the ones behind it, which is the entire argument for HTTP/2 multiplexing. State in the design doc why a real gateway would use HTTP/TLS and what the simple binary protocol deliberately omits. (b) is the failure mode that actually bites: a stalled client or a wedged generator looks like a healthy system with no traffic. (c) is what turns separate artifacts into one system whose latency you can defend.
- **Reach for:** monotonic request IDs as the application-level truth about what arrived; non-blocking accept with `epoll` on the server side; in-process wiring for everything except the loopback hop — that's the point, not a shortcut.
- **Learn:** that application-level reliability is a reimplementation of transport-layer ideas with different tradeoffs; that integration latency is not the sum of component latencies; that a transport's flow control and your admission policy are two different backpressure mechanisms that must agree.
- **AI hook:** keep a clean, zero-alloc boundary where request features can later feed a model-version-pinned backend without coupling feature code to the serving loop (step H in [`20-ai-extension.md`](20-ai-extension.md)).

**Step 5 — 📖 Read:** Gil Tene's *"How NOT to Measure Latency"* (talk or writeup, ~1 hour) on coordinated omission + HDR histograms.

**Step 6 — 🔨 Build: latency methodology.**

- **What:** a per-stage percentile breakdown across the whole pipeline — accept → admit → enqueue → decode/prefill → first token → completion — under live load, reported as **TTFT and TPOT separately**.
- **Why:** this is where the project stops being a codebase and becomes a measured system. "TTFT p99 is [N]ms and TPOT p99 is [M]ms while serving [R] req/s with a [B]-deep resident batch on the same pinned node" is the sentence the whole phase exists to produce.
- **Reach for:** HDR-style histograms, warmup separated from the measured interval, coordinated-omission correction (step 5 is the prerequisite, not the background reading), and `perf stat -p` for aggregate CPU-state evidence alongside the histograms — never instead of them. Record the offered rate with every percentile; a latency number without its load level is not a number.
- **Learn:** why TTFT and TPOT are different measurements answering different questions, and why one "latency" number hides the tradeoff that phase 6 is entirely about.

- **Overload run — the failure-mode measurement.** *What:* one deliberately oversubscribed run, with the generator at roughly 5–10× the rate the receiver can drain, instrumented with the same histograms. *Why:* steady-state latency is the number you want; overload behaviour is the number that proves you understand your own system. Which stage saturates first, whether the transport's flow control or your shedding policy absorbs the excess, what p99.9 and the maximum do, whether the accept queue grows without bound, whether the breaker trips and at what threshold, and whether the kill switch still responds under pressure — a serving interviewer will ask at least one of these, and "I never ran it overloaded" is a bad answer. *Reach for:* the offered rate as the only controlled variable; listen-backlog and connection counters to tell transport drops from application refusals; your own admission counters as the leading indicator. *Learn:* that a system's failure mode is part of its specification — a gateway that refuses fast is a different product from one that queues until it dies, and the design doc should say which one this is.

**✅ Verify:** Kill the generator mid-run — does the gateway detect staleness and stop waiting? Can you explain why TCP head-of-line blocking is a real constraint for multiplexed request streams, and what HTTP/2 does about it? Can you explain why the receiver spins instead of using `select()`/`poll()`, and what problem `io_uring` (post-dating OSTEP ch. 33) was built to solve? Can you state your admission policy's behaviour at 2× capacity from the overload run's data?

**Artifact:** `loadgen/` + integrated pipeline + latency report.
**Claims this phase should earn:** *"Drove the engine end-to-end from an open-loop load generator over a loopback TCP request stream: per-stage TTFT/TPOT pipeline measured under concurrent load with coordinated-omission-corrected HDR histograms; documented overload policy and shedding behaviour at [X]× offered capacity."*

---

**🎯 Interview drills (Phase 3 ships → you answer these):**

- Open-loop vs. closed-loop load generation: what does a closed-loop generator hide, and why does that flatter your p99?
- Why TCP for request/response and UDP for market data (the reverse of the archived risk-engine design): what does each transport give the application that the other cannot?
- Head-of-line blocking: why it is fatal for multiplexed request streams, and what HTTP/2 multiplexing actually fixes.
- Capacity math: nodal delay components; propagation ≈ 5µs/km of fiber → the entire colocation conversation, and why loopback measurements have no propagation term.
- Flow control vs. admission control: two backpressure mechanisms, and what happens when they disagree.
- Socket tuning vocabulary: `SO_RCVBUF`, listen backlog, `epoll` edge vs. level trigger, `TCP_NODELAY` and why Nagle interacts with small responses.

**Next:** [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md).
