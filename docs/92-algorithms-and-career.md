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
- **AI infra (co-primary in v4):** the inference engine is now the flagship, so this track no longer waits on an extension — mid-tier (inference startups, accelerator companies, quant AI platform teams) is realistic at junior level once the engine's kernels and decoder are measurable, with the scheduler strengthening the application. Frontier-lab infra = later-via-mobility only.
- **Both tracks ride the same build (v4):** the engine is systems evidence for quant rounds and serving evidence for AI-infra rounds; the two resume variants ([`90-resumes.md`](90-resumes.md) A/B) re-spine the same facts. Neither track is waiting on the other.

**The moat:** execution moat, not knowledge moat — the compound (production C++ + measured projects + CUDA artifact + correctness story) is rare because most people attrit at every step. It decays if stalled and converts to a career moat only after the first job. Defense = velocity. AI-resistance: the durable walls are physical grounding, verification judgment, and accountability — knowledge recall is a snapshot moat with an expiration date you don't control. Practice wielding AI inside builds with the measurement discipline applied to its output.

**Contacts / funnel (referral-driven market):** Duncan Bees — Openchip (Barcelona, RISC-V accelerators): informational conversation NOW (team structure, junior screening, remote/relocation). Same profile fits Tenstorrent (Toronto HQ) and North American RISC-V/analogs. Keep contacts warm early.

**Documents:** two one-page resume variants ([`90-resumes.md`](90-resumes.md) Appendix A/B); never hybrid. Resume = output of shipped work.

**Sequence (there is no deadline — but there is a hiring calendar, and the plan should not pretend otherwise):**

**Stage 1 — apply at the Phase 1 gate, not at the full proof pack.** The plan as written starts applications only after the complete core proof pack exists, which by the sizing table is 9–12 months out (Phase 0's reading alone is roughly 2,200–2,400 dense pages at the stated 100–140 pages/week, before any build). That is too long to go without interview reps, and interview loops are the instrument that tells you which gaps are real. The Stage-1 proof pack is: the Orderbook and Backtester as shipped (with the credibility patches in [`40-deepening-queue.md`](40-deepening-queue.md) landed), plus the admission core (Phase 1), the decoder (G) if it has landed, the correctness gate, and the ablation from Phases 1–2. That is a defensible portfolio. Write the Stage-1 README and design doc at that point, start applying to both tracks, and let the loop tell you what to fix. The gateway and scheduler then ship *while* you are interviewing, as upgrades, rather than as prerequisites.

**Stage 2 — full proof pack, remaining AI modules, AI variant live.** Unchanged from the original sequence, except that it now runs concurrently with an active job search rather than gating it.

**Review trigger (so a slipping plan has a defined failure state):** if the Phase 0–1 reading is not complete within about six months of start, stop treating the curriculum as a prerequisite and switch to applying with the current portfolio plus whatever has shipped, continuing the spine part-time. A plan with no trigger can silently consume a year.

Algorithms parallel throughout. If the core slips, the extension spine pauses; applications do not wait for every AI module.
