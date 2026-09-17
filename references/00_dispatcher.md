# Fact Vs Judgment Classifier — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [I wish I knew this before I started writing online](https://www.youtube.com/watch?v=iCaX5cZ6roo)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Epistemic Tag & The Dogma Pipeline

Every sentence you write about a system carries an **epistemic tag** — a hidden claim about *how the reader is allowed to disagree with it*. There are only two tags that matter in engineering prose:

- **FACT** — a verifiable data point. Someone else could reproduce it, read it in a spec, or query the same dashboard. Disagreement means "you measured wrong."
- **JUDGMENT** — a trade-off, preference, prediction, or interpretation. Reasonable engineers with the same data can land elsewhere. Disagreement means "I weight the trade-off differently."

Fitzpatrick's insight is that **unlabeled prose defaults to the FACT tag in the reader's mind.** Readers are cheap-assigners: they grant maximal authority to whatever you wrote and only downgrade it if you spend words inviting them not to. So an opinion written in the declarative voice — *"Postgres can't handle this workload"* — is read as law, not as a person's weighting. That is the machine that manufactures **dogma**: a private trade-off laundered into a public constant.

The cost is asymmetric and it is what makes this a high-leverage skill:

| Failure mode | Epistemic error | Consequence |
|---|---|---|
| **Dogma** (the expensive one) | Judgment written as Fact | Debate collapses; no one brings counter-evidence because the premise looks settled; the decision becomes unauditable and un-revisitable |
| **Underclaiming** (the cheap one) | Fact written as Judgment (*"we think latency might be sort of high"*) | Reader can't tell what's real; incident triage loses its ground truth; hedging erodes trust in the data itself |

Judgments stated as facts **raise the temperature** of a review; facts stated as judgments **erase the floor** it stands on. The operational fix is not "add more caveats" — caveats are noise. The fix is **classification**: split each claim, tag it, and let the tag dictate the grammar.

```text
BEFORE — Untagged prose (the Dogma Pipeline)
┌───────────────────────────────────────────────────────────────────────────┐
│  "Postgres can't handle this traffic. We obviously need to shard. Also    │
│   our p99 is 480ms and a vendor says Cassandra is better."                │
└───────────────────────────────────────────────────────────────────────────┘
                                  │  reader assigns max authority
                                  ▼
        ┌───────────────────────────────────────────────┐
        │  LAUNDERED: opinion ──▶ premise ──▶ mandate   │
        │  Counter-evidence has no door to enter.       │
        │  Reviewer who disagrees must attack a person. │
        └───────────────────────────────────────────────┘

AFTER — Classified prose (Tag-Segregated)
┌───────────────────────────────────────────────────────────────────────────┐
│  FACT  ▸ p99 write latency 480ms at 40k writes/s on node pg-07            │
│          (grafana/prod/writes-7d, 2026-02-11)                             │
│  FACT  ▸ cp-cache misses climbed 4% → 31% across the same window          │
│  JUDG  ▸ I'd treat this as single-node saturation and shard by tenant,    │
│          because our read path is already replica-served.                 │
│          Counter-case: if the miss spike traces to the schema change,     │
│          I'd revert that first and re-measure. (author: @pk)              │
└───────────────────────────────────────────────────────────────────────────┘
                                  │  reader assigns minimum authority to JUDG
                                  ▼
        ┌───────────────────────────────────────────────┐
        │  AUDITABLE: premise (F) and weighting (J) are │
        │  separable. Reviewer attacks either one.      │
        └───────────────────────────────────────────────┘
```

The mental model to hold: **you are not writing sentences, you are tagging claims.** Every paragraph should be readable as a ledger with two columns. When someone disagrees, the tag tells them *where* to aim: at your measurement (Fact) or at your weighting (Judgment).

---

## 2. Core Transformation Protocols

### Rule 1: The Epistemic Tag Pass
Read the draft and tag every **claim-bearing clause** with `F` (fact) or `J` (judgment). Ignore connective tissue. A sentence with two claims gets two tags. Deliverable: a tagged ledger, not a reworded paragraph.

### Rule 2: The Reproducibility Gate (Facts must be falsifiable)
A clause earns `F` only if you can hand over the artifact that would falsify it — a commit SHA, a benchmark command, a log line, a trace span, a spec clause, a line count, a version-pinned file path. **No artifact ⇒ it is a `J`**, no matter how confident it feels. Run it backward too: if the artifact *proves* the claim, it is a Fact and must be stated flatly.

