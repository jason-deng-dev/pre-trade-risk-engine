# 41 — Orderbook: Network Ingestion Ladder (AF_XDP)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Companion docs:** [`40-deepening-queue.md`](40-deepening-queue.md) (this is the largest row in it) · [`31-verification-and-observability.md`](31-verification-and-observability.md) (sanitizers on the new code; the observation-cost rule) · [`12-phase-2-microarch.md`](12-phase-2-microarch.md) (the syscall/context-switch physics this cashes in) · [`90-resumes.md`](90-resumes.md) (the keyword rule this changes)

**Shape of this workstream:** the Orderbook's matching core is measured; its *ingestion path* is not. Everything up to now arrives through a synthetic loopback feed, and the claim on the page says so. This replaces the synthetic path with a real one, in rungs, each measured against the last — so the feed handler becomes part of the artifact rather than a test fixture, and the latency number gains a wire end.

**Why this is the Orderbook's next problem and not a new project:** a matching engine's latency budget is dominated by what happens *before* the book sees the message. The kernel network path is interrupt → NAPI poll → `skb` allocation → protocol processing → socket queue → wakeup → `recvmsg` → copy. That is two copies and a scheduler round trip per packet, all of it measurable, all of it removable. Phase 0's syscall cost model and Phase 2's kernel-bypass drill explain *why* it exists; this is where the explanation gets a number attached.

**Placement and gate:** a Quant Anchor. Gate: the Orderbook's replay harness is stable and its correctness gate is green (bit-identical fills on the 1M-message replay). Like every anchor, it **pauses** whenever the engine spine needs the hours — and of all the anchors, this is the one sized to need that rule.

---

## The ladder

Build it in rungs, exactly like the GEMM ladder and the scheduler. Each rung is measured before the next is written, and **a rung that buys nothing is a finding worth keeping** — the rung-2 or rung-3 number is often "less than expected, here is why," and that answer is worth more in an interview than a stack of wins.

| Rung | Path | What it isolates | Hardware needed | Effort |
| --- | --- | --- | --- | --- |
| **0** | the existing UDP-multicast loopback feed | baseline: everything the kernel does per message | none | done |
| **1** | `SO_BUSY_POLL` (`SO_PREFER_BUSY_POLL`, `net.core.busy_poll`) + `PACKET_MMAP`/tpacket ring | the interrupt/wakeup cost alone, and one of the two copies | none — works on any NIC, including a VM | ~2–3 days |
| **2** | AF_XDP **copy mode** (`XDP_REDIRECT` into a UMEM socket, generic or native XDP) | the socket layer and `skb` gone; DMA→kernel→UMEM copy remains | any NIC with XDP support; generic/SKB mode works everywhere | ~1 week |
| **3** | AF_XDP **zero-copy** (`XDP_ZEROCOPY` bind) | the last copy — NIC DMAs straight into your frames | a driver that implements it, plus a *packet source* (below) | ~1–2 weeks |

**Rung 1 is the highest value per hour in this doc.** `SO_BUSY_POLL` converts the wakeup path into a spin and tcpdump-visible behaviour does not change at all; the tpacket ring removes one copy with no new dependency. If the ladder stops after rung 1, the workstream has still produced a real before/after.

---

## Hardware reality (read this before writing code)

Three separate constraints, and each one can stop you:

1. **XDP mode.** Native XDP needs a driver that implements it. Consumer onboard NICs are the usual blocker — Intel `igc` (i225/i226, common on X670 boards) does native XDP but **not** zero-copy; Realtek typically does neither. Check with `xdp-tools`' `xdp-features` and `ethtool -i`, and treat a failed `XDP_ZEROCOPY` bind as information rather than an error to work around. The cheap enablers: a used Intel X520/X710 (~$50–80, `ixgbe`/`i40e`, solid XDP + zero-copy), or one cloud bare-metal afternoon.
2. **The packet source.** This is the practical blocker nobody mentions: AF_XDP needs packets arriving on a real NIC, and a loopback feed is not one. Your options, best first: (a) **two ports and a cable** — transmit the recorded feed from port A, receive on port B with AF_XDP (`tcpreplay` or your own sender); (b) **a second machine** on a direct link; (c) **`pktgen`** (in-kernel, `/proc/net/pktgen`) for throughput work where you do not need realistic message content; (d) a QEMU VM with `virtio-net`, which supports XDP and, on recent kernels, AF_XDP zero-copy — the lightest path if you do not want to buy hardware. **WSL2's `hv_netvsc` supports neither native XDP nor zero-copy**, so the Windows-side dev loop can build and unit-test the ring plumbing but cannot produce rung-2/3 numbers. Develop anywhere; measure on bare metal.
3. **Root.** AF_XDP, XDP program loading, and `ethtool -T` all need privileges. Note in the design doc that the data path is unprivileged *once bound* — that distinction ("the loader needs root, the hot path does not") is itself a good boundary answer.

