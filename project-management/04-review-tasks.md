# Review tasks — Mostafa

Rows in the management plan where Mostafa is a reviewer. Following the review rules, log feedback in GitHub (PR review or issue comment) during the review window. Model rows need checks of metrics, claims, figures, and reproducibility.

**How this file works:** the checklists guide each review. The review log at the end records what was found and whether it was resolved. A row is `[open]` until its review starts, `[in review]` while feedback is outstanding, and `[closed]` once the owner has resolved it. The log is the evidence for the plan's rule that work is done only after reviewer feedback is resolved, and it is useful for peer evaluation.

## Review checklist for every model row

- [ ] Code and environment are committed, and the notebook or script runs from the repo
- [ ] Dataset source, license, and manifest entry are recorded
- [ ] The split is grouped (recording, measurement, or file), with no leakage into test
- [ ] Thresholds and model selection use validation only; the test split is scored once
- [ ] Macro F1 and per-class recall are reported alongside accuracy; the confusion matrix is shown
- [ ] A validated handoff CSV exists (`python src/validate_model_outputs.py`) and its labels match the agent label map
- [ ] Scores are called "calibrated" only if calibration evidence exists
- [ ] Claims stay within the evidence: no fusion, field, or new-site claims

## Row 8 — Thermal vision planning and data prep (owner Jacob; review Nov 7–8, with Tony)

- [ ] Get the link to the `thermal-ir-pipeline` branch or PR (the repo shows results there)
- [ ] Dataset 2: license, access, labels (drone boxes; birds as hard negatives)
- [ ] Recording-disjoint split manifest; every-fifth-frame sampling documented
- [ ] Exclusion rules (corrupt JPEGs, invalid boxes) applied and counted
- [ ] Output labels compatible with the agent (`drone` / `no_drone`)

## Row 7 — Radar model evaluation and limitations (owner Tony; review Nov 14–15, with Jacob)

- [ ] Baseline vs RadarCNN results match `docs/radar-model-results.md`
- [ ] Error analysis covers drone↔human confusion; compare with the EDA finding on D4/D5 Doppler spread
- [ ] Limitations: session/subject overlap unverified, target-centered segments, no continuous detection
- [ ] Calibration (ticket 6): temperature scaling on validation, calibration error reported, `calibrated_score` added
- [ ] `radar-test-outputs.csv` passes the validator

## Row 9 — Thermal vision training and evaluation (owner Jacob; review Nov 14–15, with Tony)

- [ ] Reported 97.44% drone recall and 0.66% bird false-positive rate are reproducible from committed artifacts
- [ ] The threshold-selection procedure uses validation only
- [ ] Model weights: confirm whether `best.pt` and `last.pt` belong in git (size, license) or should go to a release or external storage
- [ ] Handoff CSV exported and validated (ticket 9)

## Row 16 — Results, error analysis, and figures (owner Tony; review Nov 27, with Jacob)

- [ ] Every number traces to a file in `results/`
- [ ] Per-model results are kept separate from simulated agent results
- [ ] At least 8 visualizations across the report and presentation; each is discussed in the text
- [ ] The acoustic and agent figures are correct and labeled (yours)

## Row 18 — Full report draft and integration (owner Jacob; review Nov 29, with Tony)

- [ ] The template structure is followed: abstract, keywords, Introduction, Data Summary, Literature Review, Methodology, Results, Conclusion, APA 7 references
- [ ] 20 pages maximum, including figures and references
- [ ] The acoustic and agent sections match your handed-off content
- [ ] Wording consistency (ticket 13): Lattice is "optional extension"; "calibrated" only where evidence exists; agent results labeled simulated
- [ ] The AI-assistance disclosure is accurate

## Rows 19–21 — Final reviews (all members)

- [ ] Row 19 (Dec 2): source and claim review of the final report
- [ ] Row 20 (Dec 5): slides and recording review; check each speaker's time (at least 2 minutes)
- [ ] Row 21 (Dec 6): playback check and final repository review before Tony submits

---

# Review log

*Status as of 2026-11-07: no reviews have started. Row 8's window is Nov 7–8.*

| Row | Owner | Review window | GitHub link | Findings | Resolved? | Status |
| --- | --- | --- | --- | --- | --- | --- |
| 8 — Thermal planning and data prep | Jacob | Nov 7–8 | | | | `[open]` |
| 7 — Radar evaluation | Tony | Nov 14–15 | | | | `[open]` |
| 9 — Thermal training and evaluation | Jacob | Nov 14–15 | | | | `[open]` |
| 16 — Results and figures | Tony | Nov 27 | | | | `[open]` |
| 18 — Full report draft | Jacob | Nov 29 | | | | `[open]` |
| 19 — Final report claim review | All | Dec 2 | | | | `[open]` |
| 20 — Slides and recording | All | Dec 5 | | | | `[open]` |
| 21 — Playback and final repo review | Tony | Dec 6 | | | | `[open]` |

## Review notes

Use one block per review:

```
### Row N — <title> (reviewed <date>)
Link: <PR or issue URL>
Checked: <items from the checklist that passed>
Findings:
1. <issue> — severity: blocking / should fix / suggestion
Owner response: <what changed, with commit>
Closed: <date>
```

### Row 8 — Thermal planning and data prep `[open]`
**[pending: review on Nov 7–8. First get the link to the `thermal-ir-pipeline` branch or PR from Jacob.]**

### Row 7 — Radar evaluation `[open]`
**[pending: review on Nov 14–15.]**

### Row 9 — Thermal training and evaluation `[open]`
**[pending: review on Nov 14–15.]**

### Row 16 — Results and figures `[open]`
**[pending: review on Nov 27.]**

### Row 18 — Full report draft `[open]`
**[pending: review on Nov 29.]**

### Rows 19–21 — Final reviews `[open]`
**[pending: reviews on Dec 2, Dec 5, and Dec 6.]**
