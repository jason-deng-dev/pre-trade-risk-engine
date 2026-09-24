# Reading Map — your list → this project

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.

Every book on your list gets a verdict and a landing spot. Nothing is "read because it's on the list."

| Your reading list | Verdict | Where it lands |
| --- | --- | --- |
| Concurrency in Action | **Read fully (Ch. 1–11)** — slotted into the Phase 0 execution order with per-chapter depth notes there (Ch. 5 with 2–3× time budget; 7.1/7.3 + 8.2 with full attention; Ch. 6/9–11 at comprehension depth). Phase 1 keeps a targeted 5.3 re-read | Phases 0–1 |
| OSTEP (Three Easy Pieces) | **Spine of Phase 0: ch. 1–34 in order with carve-outs** — exact read/skip map in Phase 0 item 1 (~70% read; concurrency half mostly covered by CiA except ch. 28/33). Persistence (36–51) cut to a 2-hr skim of 36/39/42; security (52–57) skipped | Phase 0 |
| Computer Networking: Top-Down (C1–5) | **Structured read (~50% of Ch. 1–6).** Read fully: 1.3–1.5 (1.4 delay/loss/throughput is the single highest-value section in the book — the interview 'capacity math'), 2.1 + 2.7, **Ch. 3 in full**, 4.1 + 4.3.1–4.3.2, 6.4.1–6.4.2 + 6.7 (day-in-the-life walkthrough). Skim: 1.1–1.2, 2.2, 2.4, 4.2, 4.3.3, 5.2, 5.4, 5.6, 6.1–6.3. Skip: 1.6–1.7, 2.3/2.5/2.6, 4.3.4/4.4/4.5, 5.3/5.5/5.7, 6.5–6.6, Ch. 7–8. Note: multicast specifics are barely covered — Beej's + `man 7 ip` fill that gap | Phase 3, just-in-time |
| Computer Organization and Design | **Structured read with a skip list (~55–60% of the book) — promote to primary source, not backup.** Read: Ch. 1 (performance/Amdahl); 2.4, 2.8, 2.12 (link/load — directly feeds Phase 4 interposition), 2.14 (arrays vs. pointers — aliasing/UB intuition); 3.5 (IEEE-754), 3.6–3.7 skim (SIMD literacy); 4.6, 4.8–4.11 (pipelining, hazards, ILP); 5.1–5.4, 5.7–5.8, 5.10 (caches, VM, MESI — Drepper's rigorous second source); 6.1–6.5 (incl. 6.4 hyperthreading — directly feeds Phase 2 core-pinning decisions). Skip: ISA mechanics (2.2–2.11, 2.15–2.19), arithmetic circuits (3.1–3.4), datapath/HDL construction (4.3–4.5, 4.7, 4.14), ECC/RAID/cache-controller implementation (5.5, 5.9, 5.11–5.12), GPUs/warehouse-scale (6.6–6.13), Appendices A–E | Phase 0 (Ch. 1 + 5.1–5.4/5.10, alongside Drepper) → Phase 2 (4.6/4.8–4.11 with Agner Fog; 6.4 before pinning decisions; 6.5) → Phase 4 (2.12) → AI extension spine (3.5, 6.6/6.7/App. C) |
| DDIA | **Not a build-time read.** Skim Ch. 1–2, 5–6 pre-interview (consistency models, replication) | Post-project queue |
| System Design (Alex Xu Vol 1/2) | **Not a build-time read.** Vol. 1 as Q&A flashcards pre-interview; **Vol. 2 skip** (it's for "design Twitter" rounds you won't face) | Post-project queue |

**Rule:** nothing on the post-project queue gets read during the build. It exists so you don't feel the itch to "finish the list" instead of shipping.

**Foundation supplements added by this plan (not on your original list):** McKenney's *Memory Barriers: a Hardware View for Software Hackers* (Phase 0), Preshing's memory-ordering posts (Phase 0, optional), Stokes' *Inside the Machine* (Phase 0, optional intuition primer), Beej's Guide to Network Programming (Phase 3), and a two-half full read of Drepper (Phases 0 + 2) that replaces COD outright.

**Which of these builds which foundation:** see [`02-foundations.md`](02-foundations.md). **What each book's chapters actually schedule:** see the phase doc they land in.