**Recorded-feed rule:** reuse the existing 1M-message replay as the workload across every rung. Same messages, same order, same seeds — otherwise the rungs are not comparable and the ladder is decoration.

---

## The architectural claim: UMEM is your PMR pool

AF_XDP hands you a **UMEM**: one pre-committed, contiguous region you split into fixed-size frames and register with the socket. The kernel and (in zero-copy mode) the NIC allocate nothing — they hand frames back and forth through four rings:

- **FILL** — you give empty frames to the kernel.
- **RX** — the kernel gives filled frames back to you.
- **TX / COMP** — the transmit side and its completion ring (build them only if you transmit).

Every ring is a single-producer/single-consumer index pair with explicit barriers — the same structure as the SPSC ring from Phase 0, with one difference that is the whole lesson: **the producer is hardware**, so the C++ memory model no longer covers the relationship and `libbpf`'s `xsk_ring_prod`/`xsk_ring_cons` accessors exist precisely because of that. An order is deserialized *in place* inside the frame, and the book takes an index into the arena.

That is the claim worth making: **the same arena allocator serves the order book and the NIC.** It is the single most cohesive sentence across the two C++ projects, and it only becomes true if you resist the urge to copy frames into a "real" buffer on arrival. Resist it. If a rung requires a copy to work, write down why — that is a finding about the driver, not a licence to abandon the design.

Cache-line discipline carries over unchanged: the ring index pairs get their own lines, and `perf c2c` now shows you contention between a DMA engine and a core rather than between two cores.

---

## Wire timestamps (do this first, at rung 0)

**Without a hardware timestamp, every latency number you own is userspace-to-userspace**, and the rung-3 story is unfalsifiable. Fix it before the ladder, not after:

- `ethtool -T <iface>` — what the NIC can timestamp. `SOF_TIMESTAMPING_RX_HARDWARE` is what you want; if the NIC exposes a PHC (`/dev/ptp*`), you can discipline it with `phc2sys`/`ptp4l` and get a genuinely trustworthy clock.
- `SO_TIMESTAMPING` on the socket, or the AF_XDP equivalent via `xsk_umem__get_data` + the RX ring's timestamp field where the driver provides it.
- Record, per message: wire-arrival (hardware), userspace-visible, book-updated. The **wire-to-book** number is the one that goes on the page; the other two are the mechanism.

If the NIC cannot hardware-timestamp, say so and label the claim userspace-to-userspace. An honest boundary beats an unlabelled number, and "my NIC can't timestamp, so here is what I can and cannot claim" is a finished answer.

---

## Real message mix: a real ITCH file

The resume rule currently allows "ITCH-style" with an honest tradeoff answer ("I modelled rather than parsed ITCH"). That answer is good. **Having parsed the real thing is better**, it costs 2–4 days, and it removes the question entirely.

- NASDAQ publishes the **ITCH 5.0 specification** and public **sample data** files — real symbol universe, real message mix, real order sizes.
- Transport framing is part of the work: **MoldUDP64** for the multicast feed (sequence numbers, heartbeat, gap detection) or the TCP variant. Gap handling is the piece that makes this more than a parser — it is the same staleness/sequencing story Phase 3 keeps for the engine, in its native habitat.
- Then: the same replay harness runs a real message mix, and the latency distribution stops being synthetic. Message *mix* is what makes a book's latency distribution bimodal (the modify/cancel fast path vs. the price-level walk), and a synthetic mix hides exactly that.

**Boundary:** parsing a sample file is not a production feed handler. No arbitration, no redundancy, no A/B feed comparison. Say that; it is true and it is fine.

---

## Correctness gate (the workstream is not done until these pass)

- **Fills bit-identical across rungs.** The same recorded stream through rung 0, rung 1, and the AF_XDP path must produce bit-identical fills and an identical event log. This is the Orderbook's existing replay claim, extended to the transport — the transport is not allowed to change the book's behaviour, only its latency.
- **Gap and loss behaviour, as a test not a hope.** Drop packets deliberately (netem, or a feed that skips sequence numbers): the book must detect it, and the policy (halt vs. continue-stale) must be the documented one. A feed handler that silently accepts a gap is the classic production incident.
- **Backpressure:** FILL ring starvation and RX ring overflow are both real and both have defined behaviour. Assert it; do not discover it under load.
- **TSan clean** with the socket thread and the book thread running; **ASan/UBSan clean** per [`31-verification-and-observability.md`](31-verification-and-observability.md), with the fuzz corpus extended to malformed frames (truncated, oversized, wrong message type) — a network path is the one place untrusted bytes enter this portfolio, and the fuzzer should be pointed at the deserializer with real hostility.

