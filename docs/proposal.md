# Project Proposal

**Course:** Formal Methods for Software Engineering
**Group:** 15 
**Topic:**  UPPAAL Modeling and Verification of Real-Time Systems
**Title:** Formal Modelling, Verification and Statistical Analysis of Constrained Federated Learning Rounds in UPPAAL

---

## 1. Topic

In federated learning (FL), a server repeatedly sends a model to several clients. The clients train locally, upload their updates over a shared network and the server aggregates the updates once enough of them have arrived to a global model. Correctness and timing depend on many interacting design choices. When does the server proceeds to aggregation, how clients share the network, how long local training takes and what happens when clients are slow or fail.

We build a **layered, configurable UPPAAL model** of such a round, extend it step by step. For every layer we **state precise properties and check them with the  UPPAAL verifier**. The deliverable is a set of **claims, each backed by a query and a result.**.

The FL learning itself is abstracted, because SGD is difficult to model in that context. Abstraction is also a key concept in formal software engineering, which is why this project is applicable for this course. We do not train neural networks and do not prove convergence. We model and verify the *system behaviour* around the learning: protocol, timing, communication, scheduling and a progress abstraction.

## 2. What we want to achieve?

| Course requirements (from the assessment sheet) | What we do |
|---|---|
| Clearly course-related, scope fits 8-9 people | UPPAAL timed automata, CTL model checking, statistical model checking with SMC; eight parallel tracks or layers/extensions |
| Substantive formal-methods work (modelling, specification, verification) | Each layer has a model, a written specification/requirement (properties) and verification results |
| Main results are checkable and reproducible | rerunable queries, expected outputs, parameters and seeds documented |
| Report: problem, methods, results, individual contributions at the end | Report plus `CONTRIBUTIONS.md` maintained by contributors |
| Each member can explain their work | Partner review: nothing is done until a second person has reviewed and can explain it, also PRs to support that principles |
| AI cannot replace verification, log substantive prompts | Every AI-generated model, property or script must pass the UPPAAL verifier or be revieved before implementing:`AI_Usage_Log.md` kept continuously |
| Deliverables: report PDF, `AI Usage Log.md`, PPT | Kept in Repo |

**What is needed?**

1. A **property catalog**: every claims/Requirements with an ID, a plain-language statement, a formal query, a query type (safety, liveness, timing, quantitative) and an expected result.
2. **results, not only "true"**: best and worst cases with round times. For example, the largest local-training setting that still meets the deadline or the smallest scheduler slot that avoids starvation etc.
3. **Counterexamples from Verifier**:  the verifier's trace is the evidence.
4. **Reproducibility** on a clean machine.
5. **Exhaustive and Statistical tests**: what is proved exhaustively, what is only statistical, what is an assumption.

## 3. Principles

- **Claims first.** Before building a feature, write the properties/requirements it should enable. No new property, no new feature.
- **Layers** Each layer is a new frozen model file that works on its own. If time runs out, the previous layer is still a complete project.
- **Assumptions and ideas are written down** in `docs/decisions_and_ideas.md`, especially all numeric parameters.
- **Exhaustive where possible, statistical where necessary** and every result is labeled as one or the other.
- **AI is a helper, the verifier is the judge.**

## 4. Starting point (baseline)

`models/00_baseline.xml` (frozen).

**Model.** One server and three clients (single round).
- Server locations: `Init → Waiting → Check (committed) → Aggregating → Done`, or `Failed` if the deadline passes first.
- Parameters: quorum `K = 2`, `T_DEADLINE = 20`, aggregation time in `[A_MIN, A_MAX] = [1, 3]`, upload time in `[NET_MIN, NET_MAX] = [1, 4]`.
- Clients: training time in `[tmin, tmax]`: C0 = [2, 4], C1 = [3, 6], C2 = [5, 30]. Then upload.
- Late uploads are accepted and ignored so that no client ever blocks.

**Results already established**

| Query | Result |
|---|---|
| `E<> S.Done` | true |
| `A[] not deadlock` | true |
| `A<> (S.Done or S.Failed)` | true |
| `A[] (S.Aggregating imply count >= K)` | true |
| `A<> (S.Done and gclk <= 13)` | true |
| `E<> S.Failed` | false (for K = 2) |


**Known weaknesses/extensions to fix first**
1. Single round only; `start` fires once and `gclk` is never reset.
2. Clients are anonymous (`count` does not know who uploaded).
3. The unlabeled self-loops in `Done` and `Failed` hide real deadlocks. Replace the check by `A[] (deadlock imply (S.Done or S.Failed))`.
4. No learning, no network model, no failures, no randomness yet.

## 5. How we extend the baseline

The model becomes **one configurable system**. Each track below adds a part and a set of properties.

