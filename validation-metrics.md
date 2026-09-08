# Validation metrics for C&Q document verification

A specification for the metrics behind the qualification review dashboard, written to be
read alongside `cq-verification-dashboard.html`. The dashboard is the output contract;
this document defines what each number means, how it is computed, and what it must not
be used to claim.

**Context.** LangGraph subgraphs verify manufacturing IQ/OQ packages against a rule set.
Ground truth exists for roughly six packages. The metric program is therefore built so
that only the *release gate* depends on labels, while everything measured per package is
reference-free.

**Two asymmetries drive every design decision below.**

1. A missed deficiency is a regulatory risk. A false alarm costs a few minutes of
   reviewer time. Metrics and thresholds are tuned accordingly — when in doubt, hold.
2. A silent gap (a rule that returns nothing, a page never read) is more dangerous than a
   wrong answer, because a wrong answer gets challenged and a gap does not. Coverage is
   measured explicitly rather than assumed.

---

## 1. Unit of evaluation

The unit is the **rule decision**, not the package. Six labeled packages is a small
document sample but ~400 labeled decisions, which is a workable base. Every metric in
this document is an aggregation over `RuleDecision` records.

```python
@dataclass
class Evidence:
    page: int
    quote: str                  # verbatim span the verdict rests on
    bbox: tuple | None = None   # optional, for scanned pages
    found_in_source: bool | None = None   # set by the citation validator

@dataclass
class RuleDecision:
    # identity
    package_id: str
    rule_id: str
    run_id: str
    system_version: str         # model + prompt + rule pack + retrieval config
    doc_type: str               # IQ | OQ

    # the verdict
    verdict: Literal["PASS", "DEFICIENCY", "UNCERTAIN", "NOT_APPLICABLE"]
    evidence: list[Evidence]

    # structured extraction (populate wherever the rule permits it)
    extracted: dict             # e.g. {"reading": 121.4, "unit": "degC",
                                #       "limit_low": 121.0, "limit_high": 124.0}
    computed_verdict: str | None # deterministic recomputation from `extracted`

    # reference-free signals
    run_verdicts: list[str]     # verdicts from k repeat runs
    second_reader_verdict: str | None

    # provenance
    node_trace: list[str]       # LangGraph nodes traversed
    retrieved_pages: list[int]
    latency_ms: int
```

**Design rule: push every rule toward extraction plus computation.** A rule phrased as
"is the reading within the acceptance criteria" should be implemented as *extract reading,
unit, and limits*, then evaluate the comparison in Python. This converts a judgment into
an extraction, which is far easier to validate, and it yields metric A2 for free. Rules
that cannot be decomposed this way (legibility, completeness of a narrative, whether a
handwritten annotation is a deviation) are the ones needing the most human oversight —
track them as a separate class.

---

## 2. Layer A — reference-free metrics

These run on **every package in production**. They need no labels because they compare the
system against the source document or against itself. They are the dashboard's
"How the system checks itself" panel.

### A1. Evidence traceability

**Question.** Does the quoted text actually exist where the system says it does?

**Formula.** For each `Evidence`, normalise whitespace and confirm the quote appears in
the extracted text of the cited page. Use exact match first, then a high-threshold fuzzy
match (token sort ratio ≥ 95) to absorb OCR noise. Fail otherwise.

```
traceability = decisions where all evidence spans resolve / decisions with evidence
```

**Threshold.** Any decision with an unresolvable citation is forced to `UNCERTAIN`. This
is a hard rule, not a reported statistic.

**What it catches.** Fabricated quotes, page-number drift, evidence copied from a
neighbouring section. It is the cheapest hallucination detector available and involves no
model at inference time.

**Failure mode.** Passing this check does not mean the quote *supports* the verdict, only
that it exists. A system can cite a real sentence and reason wrongly from it. Pair with A4.

### A2. Deterministic recomputation agreement

**Question.** When the arithmetic is redone in code, does it match what the model said?

**Formula.** For every decision with a populated `extracted` block, evaluate the rule
deterministically and compare to `verdict`.

```
recompute_agreement = decisions where computed_verdict == verdict / decisions recomputable
```

