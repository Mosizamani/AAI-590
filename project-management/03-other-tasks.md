# Other tasks — Mostafa

The owner tasks in the management plan that are not agent or acoustic work, plus housekeeping tickets and whole-team rows.

**How this file works:** the checkpoints in this file feed the report section at the end. Each report paragraph lists the checkpoints it depends on and carries one status:

- `[pending]` — placeholder; the checkpoints are not done.
- `[draft]` — written; numbers may still change.
- `[final]` — every number traces to a committed file in `results/`.

Only `[final]` paragraphs go to Jacob for the full report (rows 15 and 18). This file is the single source for this report text; it replaces the separate drafts in `reports/`.

## Row 3 — Radar data audit, cleaning, EDA, and data summary (Nov 2–6, review Nov 7–8 by Jacob)

- [x] Radar EDA notebook (`KTH 77 GHz FMCW radar-EDA.ipynb` on `mostafa`)
- [ ] Ticket 3a: replace the hard-coded `/Users/...` data path with a config or environment variable
- [ ] Ticket 3a: rename to the repo convention, e.g. `notebooks/02_radar_eda.ipynb`
- [ ] Open a PR to `main` for Jacob's review
- [ ] Write the radar **Data Summary** text for the report:
  - [ ] class balance (drones are 77% of segments; pigeon and raven are tiny)
  - [ ] the official split leaks: all 130 of 130 recordings span multiple splits
  - [ ] Doppler spread by class (D4 and D5 look like birds or humans)
  - [ ] range-cell quantization bands

## Row 5 — RF v4 versus v5 evaluation and error analysis (Nov 9–13, review Nov 14–15 by Tony and Jacob)

Notebook: `notebooks/RF_Intrusion_07_CNN_v5_Training_and_Evaluation.ipynb` on `mostafa` (672bdfc). The written evaluation is section 12 of that notebook.

- [x] Re-run or collect the v4 and v5 held-out results from the AAI-540 repo
- [x] Per-class precision, recall, F1, and confusion matrices; focus on the Benign/Machine trade-off
- [x] Error analysis: which Benign signals become Machine, and at what thresholds
- [ ] Ticket 7: the notebook writes `rf-test-outputs.csv` with temperature scaling, and the six labels match the agent map. The CSV is not in the repo yet. It comes from the diagnostic checkpoint (weight 1.25, seed 42), while the evaluation keeps v4
- [x] Report content: RF evaluation paragraph drafted below. The figure is still the notebook confusion matrices, not a file in `results/`

## Row 12 — Data leakage, split integrity, and metric checks (Nov 16–18, review Nov 19 by Tony and Jacob)

- [ ] Radar: confirm the measurement-disjoint split; state that session/subject overlap is unverified
- [ ] RF: confirm the v4/v5 split policy. Section 6 already scored every loss weight on the test split, so that comparison is not blind. Section 7 then left the test split unscored and kept v4
- [ ] Acoustic: file-level split, hash dedup, and the padding-shortcut result (ticket 8a)
- [ ] Thermal: confirm Jacob's recording-disjoint split and frame-sampling policy
- [ ] Check every metric: macro F1 and per-class recall reported alongside accuracy; thresholds chosen on validation only
- [ ] Write a one-page checklist result in `docs/` and link it from the report's limitations

## Row 17 — Repository setup and reproducibility check (Nov 23–26, review Nov 27 by Jacob)

- [ ] Fresh clone → follow `SETUP.md` → run all tests and scripts on a clean environment
- [ ] Pin dependencies (`requirements-*.txt`) for radar, acoustic, and agent
- [ ] Every result in the report traces to a committed file in `results/`
- [ ] README: final status, structure table, and how to run the agent replay
- [ ] Remove local paths, Drive IDs, and large outputs from notebooks

## Housekeeping tickets

- [ ] Ticket 1: open the PR from `mostafa` to `main` (the docs are already committed there)
- [ ] Ticket 2: `data/dataset-manifest.csv`, DADS license (MIT), verification date, checksum; mark Kaggle as secondary
- [ ] Replace the decision-log placeholder date in `docs/project-decisions.md`

## Whole-team rows

- [ ] Row 19 — Final report and source/claim review (Nov 30–Dec 1, review Dec 2)
- [ ] Row 20 — Slides, rehearsal, and recording (Dec 2–4, review Dec 5): at least 2 minutes of speaking each; own the acoustic and agent slides
- [ ] Row 21 — Playback check and final repository review (Dec 6; Tony submits)
- [ ] Peer evaluation submission

---

# Report sections — Radar data summary, RF evaluation, data integrity, reproducibility

*Status as of 2026-11-07.*

## Radar data summary `[draft]`
*Depends on: Row 3 — radar EDA notebook (done); ticket 3a and the PR to `main`. Numbers are from `KTH 77 GHz FMCW radar-EDA.ipynb`.*

The radar data are the KTH 77 GHz FMCW measurements of drones, birds, humans, and a corner reflector, recorded with a mechanically scanning SAAB SIRS 1600 radar (Karlsson, 2022). The file matches its documentation: 130 measurements, 75,868 segments, 67 scan-edge segments, and no non-finite samples. Each segment is a 15 ms crop of 5 range cells by 256 azimuth sweeps.

