# Algorithms Track + Career Strategy

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`90-resumes.md`](90-resumes.md) (the outputs this strategy ships) · [`91-interview-territory.md`](91-interview-territory.md) (Appendix F is the LC taxonomy this track feeds)

## Appendix C — Algorithms Track (parallel to the build; never competitive with it)

**Diagnosis (first contest: Q1+Q2 after NC150):** the gap is unlabeled retrieval, not volume. NC150 teaches patterns WITH labels; contests test deconstruction of unseen problems.

- **Daily (30–45 min):** 1–2 NEW untagged problems (shuffled pool). Generate candidate approaches out loud before coding. Prioritize mediums + the medium-hard boundary.
- **Weekly:** every contest (virtual if timing is bad) + same-day post-mortem: re-solve Q3(+Q4) WITHOUT editorial → read editorial → Anki the pattern as a decision rule ("constraint X → think Y").
- **Spaced re-solves:** 1-3-7 spacing on FAILED contest problems, not everything solved.
- **Metric:** contest rating, not problems done. Target ~1900 (≈ consistent 3-solves). Trajectory: 2–4 months of weekly contests.
- **Section audit rule:** does this teach a pattern I'll recognize in an unseen problem? (Example audit — Math & Geometry: do Pow(x,n), Spiral Matrix, Rotate Image, Happy Number, Set Matrix Zeroes; sprint the trivial five; skip GCD-of-Strings, Multiply Strings, Detect Squares.)
- **Budget:** ≤45–60 min/day + contest weekend. The day this eats spec hours, the allocation has gone wrong.

---

## Appendix D — Career Strategy (re-read quarterly)

**Role gates:**

- **Quant dev (PRIMARY):** interview = C++/systems depth + LeetCode + projects. This spec targets it directly.
- **QT (cheap parallel shots):** pedigree-blind at Jane-Style firms; gate = probability/mental-math/estimation gauntlet (trainable, not resume-driven). Systems knowledge ≈ zero advantage in the room; take shots when invited.
- **QR (2–3 year path):** tier-1 entry near-zero without research evidence; path = internal mobility from a dev seat or funded master's later.
- **AI infra (SECONDARY, post-extension):** mid-tier (inference startups, accelerator companies, quant AI platform teams) realistic at junior level once the [AI extension spine](20-ai-extension.md) ships; frontier-lab infra = later-via-mobility only.

**The moat:** execution moat, not knowledge moat — the compound (production C++ + measured projects + CUDA artifact + correctness story) is rare because most people attrit at every step. It decays if stalled and converts to a career moat only after the first job. Defense = velocity. AI-resistance: the durable walls are physical grounding, verification judgment, and accountability — knowledge recall is a snapshot moat with an expiration date you don't control. Practice wielding AI inside builds with the measurement discipline applied to its output.

**Contacts / funnel (referral-driven market):** Duncan Bees — Openchip (Barcelona, RISC-V accelerators): informational conversation NOW (team structure, junior screening, remote/relocation). Same profile fits Tenstorrent (Toronto HQ) and North American RISC-V/analogs. Keep contacts warm early.

**Documents:** two one-page resume variants ([`90-resumes.md`](90-resumes.md) Appendix A/B); never hybrid. Resume = output of shipped work.

**Sequence (there is no deadline):** Core Quant Spine first (AI hooks inline; extension modules only at green gates) → the core proof pack exists (see [`15-phase-5-docs.md`](15-phase-5-docs.md)) → applications start → remaining AI extension modules → AI variant live. Algorithms parallel throughout. If the core slips, the extension spine pauses; applications do not wait for every AI module. The gate is *the proof pack existing*, not a date — which is the whole reason this plan tracks hours per category instead of weeks elapsed.