Report the denominator prominently — a high agreement rate over 20% of rules means little.
Track `recomputable_share` as a second-order metric and drive it upward over time; it is
the single best proxy for how much of the system is verifiable rather than trusted.

**Threshold.** Any disagreement is forced to `UNCERTAIN`. Log the pair for triage — a
disagreement almost always means a misread value, unit, or limit, which is a specific and
fixable bug.

**What it catches.** Misread decimals, transposed limits, unit confusion (°F reading
compared to a °C limit), sign errors, comparisons against the wrong column.

### A3. Self-consistency across repeat runs

**Question.** Does the system give the same answer when asked again?

**Formula.** Run each check `k` times (k=3 is usually enough) with varied seed and, ideally,
permuted chunk order so the variation covers retrieval and not just sampling.

```
consistency = decisions where all k runs agree / all decisions
```

**Threshold.** Unanimous → proceed. Any disagreement → `UNCERTAIN`.

**What it catches.** Ambiguous source pages, borderline readings, prompt fragility. Item
level instability is a strong predictor of item-level error, which makes this a per-decision
confidence score obtainable with zero labels.

**Cost note.** This multiplies inference cost by k. Apply it selectively: always on rules
in the high-risk class, sampled at 20–30% on rules with a long clean track record.

### A4. Cross-checker agreement

**Question.** Does an independent reader reach the same verdict?

**Formula.** A second pass with a different prompt formulation (and preferably a different
model) evaluates the same rule on the same page without seeing the first verdict. Report
raw agreement and Cohen's kappa, since raw agreement is inflated when one verdict dominates.

```
agreement = matching verdicts / compared decisions
kappa     = (p_observed - p_expected) / (1 - p_expected)
```

**Threshold.** Disagreement → `UNCERTAIN`.

**What it catches.** Ambiguity, under-specified rules, prompt-specific quirks.

**Failure mode — state this whenever the metric is presented.** Two LLMs share failure
modes. High agreement is evidence of stability, not of correctness, and a systematic blind
spot will be agreed upon confidently by both readers. Layers B and C exist precisely
because A4 cannot detect this.

### A5. Coverage — rules answered and pages read

**Question.** Did anything get silently skipped?

**Two independent formulas:**

```
rule_coverage = rules returning an explicit verdict / rules applicable to the package
page_coverage = pages retrieved by at least one node / pages in the package
```

`NOT_APPLICABLE` counts as an explicit verdict *only if* the decision carries a reason and
evidence. An empty or malformed response does not count.

**Threshold.** `rule_coverage` below 100% blocks the package outright — it is a pipeline
failure, not a finding. Uncovered pages are listed for reviewer attention rather than
blocking, since some pages legitimately carry no rule-relevant content.

**What it catches.** The dominant hidden-recall failure. A subgraph that errors out and
returns an empty list looks identical to a clean package unless coverage is measured.

### A6. Metamorphic stability

**Question.** Does an immaterial change to the document change the verdict?

**Method.** Apply transformations that must not alter any verdict, then measure verdict
flips:

- re-render or re-export the PDF
- re-scan at a different DPI; apply mild skew or noise
- reorder independent sections
- split a table across a page boundary
- rename a field label to a known synonym ("Calibration Due" → "Next Cal Date")
- insert irrelevant content elsewhere in the package

```
stability = decisions unchanged under transformation / decisions tested
```

**Threshold.** Report per transformation. A flip rate above ~2% on any single
transformation indicates the pipeline is keying on layout rather than content.

**What it catches.** Brittleness to scan quality and formatting — the leading cause of
inconsistency on executed protocols, which are scanned, hand-annotated artifacts.

**Cost note.** This is the most expensive Layer A metric. Run it nightly on a rotating
sample rather than on every package.

### A7. Schema and internal consistency

**Question.** Is the output structurally sound and self-consistent?

**Checks (all deterministic):**

- output validates against the `RuleDecision` schema; enums are legal; numerics parse
- units are recognised and convertible
- date ordering is sane: `calibration_date < test_date < calibration_due_date`
- entity consistency: the same equipment ID, protocol number, and revision appear
  consistently across sections of one package
