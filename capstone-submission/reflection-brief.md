# Reflection Brief — Evaluation and Observability Capstone

**Name:** Sayan Dutta  
**Date:** 2026-09-25  

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | macOS Darwin 25.6.0 (arm64) |
| Python version | Python 3.11.5 |
| Date run | 2026-09-24 |
| Ran any system live? (which) | No. Live Anthropic API execution was bypassed due to lack of production API key; all evaluation and validation evidence was generated using the repository's offline replay fixtures and validation test harness. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | `capstone-submission/01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 1 / 2 / 1 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> `{"kind":"escalation","policy_id":"POL-EMPTY","field":"coverage_limit","reason":"missing required field","retries_used":1,"status":"retry_futile_escalation"}`
>
> The system executed exactly **1** API call before halting. Retrying when a mandatory source field is entirely absent from the underlying document is actively harmful: because the input data contains zero information to extract, repeated attempts will merely incur unnecessary token costs, inflate operational latency, and risk prompting the model into hallucinating a plausible number. Immediate escalation halts wasteful compute loops and routes the deficit straight to human operators.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> From `routing_decisions.json`, policy `POL-101` was routed to `human_review` with the explicit reason:  
> `"reason": "fields_below_threshold=['premium_amount']"`  
> While the independent secondary reviewer and deterministic integration checks were fully satisfied, the granular confidence score for `premium_amount` plummeted to `0.65` (well below the required `0.90` threshold). Had the architecture naively relied on an unstratified global or mean confidence score across the document (where all 5 other fields were scored at a near-perfect `0.99`), the policy would have mistakenly auto-approved, slipping an unreliably extracted financial figure past auditing.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> In `calibration-report.txt`, the pathological slice is:  
> `umbrella  exclusions  n=2 conf=0.93 acc=0.00 brier=0.865`  
> contrasting sharply against the aggregate metric:  
> `OVERALL brier=0.291`  
> Slicing along the cross-product of policy category and target field reveals localized catastrophic overconfidence. While the global Brier score appears moderately acceptable (~0.29), it completely masks the fact that the model is 93% confident yet 0% accurate on umbrella policy exclusions. Slicing prevents severe systemic blind spots from being washed out by large volumes of easy, high-accuracy extractions.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `fixtures/documents/appraisal_informal_sqft.txt` |
| Classified type | `single_family` property; normalized `gross_living_area_sqft` to `2400` |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `capstone-submission/02-mortgage-extraction/discrepancy-run.txt`:  
> `{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0}, "consistent": false`  
> Tool-use enforces **syntactic shape and type constraints** (e.g., ensuring a field is a valid numeric float rather than an arbitrary string). The second-pass validator enforces **semantic domain and relational truth** (confirming that A + B + C = Total). Tool-use cannot detect mathematically flawed arithmetic within valid JSON; conversely, downstream mathematical validation cannot prevent the model from crashing the JSON parser if the model emits truncated, ill-formed syntax.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `capstone-submission/02-mortgage-extraction/missing-field-run.txt`:  
> `"stated_monthly_total": null`  
> The system returned `null` because the underlying Pydantic schema purposefully declares optional components as nullable (`Optional[float] = None` / union with `None`) paired with strict prompt instructions prohibiting assumption. This explicit schema affordance gives the LLM a valid, typed outlet to declare "unspecified in source text" rather than forcing it to confabulate an imaginary dollar amount to satisfy a non-nullable constraint.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> In `extract-run.txt`: `"gross_living_area_sqft": 2400` derived from natural language `"about 2,400 sq ft"`.  
> Normalizing during the extraction pass translates unstructured linguistic ambiguity (commas, abbreviations, approximation qualifiers) into a standardized canonical integer at the point of origin. This decouples downstream underwriting rule-engines and validation formulas from raw string regexes, guaranteeing that subsequent modules operate exclusively over clean, deterministic data types.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed in 60.14s |
| Briefing file | `capstone-submission/03-supply-chain/briefing.txt` |
| Section the conflict landed in | `Contested` |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> In `briefing.txt`:  
> `95.0 percent — supplier_audit (as of 2026-04-10)` vs. `78.0 percent — logistics (as of 2026-04-05)` for `on_time_delivery_rate`.  
> Rather than averaging the values into an artificial ~86.5% metric, preserving both distinct data points alerts operational stakeholders to a genuine real-world divergence. A procurement manager can see that internal shipping telemetry from April 5 showed acute fulfillment bottlenecks (78%), whereas the supplier's self-reported audit on April 10 claimed near-flawless performance (95%). Retaining attribution and timestamps empowers analysts to investigate supplier bias or timeline improvements rather than acting on a deceptive compromise.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `capstone-submission/03-supply-chain/timeout-run.txt`:  
> `> Sources unavailable: logistics unavailable (timeout)` and `late_shipment_count _[missing source: timeout reading logistics]_` under `Incomplete`.  
> An unreachable source is treated as an explicit epistemic unknown, distinct from a valid source reporting "0 defects" or "no events observed." The coordinator uses fault-tolerant resilient gathering (via exception catching and timeout wrappers), enabling the synthesis engine to gracefully degrade: it publishes verified intelligence from surviving nodes while explicitly warning the reader about the blind spot under `Incomplete`.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> In `briefing.txt`: `logistics (as of 2026-04-05)` reporting `78.0 percent` vs. `supplier_audit (as of 2026-04-10)` reporting `95.0 percent`.  
> Attaching temporal provenance turns what would otherwise look like an irreconcilable logical collision into an observable state change over time. Readers can discern that the audit took place five days after the shipping log, allowing for post-incident recovery or corrective actions to explain the shift.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> The standout demonstration occurred in System 2 (`capstone-submission/02-mortgage-extraction/discrepancy-run.txt`). The extraction model produced pristine, fully validated JSON that looked flawlessly authoritative at first glance. However, the external arithmetic validator flagged `"consistent": false` because the itemized wage stream summed to `$9,642.17` while the printed document header claimed `$10,892.17` (a `$1,250.00` discrepancy). An unvalidated pipeline would have naively persisted false financial solvency.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> This principle was most dramatically illustrated by System 1's calibration report (`calibration-report.txt`). For the `umbrella / exclusions` category, the extraction agent expressed extreme certainty (`mean_predicted_confidence = 0.93`), yet its actual accuracy was exactly zero (`observed_accuracy = 0.00`, resulting in a massive Brier score of `0.865`). High confidence is a measure of internal model activation, not ground truth; trusting raw confidence scores without empirical calibration and human-in-the-loop fallback creates catastrophic risk.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> In an automated medical insurance claims adjudication engine processing unstructured clinical notes and hospital billing summaries, I would implement **independent review with deterministic routing backed by validated retry escalation**.  
> 
> Primary extraction would capture billing codes and totals with schema enforcement. A secondary deterministic rule-layer would check claim codes against pre-existing policy caps and math totals. Any discrepancy or low per-field confidence would route immediately to human adjusters.  
> 
> To detect degradation in production, I would instrument:
> 1. **Field-level calibration drift**: Tracking rolling Brier scores across clinical specialty slices.
> 2. **Human routing rate**: Monitoring spikes in triage queues indicating upstream prompt or PDF format drift.
> 3. **Retry-to-escalation ratios**: Surfacing any jump in missing-source errors signaling changes in provider hospital templates.
