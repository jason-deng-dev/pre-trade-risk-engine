# Phase 3: Mock Multicast Feed + Integration

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`01-reading-map.md`](01-reading-map.md) (the Top-Down structured pass) · [`30-perf-playbook.md`](30-perf-playbook.md) · [`03-explain-loop.md`](03-explain-loop.md)

**Shape of this phase:** stop measuring a component and start measuring a *system*. Everything before this was a microbenchmark; from here the numbers include a network hop and a real message rate.

---

**Step 1 — 📖 Read:** **Top-Down structured pass** (see the [Reading Map](01-reading-map.md)): read 1.3–1.5 first — 1.4's delay/loss/throughput math is the back-of-envelope material interviews probe — then 2.1 (transport services frame) and 2.7 (socket concepts), then **Ch. 3 in full**: UDP vs. TCP, reliability, flow control, congestion control, head-of-line blocking. Pay special attention to **3.4 (reliable data transfer principles — sequence numbers, ACKs, timeouts)**: your feed handler's gap-detection-and-snapshot design is 3.4 done at application layer, and you should say exactly that in the design doc.

**Step 2 — 🔨 Build: the feed — a publisher and a subscriber.**

- **What:** a simulator that publishes a sequenced binary market-data stream onto a loopback multicast group at a configurable rate, and a handler that consumes it without blocking.
- **Why:** the risk engine has to be measured under realistic load. A mock feed gives you a controllable, reproducible one — including the failure modes a real feed has.
- **Reach for:** UDP multicast and non-blocking sockets; socket buffer sizing as a tunable you actually test; a message format simple enough that parsing is never the bottleneck you're studying.
- **Learn:** why the receiver's design follows from the transport's guarantees — and why fan-out at this layer rules out the protocol you'd reach for first.

**Step 3 — 📖 Read:** socket API specifics — `man 7 ip`, `man 7 udp`, **Beej's Guide to Network Programming (targeted: datagram sockets + multicast sections)**, and Top-Down's socket-programming section (~1.5 hours, reference-style, look up as you code). Note: Top-Down barely covers IP multicast — Beej's and the man pages are the authoritative sources here by design.

**Step 4 — 🔨 Build: gap detection + snapshot recovery, then the full pipeline.**

- **What:** (a) stale-detection on a sequence gap, a snapshot request, a rebuild, and a resume; (b) the whole chain wired end to end — feed handler → your Orderbook → synthetic strategy → risk engine → order entry → fills → ledger.
- **Why:** (a) is the single most realistic thing in this project: sequenced feeds lose messages, and recovery is the part that has to be right. (b) is what turns four separate artifacts into one system whose latency you can actually defend.
- **Reach for:** monotonic sequence numbers as your only source of truth about what you missed; in-process wiring for everything except the UDP hop — that's the point, not a shortcut.
- **Learn:** that application-level reliability is a reimplementation of transport-layer ideas with different tradeoffs; that integration latency is not the sum of component latencies.
- **AI hook:** keep a clean, zero-alloc boundary where book state can later feed a feature-computation layer without coupling feature code to the matching loop (step H in [`20-ai-extension.md`](20-ai-extension.md)).

**Step 5 — 📖 Read:** Gil Tene's *"How NOT to Measure Latency"* (talk or writeup, ~1 hour) on coordinated omission + HDR histograms.

**Step 6 — 🔨 Build: latency methodology.**

- **What:** a per-stage percentile breakdown across the whole pipeline, under live feed load.
- **Why:** this is where the project stops being a codebase and becomes a measured system. "p99.9 risk check is [N]ns while ingesting [M] msgs/sec on the same pinned node" is the sentence the whole phase exists to produce.
- **Reach for:** HDR-style histograms, warmup separated from the measured interval, coordinated-omission correction (step 5 is the prerequisite, not the background reading), and `perf stat -p` for aggregate CPU-state evidence alongside the histograms — never instead of them.
- **Learn:** why a latency number without a load level and a measurement method is not a number; why the tail and the mean are answering different questions.

**✅ Verify:** Kill the feed mid-run, restart with a seq jump — does the book recover via snapshot? Can you explain why TCP head-of-line blocking is fatal for market data fan-out but acceptable (even desirable) for order entry? Can you explain why the receiver spins instead of using `select()`/`poll()` — and what problem `io_uring` (post-dating OSTEP ch. 33) was built to solve?

**Artifact:** `feed/` + integrated pipeline + latency report.
**Claims this phase should earn:** *"Drove the engine end-to-end from a loopback UDP multicast feed with sequence-gap detection and snapshot recovery; per-stage latency pipeline (feed→book→risk→fill→ledger) measured under live feed load with coordinated-omission-corrected HDR histograms."*

---

**🎯 Interview drills (Phase 3 ships → you answer these):**

- Why UDP for market data / TCP for order entry; why TCP head-of-line blocking is fatal for fan-out but acceptable for order entry.
- Gap-and-snapshot = reliable-data-transfer principles at application layer: sequence numbers reimplemented, retransmission replaced by snapshot.
- Capacity math: nodal delay components; propagation ≈ 5µs/km of fiber → the entire colocation conversation.
- Multicast vs unicast fan-out; SO_RCVBUF / socket-tuning vocabulary.

**Next:** [`14-phase-4-zero-alloc.md`](14-phase-4-zero-alloc.md).