- referenced attachments and deviations exist in the package

**Threshold.** Schema failure blocks. Consistency failures force `UNCERTAIN`.

**What it catches.** Malformed generations, and a surprising number of real document
defects — inconsistent equipment tags across a package is itself a finding.

---

## 3. Layer B — synthetic defect injection

**This is the highest-leverage way to obtain recall figures without labeling effort**, and
it deserves priority over expanding the hand-labeled set. Take known-good packages and
programmatically inject defects. Each mutation is ground truth by construction, so recall
per defect class can be measured over hundreds of generated cases.

### B1. Defect recall by class

Maintain a mutation catalog. A starting set, each mapped to the rule it should trigger:

| Mutation | Should trigger |
|---|---|
| Blank a result field | Incomplete record |
| Remove a signature or date | Signature/date missing |
| Shift a calibration due date to before the test date | Calibration expired at test |
| Push a measured value just outside the acceptance limit | Result outside criteria |
| Push a value just *inside* the limit (negative control) | Nothing |
| Change units without changing the number | Result outside criteria |
| Swap an equipment ID for a valid-looking wrong one | Equipment ID mismatch |
| Change the protocol revision in the executed copy | Revision mismatch |
| Reference a deviation that is never closed | Deviation not closed |
| Remove an attachment; degrade one until illegible | Attachment missing/illegible |
| Reorder test steps so a dependency runs first | Sequence violation |
| Delete a page | Coverage / completeness |

```
defect_recall[class] = injected defects detected / injected defects
```

**Design points that make this metric honest:**

- **Include negative controls.** Mutations that should change nothing must not produce a
  flag; otherwise you optimise recall by flagging everything.
- **Vary the margin.** A value 0.05 outside the limit and one 5.0 outside are different
  difficulties. Report recall banded by margin, or the number flatters the system.
- **Inject into realistic locations** — inside scanned tables and handwritten margins, not
  just clean digital text.
- **Keep the generator versioned** and treat its output as test data under change control.

**Threshold.** Set per class, weighted by risk. Calibration and acceptance-criteria classes
should be held near 100%; attachment legibility will be lower and that is worth stating
openly rather than hiding.

### B2. Perturbation invariance

The labeled counterpart to A6: apply semantic-preserving transformations to a package with
known verdicts and confirm the verdicts hold. Same formula as A6, but with a known-correct
baseline rather than only self-comparison.

---

## 4. Layer C — the benchmark release gate

Six hand-labeled packages (~400 decisions). This is the tab in the dashboard, and its
purpose is **regression detection on system change**, not accuracy estimation.

### C1. The gate metric — paired decision diff

Run the current and previous system versions over the same benchmark and diff at the
decision level.

```
fixed        = decisions wrong in v_prev and right in v_new
broken       = decisions right in v_prev and wrong in v_new     <-- the gate
new_holds    = decisions cleared in v_prev, now UNCERTAIN, correctly
lost_holds   = decisions held in v_prev, now cleared incorrectly
```

**Gate condition: `broken == 0` and `lost_holds == 0`.** Anything else blocks the release
and requires an explicit, documented override.

**Why the diff rather than aggregate recall.** With ~400 decisions, a change that flips
five is plainly visible in a paired comparison but moves aggregate recall by about one
point — well inside the noise of a sample this size. The paired test is far more sensitive,
and it is the question you actually care about: *did this change break something that used
to work?*

### C2. Recall and precision, reported with intervals

```
recall    = true deficiencies found / true deficiencies present
precision = confirmed deficiencies / flagged deficiencies
```

**Always publish the Wilson score interval and the denominator.** At n≈400 decisions and
~100 deficiencies, a point estimate of 98% carries an interval of roughly ±3 points. A
naked "98%" invites someone to find a counterexample and discredit the whole programme; a
stated interval survives that.

These are context on the dashboard, not the headline. C1 is the headline.

### C3. Holdout discipline

**Split the benchmark: four packages for iteration, two locked.** The locked packages are
never inspected during tuning and are used only for the gate run. Without this, months of
iteration will fit the system to six documents and the number stops predicting anything
about a seventh. Mark the locked subset visibly on the dashboard — the lock is what makes
the number credible to a reviewer.