**Class balance.** There are 15 source labels: six drones (D1–D6), walking and running humans, six bird labels, and the corner reflector. Drones contribute 77% of all segments; D5 (Tarot 680 Pro) has the most at 12,093. Pigeon (32 segments) and raven (19 segments) are single recordings, too small to treat as stable classes.

**The official split leaks.** All 130 recordings contribute segments to more than one of the published train, validation, and test splits, and 129 have overlapping time spans across splits. Neighboring segments about 100 ms apart are assigned to different splits 38% of the time. A model could therefore see the same flight during training and testing. This project uses a measurement-disjoint split instead (see the radar methodology).

**Geometry.** Reported range runs from 5.7 m to 194.3 m. The median gap between segment centers is 100 ms, with 97.6% of gaps between 80 and 120 ms. The peak range cell is the center cell in 99.0% of segments, so the five range cells are a tight crop around the detection and separate classes only weakly.

**Doppler spread separates targets.** The share of center-cell Doppler power outside ±1 m/s (median) is:

| Target | Share outside ±1 m/s |
| --- | ---: |
| Corner reflector | 0.00 |
| DJI drones (D1–D3) | 0.04–0.07 |
| Syma (D6) | 0.13 |
| Custom FPV (D4) | 0.75 |
| Tarot (D5) | 0.87 |
| Walking humans | 0.96 |
| Birds | 0.98 |
| Running humans | 1.00 |

D4 and D5 therefore resemble birds and humans on this measure, which is consistent with the RadarCNN's drone–human confusions.

**Range rate vs Doppler.** The Doppler centroid tracks range rate only partly. Pearson r is 0.50 for drones, 0.56 for humans, 0.16 for birds, and 0.30 for the reflector. The points fall in vertical bands because range is quantized: one 0.94 m cell over a 100 ms gap is 9.4 m/s.

## RF v4 versus v5 evaluation `[draft]`
*Depends on: Row 5. Numbers are from the notebook outputs on `mostafa` (672bdfc), not yet from files in `results/`. The handoff CSV is not committed.*

The data are the unified multi-class RF intrusion spectrograms, 128×128 images, six balanced classes, on the assignment split of 9,600 train, 2,400 validation, 2,400 test, and 9,600 production. v4 is one recorded test run. The saved v5 is the pooling head with Machine loss weight 1.0, also one test run.

| Version | Accuracy | Macro F1 | Machine recall | Benign→Machine | Machine→Benign |
| --- | ---: | ---: | ---: | ---: | ---: |
| v4 | 0.8875 | 0.8865 | 0.6450 | 59 | 118 |
| v5, weight 1.0 | 0.8983 | 0.8965 | 0.6375 | 38 | 122 |

v5 raises Benign recall from 0.8525 to 0.9050 and Machine precision from 0.7500 to 0.7994. Machine recall falls from 0.6450 to 0.6375 (258 correct versus 255). The keep rule requires a higher Machine recall, so this evaluation keeps the v4 model.

Section 7 compares each v5 weight as the mean of three seeds with v4's single recorded macro F1 of 0.8865. No weight reaches that floor. The highest mean is 0.8831 at weight 1.25. Mean validation Machine recall rises from 0.6167 at weight 1.00 to 0.7867 at weight 1.45, at the cost of Benign recall, which falls from 0.8800 to 0.5800. Section 6 scored weights 1.25 through 1.50 on the test split before this rule, so that table is exploratory.

The later tables use the seed-42 checkpoint of weight 1.25 and are diagnostic. On its validation predictions, argmax gives 105 Benign→Machine and 95 Machine→Benign. Temperature 1.5158 is fit on validation only. Validation negative log-likelihood falls from 0.2616 to 0.2453, and the argmax stays the same. Test expected calibration error in 15 equal-width bins falls from 0.0290 to 0.0213. The chosen Machine threshold on the calibrated validation probabilities is 0.50 (Benign→Machine 102, Machine→Benign 98).

`rf-test-outputs.csv` would contain the 2,400 test rows from this diagnostic checkpoint, with `calibrated_score` filled. `rf-production-outputs.csv` is not the agent handoff. The agent score floor is still the 0.60 placeholder, which is separate from this 0.50 label threshold.

Figure: notebook confusion matrices `[draft]`. A Benign/Machine trade-off figure saved under `results/` is still `[pending]`.

## Data leakage, split integrity, and metric checks `[pending]`
*Depends on: Row 12 — all checkpoints.*

| Modality | Split unit | Leakage check result | Thresholds on validation only | Macro F1 / per-class recall reported |
| --- | --- | --- | --- | --- |
| Radar | measurement | [pending] — session/subject overlap unverified | [pending] | [pending] |
| RF | [pending] | [pending] | [pending] | [pending] |
| Acoustic | file (hash-deduplicated) | [pending] — padding-shortcut result | yes (1% / 5% FA) | [pending] |
| Thermal | recording | [pending] | [pending] | [pending] |

**[pending: one paragraph summarizing the checks and linking to the checklist in `docs/`.]**

## Reproducibility `[pending]`
*Depends on: Row 17 — all checkpoints.*

**[pending: tested environment and pinned dependencies; the fresh-clone run result; the location of every result file used in the report.]**

## References

Karlsson, A. (2022). *Radar measurements on drones, birds and humans with a 77GHz FMCW sensor* (Version 2.0) [Data set]. Zenodo. https://doi.org/10.5281/zenodo.5896641