### Idea A: Protocol core
**Add:** multiple rounds (a per-round clock), client identities , round tags  so the server counts an upload only if it belongs to the current round and has not been counted and a rule for clients still training when a new round starts (abandon, or upload a stale update).
**Verify:** all R rounds terminate, no client counted twice, no stale update aggregated, quorum respected in every round, no deadlock outside terminal states. Show the counterexample when the round check is removed.

### Idea B: Synchronization
**Add:** switchable modes: wait-for-all, quorum (baseline), deadline-based (aggregate whoever arrived if at least `Kmin`) and optionally asynchronous with bounded staleness `s`.
**Verify:** wait-for-all has unbounded round time if a client's compute time is unbounded (counterexample) and bounded otherwise etc. Proof based on mode.

### Idea C1: Scheduler (shared uplink)
**Add:** a `Scheduler` template with simultaneous uploads and policies: FIFO, round robin (slot Q), and optionally priority and TDMA.
**Verify:** 

### Idea C2: Channel and faults
**Add:** a two-state channel (Good/Bad) that changes upload rate, message loss with bounded retransmission, optional background traffic, and client crashes (at most `F`).
**Verify:** 

### Idea D: Learning abstraction
**Add:** parameters (update size `S`, local epochs `E`, device speed classes) from which training and upload times are *derived* (instead of free numbers), plus an integer `progress` that increases after each aggregation by `gain(k, E)`, saturating at a target. Optionally calibrate `gain` with measurements from a small real federated run.
**Verify:** progress never decreases; the target is reachable, earliest and latest **time-to-target** per configuration, the largest `E` that still meets the deadline.
**Note:** this verifies system behaviour under stated assumptions, not SGD convergence.

### Idea F: Statistical model checking (UPPAAL SMC)
**Add:** random training and network times, random channel changes and failures.
**Verify (quantitative):** `Pr[<=T](<> S.Failed)`; `E[<=T; N](max: gclk)`; policy comparison; sweeps over N, K, deadline and failure rate.

### Optional extensions
- Adaptive deadline or quorum (controller that stays within bounds, no oscillation, still terminates).
- A security angle: an attacker that replays old updates.
- A small Lean lemma

### The verification matrix
The central result is a table of properties against configurations (rows: properties, columns: sync mode × scheduler), with ✓, ✗ (counterexample) or an exact bound

## 6. Properties and requirements

### Property catalog
Every claim/requirements lives in `docs/property_catalog.md`, one row per property:

`ID | layer | plain-language claim | UPPAAL formula | type | method (exhaustive/SMC) | expected result | status`

**ID scheme:** Ideas here?

**Property types**

| Type | Meaning | Example |
|---|---|---|
| Safety | something bad never happens | `A[] (S.Aggregating imply count >= K)` |
| Liveness | something good eventually happens | `A<> (S.Done or S.Failed)` |
| Reachability (also sanity) | a situation is possible | `E<> S.Done` |
| Quantitative | probability or expectation | `Pr[<=100](<> S.Failed)` |


### Exhaustive vs. statistical
- Exhaustive verification (`verifier`) uses small N (3-4) and yields proofs or counterexamples.
- SMC handles larger N (for example 8-16) and yields estimates with a stated confidence level.
- Every result in the report is labeled with its method. All stochastic parameters are stated assumptions.

## 7. Repository and workflow

### Structure

```
fl-uppaal/
├─ README.md                    short summary, how to run, claim → file map
├─ CONTRIBUTIONS.md             who did what, updated weekly
├─ docs/
│  ├─ proposal.md               this document
│  ├─ property_catalog.md       all claims
│  ├─ decisions_and_ideas.md              assumptions and design decisions
│  └─ AI_usage_log.md           kept from day one
├─ models/
│  ├─ 00_baseline.xml           frozen, never edited
│  ├─ 01_..., 02_...            one file per milestone
│  └─ final.xml
├─ queries/                     named by property ID
├─ scripts/
├─ results/
├─ report/                      report source
└─ slides/
```
## 8. Team structure 

Table here.

## 9. Deliverables

Folder `Group X - Group Project - Project Title/` containing:
1. `Project Report.pdf`, with each member's individual contribution at the end (printed copy at the presentation).
2. `Project Source Code.zipor repo link`: models, queries, scripts, expected results, README with run instructions.
3. `AI Usage Log.md`: substantive prompts and the project stage in which they were used.
4. `Project PPT.pptx`, optionally with a short screen recording (a counterexample trace, an SMC run).


## 10. Scope tiers
here we can define our scope

## 11. Non-goals

- No training of real neural networks and no convergence proofs.
- No claims about real network performance; results hold for the model and its stated assumptions.
- Not everything is checked exhaustively; large N is statistical and labeled as such.
- No simulation-only results counted as formal work.

## 12. Timeline

coming soon.