---

## Benchmarks — the workstream's actual output

Every row is a table in the Orderbook README, generated by the harness, with the raw `perf` output committed under `bench/results/` (playbook §7).

| Measurement | Why it is the one that matters |
| --- | --- |
| **Cycles in the kernel network stack, per message, rung 0 → 3** | `perf stat -e cycles` without `:u` — this is the mechanism number, and it isolates exactly what the rung removed |
| **Syscalls per message** (expect 1–2 → 0) | `strace -c`, or `perf trace`; the bluntest and most convincing single line |
| **Wire-to-book latency: p50 / p99 / p99.9 / p99.99**, hardware-timestamped | the headline, and the reason the timestamp work comes first |
| **CPU cost per message at sustained rate** (10k/s and at the NIC's practical ceiling) | at low rates latency dominates; at high rates the CPU saving is the story. Measure both or the finding is half-told |
| **FILL-ring starvation and RX overflow counts under load** | proves the ring sizing is a decision, not a default |

**Boundary to state in the writeup:** rung 3 on a 1–10G link in a home lab is not a colocated exchange feed. The finding is the *shape* of the removal (what the kernel path cost, what removing it bought), with the machine annotated — the same provenance discipline as the cross-node NUMA number.

**Claims this workstream should earn:** *"Replaced a loopback feed with an AF_XDP user-space data path over a real NIC: cycles in the network stack per message [A] → [B] (perf-verified), syscalls per message [N] → 0, wire-to-book p99.9 [C]ns measured against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream."* — with the rung stated, and the ITCH parse as its own bullet once it runs the same harness.

---

## Cut order (within this workstream)

1. Rung 3 (zero-copy) — cut first; needs a specific driver and a packet source
2. Rung 2 (AF_XDP copy mode) — falls back to rung 1's numbers without it
3. Real ITCH file — the synthetic mix stays if hours compress
4. **Never cut:** wire timestamps (without them nothing here is claimable), the fills-bit-identical gate, and gap/loss testing

Rungs 0–1 plus timestamps need no special hardware and no privileges beyond `perf`. If the whole workstream stalls, that much still ships — which is why it is rung 1 and not a footnote.

---

## 📖 Readings (all free, all just-in-time)

| Resource | Covers |
| --- | --- |
| Kernel `Documentation/networking/af_xdp.rst` | the authoritative ring semantics, UMEM, copy vs. zero-copy modes. Read this *before* any blog |
| `xdp-project/xdp-tutorial` (the AF_XDP lessons) | the working skeleton; the fastest path to a bound socket |
| `libbpf`/`libxdp` docs + `xsk_ring_*` accessors | the ring API and why the barriers are where they are |
| `man 7 packet`, `SO_BUSY_POLL` kernel docs | rung 1 — the cheapest rung needs the least reading |
| `ethtool -T`/`-i` output, `xdp-features` | capability discovery as a habit |
| NASDAQ **ITCH 5.0 spec** (+ MoldUDP64 section) and the public sample file | the real message mix |
| Top-Down ch. 3 (already on your list) | the transport framing you are bypassing; re-read §3.7 |

**Drills (this workstream ships → you answer these, cold):**

- Walk the kernel receive path and say what each stage costs — then say which stages AF_XDP removes and which it does not.
- What is the difference between generic/SKB mode, native mode, and zero-copy — and what does each cost you?
- Why does the FILL ring exist at all? What happens when it runs dry, and how do you size it?
- Why do the ring accessors use explicit barriers instead of `std::atomic`? What breaks if you use the C++ memory model here?
- Where do your frames live, who allocates them, and what does that have to do with your order book's allocator?
- How do you know a packet was lost, and what does your book do about it?
- What is your wire-to-book number, and how was the wire timestamp obtained?
- Why is `SO_BUSY_POLL` a different tradeoff from AF_XDP — what does each buy and what does each cost in CPU?

---

## War-story prompts (write these down as they happen)

The two that almost certainly will: **the ring that appeared to work and dropped frames intermittently** (usually a barrier, a ring index not being committed, or a descriptor returned to the wrong ring — and it will present as a book bug first), and **the number that got worse at rung 2 before it got better** (busy-polling a core against SMT siblings, or a NIC on the wrong NUMA node — Phase 2's topology lesson, arriving as a production bug). Keep both. They are the interview answers this workstream exists to produce.

**Next:** run the transfer checkpoint in [`00-master.md`](00-master.md) out loud, then update the Orderbook README's claims table and the audit rows in [`90-resumes.md`](90-resumes.md).