Rotate a locked package into the working set only when you add a fresh replacement, and
record the rotation.

### C4. Labeling provenance

Record, for each benchmark package: who labeled it, when, against which rule pack version,
and how disagreements were resolved. Two independent labelers with adjudication is the
defensible standard. When the benchmark eventually contradicts a reviewer in production,
the first question asked will be who decided the ground truth was ground truth.

### C5. Gate trigger conditions

Rerun the gate on **any** change to: model version or endpoint, prompts, rule definitions,
retrieval configuration, chunking, OCR pipeline, or extraction schema. Silent provider-side
model updates are the usual way a validated system drifts out of state, so pin model
versions explicitly and schedule a periodic gate run even when nothing has changed.

---

## 5. Layer D — production feedback

Reviewers are already producing ground truth. Capture it and this becomes, within months, a
larger and more representative labeled set than anything hand-built — and one distributed
like real work rather than like a curated sample.

### D1. Override rate

Structure the review UI so every finding is explicitly **accepted, edited, or rejected**.

```
override_rate[rule_id] = (edited + rejected) / flagged
```

Track by rule and by document type. The trend matters more than the level — a falling
override rate is the clearest evidence of improvement available, and it is what the
dashboard trend panel shows.

### D2. Reviewer additions — the only true recall signal in production

Deficiencies the reviewer found that the system missed. Make this a first-class, low-
friction control in the UI, because it is the only production measurement of recall.

```
production_miss_rate = reviewer-added deficiencies / total confirmed deficiencies
```

If this is hard to log, it will not be logged, and you will lose the most important number
in the programme.

### D3. Blind audit sampling

Override rate alone is biased — it only sees what the system flagged. Add a fixed random
audit: each period, re-review a percentage of packages by hand, **blind to the system
output**, plus every package containing a critical deficiency.

```
audit_recall = deficiencies caught by system / deficiencies found in audit
```

State the sample basis on the dashboard. If the audit was performed with the system's
findings visible, the confirmation rate is inflated and will not survive scrutiny.

### D4. Operational outcomes

```
auto_clear_rate    = PASS decisions / total decisions
escalation_rate    = UNCERTAIN decisions / total decisions
review_hours_per_package     (measure the manual baseline properly, or it will be challenged)
cycle_time         = execution complete -> approved
```

These drive the dashboard funnel. Report `auto_clear_rate` **at a fixed escalation policy**
— throughput and safety trade against each other, so a clear rate quoted without the policy
that produced it is meaningless.

---

## 6. Escalation policy

Layer A signals combine into a hold decision. Make this explicit, versioned, and testable
rather than leaving it implicit in prompt wording.

```python
def escalate(d: RuleDecision) -> bool:
    return (
        any(e.found_in_source is False for e in d.evidence)     # A1
        or (d.computed_verdict is not None
            and d.computed_verdict != d.verdict)                 # A2
        or len(set(d.run_verdicts)) > 1                          # A3
        or (d.second_reader_verdict is not None
            and d.second_reader_verdict != d.verdict)            # A4
        or not schema_and_consistency_ok(d)                      # A7
        or (d.rule_id in HIGH_RISK_RULES and d.confidence < T)
    )
```

Two properties to preserve:

- **Holds are cheap, clears are expensive.** Every condition resolves toward `UNCERTAIN`,
  never toward `PASS`. The cost of a hold is reviewer minutes; the cost of a wrong clear is
  a deficiency reaching an approved package.
- **The escalation counts should reconcile with the funnel.** The number of items held by
  Layer A checks, after de-duplicating items that trip more than one check, should equal
  the `UNCERTAIN` count shown in the dashboard funnel. If those two numbers disagree, the
  instrumentation is wrong somewhere.

---

## 7. Instrumenting the LangGraph subgraphs

Emit metrics **per node**, not only per verdict. When a decision is wrong you need to know
whether retrieval never surfaced the page or the reasoning misread it — that distinction is
the difference between a number and a fix.