### Rule 3: The Universality Ban (Judgments must not speak in constants)
Judgments may never be phrased as universal laws. Strike `always`, `never`, `obviously`, `clearly`, `everyone knows`, `the right way`, `simply`, `just`, `can't`, `won't scale`, `best practice`. Every surviving `J` is **owned** (first person or named team) and **scoped** (to a constraint, workload, org, or timeframe).

```text
BAD  (judgment in the universal voice):   "You should never use an ORM."
GOOD (owned + scoped):                    "Given two of our three prod
                                           incidents traced to ORM-generated
                                           SQL, I'd restrict ORMs to simple
                                           CRUD here."
```

### Rule 4: The Hedge-Strip / Hedge-Add Asymmetry
Apply **opposite** edits to the two columns — this is the part most writers get backwards:

| Column | Edit | Why |
|---|---|---|
| `F` Facts | **Strip** hedges: delete `I think`, `it seems`, `sort of`, `maybe`, `we believe`, `from what I can tell` | Hedging a reproducible measurement transfers authority to the wrong place and poisons the ground truth |
| `J` Judgments | **Add** ownership and a counter-condition: `I'd…`, `I weigh X above Y`, `I'd reverse this if…` | Ownership moves disagreement off the person's competence and onto the trade-off (Fitzpatrick: *"own the opinion, so the reader can own the counter-opinion"*) |

### Rule 5: The Trade-Off Form (the canonical Judgement template)
Every `J` is rewritten into the decision-relevant canonical shape:

```text
Given [constraint/workload], I'd choose [option],
because [weighted trade-off],
and I'd flip if [counter-condition].
```

A judgment that survives this template is **revisitable**: a reader who supplies the counter-condition changes the decision without insulting the author.

### Rule 6: The Dogma Detector (the high-value grep pass)
Scan for these grammar traps — each one is a judgment wearing a fact's clothes:

| Dogmatic Anti-Pattern (J masquerading as F) | Clean Replacement (tagged, scoped, owned) |
|---|---|
| *"Microservices are the right architecture."* | *"FACT: deploy cadence went 1/wk → 4/wk after the split. JUDGEMENT: I'd take that deal again, but only with a platform team — the ops surface tripled and 2 of 4 teams were underwater for a quarter."* |
| *"Postgres can't handle this workload."* | *"FACT: single-node pg-07 saturated at 40k writes/s, p99 480ms. JUDGEMENT: above ~10k writes/s I'd partition before scaling the box."* |
| *"This is obviously the cleaner approach."* | *"FACT: 120 LOC vs 340; 2 services touched vs 3. JUDGEMENT: I judge this cleaner."* |
| *"The tests are flaky."* | *"FACT: `ci/run:8812` failed 6/40 retries on `test_billing_retry`; all 6 on the 1st of the month. JUDGEMENT: I suspect a timezone fixture, not the queue."* |
| *"We should never use ORMs."* | *"JUDGEMENT (mine, scoped to our schema): ORMs only for simple CRUD. FACT: 2 of our 3 prod incidents traced to ORM SQL."* |
| *"Nobody wants a monolith in 2026."* | *"JUDGEMENT: for a 6-person team I'd pick a modular monolith. Counter-case: if we get a second deployment cadence owner, I'd split the boundary first."* |
| *"Latency is bad on this endpoint."* | *"FACT: p99 = 480ms at 40k writes/s (`grafana/prod/writes-7d`)."* |
| *"I'd argue the cache is fine."* | *"FACT: cache hit rate 69% → 31% over 7d. JUDGEMENT: I don't yet think that explains the p99 — I'd check eviction size first."* |

### Rule 7: The Calibration Ratio (paragraph-level audit)
Count your two columns. A reviewable engineering paragraph typically runs **facts first, then exactly one owned judgment**, with the judgment explicitly marked as separable. Ratios to treat as alarms:

- **Zero `F`, many `J`** → opinion piece wearing a technical-report costume. Convert the load-bearing claims into measurements or drop them.
- **Many `F`, zero `J`** → you dumped telemetry and left the decision to the reader. Add one tagged judgment so the document can actually move a decision.
- **`J` sentences outnumbering `F` in an RFC/ADR** → the doc cannot be approved because nothing is checkable.

---

