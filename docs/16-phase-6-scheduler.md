# Phase 6: The Scheduler (continuous batching)

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> Companion docs: [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) (why this phase exists) · [`13-phase-3-loadgen.md`](13-phase-3-loadgen.md) (the load generator this phase is measured under) · [`20-ai-extension.md`](20-ai-extension.md) (step G, the backend) · [`30-perf-playbook.md`](30-perf-playbook.md)

**Shape of this phase:** the first concurrent *policy* system in the project. Everything before it was a filter or a pipeline with a fixed path; here the path itself is the thing being designed — which requests run together, which wait, which are refused, and what that costs the ones already running.

**Gate:** phase 1 green (the admission path, the ledger, and the correctness machinery exist), step A shipped (roofline and bandwidth reasoning), and step G's decoder passes its token-for-token gate single-stream. The scheduler is a concurrency layer over a correct engine — never a way to make an incorrect engine look busy.

**Why this is the centerpiece:** continuous batching is the contribution that separates a serving *engine* from a serving *wrapper*. It is also the only part of this project whose design is policy rather than mechanism, which makes it the most interviewable thing built here — and the easiest to get subtly wrong.

---

## Rung 0 — 📖 Read, then 🔨 the naive reference scheduler

**📖 Read:** **Orca** (Yu et al., OSDI 2022) for iteration-level scheduling — the origin of the design — then **vLLM / PagedAttention** (Kwon et al., SOSP 2023) for KV paging and the memory-budget mechanism. Both are short. Read for the *decision* each paper makes and what it trades away, not for implementation detail. Skim **Pope et al. (2022)** for the batch-economics and KV-memory arithmetic you will reproduce in your own sweep.

**🔨 Build:** the deliberately naive reference: one request at a time, no batching, FCFS, no admission policy beyond "queue everything." Around a day.

**Why first:** it is the differential oracle, exactly as the naive admission path was in phase 1 — and it is row 0 of the batching table, the honest answer to "what did batching actually buy." Same reasoning as the reference implementation in [`11-phase-1-core.md`](11-phase-1-core.md): build it here or write it twice.

**Learn:** what a scheduler *is* — an ordering policy over work that arrives asynchronously — before it is an optimization.

---

## Rung 1 — 🔨 Continuous batching + KV-budget admission (the ship line)

**What:** iteration-level scheduling. Each decode step iterates over the resident batch; new requests join mid-generation at the next iteration boundary; finished requests drain and free their KV. Admission is a projection: a request enters only if `prompt_len + max_new_tokens` fits the KV budget (worst case, reject-at-capacity). No preemption at this rung.

**Reach for:** the KV allocator behind the interface chosen in the decoder phase — contiguous at this rung, replaceable without touching attention; iteration-boundary joining so no request waits for the whole batch to finish; an O(1) admission test against a running KV total; the same SPSC dispatch and admission-control patterns the gateway already uses.

**Why it pays:** the batch is the unit of throughput, and decode is bandwidth-bound — every additional resident sequence amortizes the same weight read. That is the whole economic argument, and §benchmarks is where it becomes your number instead of a paper's.

**🔓 Unlocked — decoder:** this is where KV *memory management* becomes a design problem rather than an allocation. The decoder's contiguous cache is the honest single-stream choice; paging exists because of this phase, and the interface you built in step G is what lets you say so with evidence.

**Decisions to document (one paragraph each):**
- Why worst-case projection and reject-at-capacity, versus admit-on-prompt and preempt-on-growth. Name what the alternative would cost (recompute or swap) and why it is a stretch rung rather than the default.
- Where the KV budget number comes from: total device/CPU memory, weights, activations, and the arithmetic from Pope et al. reproduced on your own machine.
- Fairness: what prevents one long-lived tenant from consuming the budget. State the policy and its failure mode.

---

## Rung 2 — 🔨 Chunked prefill

**What:** split a long prompt into chunks so prefill work interleaves with decode steps instead of stalling the batch. TTFT for the newcomer trades against TPOT for the residents *by policy*, not by accident.

**Why it is rung 2 and not rung 1:** it only becomes meaningful once rung 1's numbers exist to compare against. Its value is the delta it produces in the rung-1 curves — without those, it is unmeasurable and therefore unclaimable.

**Learn:** that prefill is compute-bound and decode is bandwidth-bound, and that co-scheduling them is a resource-partitioning problem — the same reasoning as mixing two workloads on one core, which is Phase 0 material applied to a new resource.

**Cut rule:** rung 2 is cut item 3 in the master's order. If time compresses, the scheduler ships at rung 1, the ladder is documented, and rung 2 is named as designed-but-unbuilt. Never claim it unbuilt.

