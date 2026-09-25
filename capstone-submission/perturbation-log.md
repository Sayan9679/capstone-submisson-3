# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Modified a cloned policy fixture by stripping out the mandatory `coverage_limit` property entirely, simulating an incomplete document where the required data point does not exist.
- **Command I ran:** `.venv/bin/python -c "from policy_extractor.retry import extract_with_retry; ..."` / executed test suite missing-field evaluation pathway.
- **What I predicted:** The pipeline would recognize that the required field is structurally missing from the input, refrain from making futile retry calls or hallucinating a replacement limit, and trigger an immediate escalation.
- **What actually happened (paste the key output line):** `{"kind":"escalation","policy_id":"POL-EMPTY","field":"coverage_limit","reason":"missing required field","retries_used":1,"status":"retry_futile_escalation"}`
- **How this differs from the unperturbed run:** An unperturbed execution processes the document and outputs a standard extraction dictionary with `retries_used: 0`. The perturbed run terminates after a single pass and emits an escalation event tagged with `retry_futile_escalation`.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** In `fixtures/documents/income_sum_mismatch.txt`, retained all component salary line items (base, overtime, commission) but preserved an incongruous `stated_monthly_total` ($10,892.17) that does not match the actual arithmetic sum ($9,642.17).
- **Command I ran:** `mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:** The JSON extraction pass would extract all fields successfully, but the second-pass mathematical consistency validator would compute the component sum, detect the delta, and flag the record as inconsistent.
- **What actually happened (paste the key output line):** `"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0` with `"consistent": false`
- **How this differs from the unperturbed run:** On a clean document (such as `appraisal_informal_sqft.txt`), the validator produces `"consistent": true` and `"discrepancies": []`. Under perturbation, it explicitly catches the $1,250 discrepancy and populates the `discrepancies` array.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Injected an artificial upstream network failure into the logistics telemetry ingestion pipeline by executing the coordinator with the `--simulate-timeout` parameter.
- **Command I ran:** `supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The coordinator would catch the timeout gracefully, prevent the synthesis pipeline from crashing, and annotate the missing logistics metrics in the `Incomplete` section.
- **What actually happened (paste the key output line):** `> Sources unavailable: logistics unavailable (timeout)` along with `late_shipment_count _[missing source: timeout reading logistics]_`
- **How this differs from the unperturbed run:** The normal investigation compiles inputs from all available feeds, placing `on_time_delivery_rate` in `Contested` due to variance between audit and logistics. Under timeout perturbation, logistics is completely absent, causing uncorroborated metrics to shift and the missing feed to be highlighted under `Incomplete`.