## 3. Engineering Application Scenarios

### 3.1 Code Review Comments
The failure: a reviewer states architecture preference as a defect, so the author must either obey or argue with the reviewer's *taste* rather than the code.

```text
BEFORE (untagged — reads as FACT, starts a status fight)
  "This needs to be extracted into a service. This design won't scale."

AFTER (claim-split, owned judgment)
  FACT  ▸ This handler now owns 3 responsibilities and grew 180 → 610 LOC.
  JUDGEMENT ▸ I'd extract the notification path behind an interface *first*
         (smallest cut, no deploy-boundary change). I'd flip if you plan to
         deploy notifications separately — then I'd want a real service now.
```
Rules in play: **Rule 2** (LOC + path = the artifact), **Rule 3** (no `won't scale` universal), **Rule 6**. Note that the Fact is also the *safe* half of the request: the author can fix the Fact without conceding the Judgment.

### 3.2 PR Descriptions
The failure: the author narrates what they *believe* the change does, and the reviewer cannot distinguish intent from evidence — so the review re-litigates the design instead of verifying behavior.

```markdown
## What changed
- FACT: Adds `RetryPolicy` with exponential backoff; replaces 4 hand-rolled
  retry loops (grep: `time.Sleep` retries → 0 results).
- FACT: `go test ./internal/queue/... -race` green; `make bench` p99 unchanged
  within noise (480ms → 476ms, n=20).
- FACT: No schema change; no public API change.

## Judgment (mine, separable)
- JUDGEMENT: 3 attempts / 5s cap is right *for our current SLO*. I did NOT
  tune it per-endpoint — I'd revisit if the payment path starts needing
  shorter timeouts. Counter-case: if you want per-endpoint config now, say so
  and I'll add the interface before merging.
```
Rules in play: **Rule 1**, **Rule 7**. The judgment section is the *reviewer's door in* — it names the exact axis on which a reviewer may overrule without reopening the whole PR.

### 3.3 Architecture RFCs / ADRs
The failure: a decision record whose "Decision" section is a mandate built on unmarked opinion, so future engineers can't tell what evidence would force a revisit — the ADR ossifies into folklore.

```markdown
# ADR-014: Tenant-Partitioned Write Path

## Context — FACTS ONLY (each must be falsifiable)
| # | Fact | Artifact |
|---|------|----------|
| F1 | pg-07 saturated at 40k writes/s; p99 480ms | grafana/prod/writes-7d |
| F2 | cp-cache hit rate fell 69% → 31% over the same window | cache dashboards |
| F3 | 82% of write volume is 4 tenants | prod write histogram |

## Decision — JUDGEMENT (owned, scoped, with counter-conditions)
JUDGEMENT (@pk, 2026-02): given F1–F3, partition by tenant and keep
single-node for the long tail. Weighted trade-off: operational simplicity
below 10k writes/s vs. headroom above.
I'd reverse this if: (a) F2's miss spike traces to the schema change rather
than load, or (b) tenant skew drops below 60% of volume.

## Rejected alternatives — JUDGEMENTS with the fact that killed each
- Global sharding — killed by F3 (over-partitions the long tail).
- Vertical scale — I judge this buys < 1 quarter; no artifact yet, so this
  one is pure judgment and should be re-measured before relying on it.
```
Rules in play: **Rule 5** (canonical trade-off form), **Rule 2** (every Context bullet carries an artifact or is moved to Judgment), **Rule 7** (Context is all `F`, Decision is one owned `J`, and the un-measured alternative is *labeled as un-measured*).

---

## 4. Verification Checklist

- [ ] Every claim-bearing clause in the draft is tagged `F` or `J` — no untagged claims survive (Rule 1).
- [ ] Each `F` resolves to a concrete artifact (path, SHA, command, dashboard, spec clause); anything without one was demoted to `J` (Rule 2).
- [ ] No `J` contains a universal quantifier or verdict word (`always`, `never`, `obviously`, `clearly`, `everyone knows`, `can't`, `won't scale`, `best practice`) (Rule 3).
- [ ] Hedges run the correct direction: stripped from `F`s, and each `J` carries **ownership + scope + a counter-condition** (Rules 4–5).
- [ ] Paragraph-level calibration holds: facts lead, exactly one separable owned judgment follows, and any judgment-heavy doc without checkable facts has been flagged rather than shipped (Rule 7).