---

## Correctness gate (phase is not done until all pass)

- **Output equality across batch compositions.** The same seeded request stream, run at batch size 1 and at capacity, must produce token-for-token identical outputs per request. Batching is a scheduling change, not a numerical one — if compositing changes a token, the bug is in your attention masking or position handling, and this gate is what catches it. Report it the way phase 1 reports replay: bit-identical, across runs and across thread counts.
- **Differential vs. rung 0.** Fuzzed request sequences (varying prompt lengths, generation lengths, and arrival patterns) through both schedulers; per-request outputs must match exactly. Divergence is a find, not a nuisance.
- **Admission invariants under fuzz:** resident KV never exceeds budget; every admitted request eventually completes or is explicitly refused; refused requests are refused once, with a reason; no request is admitted twice.
- **Concurrency edge cases, as fuzz invariants:** a request cancelled by the client mid-generation frees its KV exactly once; a client that disconnects without reading does not block drain; a batch where every request finishes on the same iteration; a single request longer than the budget (must refuse, not hang); arrival exactly at the budget boundary.
- **TSan clean** under concurrent arrivals and completions.
- **Explain-loop gate (shipping requirement):** cold, spoken, no notes — Orca's iteration-level scheduling and vLLM's paging, why each beats the naive design, and how your design differs and why. This is the feedback substitute the plan cannot otherwise provide.

---

## Benchmarks — the phase's actual output

Every number here is a table in the README's claims pack, generated by `bench/`, not transcribed by hand.

- **Throughput vs. per-token latency, by batch size.** The headline curve. Present it with the mechanism, not next to it: achieved DRAM bandwidth during decode against the roofline ceiling measured in step A. The curve's shape is the argument — it flattens exactly where bandwidth saturates.
- **TTFT and TPOT under concurrent load**, coordinated-omission-correct, on the phase-3 harness, at stated offered rates. Two SLOs, because admission trades one against the other.
- **Rung deltas.** Rung 0 vs. rung 1 vs. rung 2, same workload: what batching bought, what chunked prefill bought. A rung that buys nothing is a finding worth keeping and stating.
- **Scheduler overhead per admission decision** — nanoseconds against a per-token cost measured in milliseconds. This is the sentence that makes the gateway a systems project.
- **Overload behavior** on the phase-3 overload harness: what the shedding policy does when the offered rate exceeds capacity, whether TTFT degrades gracefully or collapses, and where the breaker trips.

**Writeups (both required):**
- **FP non-associativity** — why batched kernels can produce different bits than single-stream (reduction order, kernel selection), what your output-equality gate therefore proves about your implementation, and where the guarantee stops. COD 3.5 plus the ONNX Runtime execution-provider determinism docs.
- **The KV budget arithmetic** — your machine's numbers, your model, your context length.

**Artifact:** `scheduler/` + `scheduler/test/` + batching tables in `bench/`.
**Claims this phase should earn:** *"Iteration-level continuous-batching scheduler with KV-budget admission over a from-scratch C++17 decoder: [N]× throughput at [B]× TPOT vs. single-stream, explained by [P]% of peak DRAM bandwidth during decode; outputs token-identical to the single-stream reference across batch compositions and thread counts; TTFT/TPOT under concurrent load, coordinated-omission-correct."*

---

**🎯 Interview drills (Phase 6 ships → you answer these):**

- Walk through one scheduler iteration: who joins, who decodes, who drains, who is refused, and in what order you decide.
- Why is decode bandwidth-bound and prefill compute-bound, and what does that mean for co-scheduling them? (Answer from your own achieved-bandwidth figure.)
- What does the KV cache cost at 2k context on this machine, and how does that number set the batch ceiling? (Arithmetic, on a whiteboard.)
- Why reject at capacity instead of preempting? What would preemption cost, and what does vLLM do?
- How do you know batching did not change your outputs? (Output-equality gate, differential oracle.)
- What is your scheduler's overhead per admission decision, and why is that number the one that matters?
- What happens at 2× offered capacity? Which policy sheds, and what does the tail do?

**Next:** run the transfer checkpoint in [`00-master.md`](00-master.md) out loud; then a short docs pass to append this phase's sections (rung ladder, KV-budget arithmetic, FP writeup, batching tables) to the proof pack started in [`15-phase-5-docs.md`](15-phase-5-docs.md). Phase 5 runs before phase 6 in the v4 sequence, so the design doc is written expecting this append — leave the scheduler section stubbed, not improvised.