| Node | Emits |
|---|---|
| Ingest / OCR | page count, OCR confidence distribution, legibility flags |
| Chunk / index | chunk count, coverage of source pages |
| Retrieve | pages retrieved per rule, retrieval hit position, unread pages (A5) |
| Extract | schema validity, field null rate, unit parse failures (A7) |
| Evaluate rule | verdict, evidence spans, k-run verdicts (A3), second reader (A4) |
| Recompute | computed verdict, agreement (A2) |
| Validate | citation resolution (A1), consistency checks (A7) |
| Aggregate | coverage (A5), escalation decision, package roll-up |

Persist every run with its `system_version`, so any dashboard figure can be reproduced and
any regression can be attributed to a specific change.

**Design the evaluators as graph nodes, not as an offline script.** A1, A2, A5, and A7 are
deterministic and cheap enough to run inline, which is what lets them gate individual
decisions rather than merely describe a batch afterwards. A3, A4, and A6 are expensive and
belong on a sampled or nightly path.

---

## 8. What each metric maps to on the dashboard

| Dashboard element | Metrics behind it |
|---|---|
| Funnel — checks run, cleared, flagged, uncertain | D4 auto-clear, escalation counts from §6 |
| Review time, cycle time, hours returned | D4 |
| "Did it miss anything?" | D3 blind audit; named misses from D2 |
| "Reviewers are changing fewer findings" | D1 override trend |
| "How the system checks itself" | A1–A7, one row each, all-packages denominator |
| Release check tab — gate verdict | C1 paired diff, `broken == 0` |
| Release check — deficiencies caught, false alarms | C2 with intervals |
| Release check — held-back packages | C3 |
| Release check — version history | C5 gate runs, including blocked releases |
| (not yet on the dashboard) | B1 defect recall by class |

B1 belongs on the release tab once the mutation suite exists — it is a far stronger recall
claim than the six-package benchmark, because the sample size is unbounded.

---

## 9. Framing for business users

Translate before presenting. Model metrics do not persuade a QA lead; reviewer outcomes do.

| Internal metric | Presented as |
|---|---|
| Recall | "Of 240 audited findings, the system flagged 238" |
| Precision | "About 1 in 12 flags is nothing, costing ~3 minutes each" |
| Escalation rate | "78% auto-cleared; 22% still needs your eyes" |
| A3/A4 instability | "It tells you when it is unsure instead of guessing" |
| A1 traceability | "Every finding links to the exact page and line" |
| C1 broken count | "No check that used to work stopped working" |

State denominators and sample bases everywhere. Surface failures voluntarily — naming the
two audit misses and what was fixed buys more credibility than a clean deck.

---

## 10. Validation posture

This system supports a GxP process, so the pipeline itself is likely in scope for
computerised system validation. The usual expectations are a risk-based approach: a defined
and versioned test set, documented acceptance criteria per rule class, ongoing monitoring,
change control over prompts and model versions, and a named human accountable for the final
qualification decision.

Design accordingly:

- The system **prepares and evidences** the review. It does not approve packages.
- Treat prompt, model, and rule pack as controlled configuration with recorded versions.
- Retain full run logs and evidence spans so any decision can be reconstructed on demand.
- Keep the benchmark, the mutation catalog, and the escalation policy under version control
  as test artifacts.

Confirm the specifics with your quality and regulatory group — the above is engineering
practice, not a compliance opinion.

---

## 11. Suggested build order

1. **`RuleDecision` schema and per-node logging.** Nothing else can be computed without it.
2. **A1, A5, A7** — deterministic, cheap, no model calls, and they catch the most dangerous
   failure class (silent gaps and fabricated citations).
3. **Escalation policy (§6) wired into the graph**, with counts reconciling to the funnel.
4. **A2** — plus the refactor that pushes rules toward extract-then-compute. This is the
   largest single quality gain available.
5. **Layer C benchmark harness** with the paired diff gate and the locked holdout.
6. **Layer D capture** in the review UI — accept/edit/reject and reviewer additions.
7. **A3, A4** on a sampled path, once cost is understood.
8. **Layer B mutation suite.** Highest value per unit of effort for recall claims; sequenced
   here only because it depends on a stable schema and harness.
9. **A6** nightly on rotation.
