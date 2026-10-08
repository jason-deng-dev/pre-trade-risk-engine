# The Foundational Core (what this build is quietly teaching you)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.

Foundations are a small set of first principles that *generate* answers — not book coverage. This schedule builds five, deliberately:

| # | Foundation | Generative for | Where it's built |
| --- | --- | --- | --- |
| 1 | The memory hierarchy (caches, coherence, NUMA, TLB) | False sharing, `alignas`, huge pages, dense-vs-pointer-chasing, PMR pools | Phase 0 (Drepper §1–3) → Phase 2 (Drepper §4–6.4.1 + COD structured read + OSTEP virt) |
| 2 | The hardware memory model (store buffers, ordering) | Breaker/kill-switch signalling, SPSC correctness, seq_cst cost, every "why is this atomic enough" question | Phase 0 (OSTEP 28 + McKenney → CiA 5 full incl. 5.3 → SPSC ring v1/v2 — **built from zero**) → Phase 1 (5.3 re-read, immediately applied) |
| 3 | What the OS does for you and what it costs (syscalls, context switches, VM) | Kernel-bypass rationale, pinning, zero-alloc hot paths | Phase 0 (OSTEP processes + memory half, ch. 6/28) → Phase 2 (re-skim before experiments) |
| 4 | Transport-layer semantics (reliability, flow control, ordering) | TCP vs. UDP, flow control vs. admission control as backpressure, head-of-line blocking, capacity math | Phase 3 (Top-Down structured read incl. Ch. 3 full, applied to the request path and load generator) |
| 5 | The C++ object model (lifetime, UB, aliasing) | Interposition, placement new, every "is this UB?" code review | Phase 4 (cppreference) + your Effective C++/Modern C++ base |

If a phase ever feels like "just following steps," return to this table — each foundation is the *why* behind a cluster of build decisions.

**How to check a foundation actually landed:** the end-of-phase transfer checkpoint in [`00-master.md`](00-master.md) — five prompts, answered without notes. The Honesty Rule those answers must obey, and its operational form, is in [`30-perf-playbook.md`](30-perf-playbook.md).
