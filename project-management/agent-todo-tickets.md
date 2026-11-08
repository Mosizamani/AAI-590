# Agent and documentation to-do tickets

Tickets that bring the repository in line with the submitted introduction and the Assignment 2.1.2 management plan, then build the human-in-the-loop coordination agent. Plan rows refer to `docs/project-management-plan.md`.

Status as of 2026-11-07.

| # | Ticket | Owner | Due | Plan row | Status |
|---|---|---|---|---|---|
| 1 | Open a docs PR with the seven updated .md files (README, plan, decisions, roadmap, acoustic evidence, handoff, agent-coordination); Tony and Jacob review | Mostafa | Nov 6 | 2 | Committed on `mostafa` (4ee2766); PR to `main` pending |
| 2 | Update `data/dataset-manifest.csv`: DADS license (MIT) with verification date and checksum; mark Kaggle as secondary | Mostafa | Nov 8 | 3 | To do |
| 3 | Add GitHub Project cards for plan rows 10, 11, and 14 with owner, reviewers, and dates | Tony | Nov 8 | 2 | To do |
| 3a | Move the radar EDA notebook to `main`: replace the hard-coded local data path with a config or environment variable, and rename it to the repo convention (e.g. `02_radar_eda.ipynb`) | Mostafa | Nov 8 | 3 | To do |
| 4 | Agent interface: shared observation (arrival time, `score_calibrated`), alert schema, label mapping, status enum with mapping from existing status types, and tests | Mostafa | Nov 11 | 10 | Committed on `agent-pipeline` (0642645); 27 tests pass; PR and row-10 review pending |
| 5 | Set per-modality uncertainty floors from validation outputs (especially RF Benign/Machine); never use the test split | Mostafa | Nov 12 | 10 | To do |
| 6 | Radar calibration: temperature scaling on the validation split, report expected calibration error, add `calibrated_score` to `radar-test-outputs.csv` | Tony | Nov 13 | 7 | To do |
| 7 | RF v4/v5: export a validated held-out handoff CSV (calibrated if possible) | Mostafa | Nov 13 | 5 | Notebook on `mostafa` (672bdfc) writes `rf-test-outputs.csv` with validation temperature scaling. The CSV is not in the repo yet. The exported checkpoint is diagnostic (weight 1.25, seed 42); the evaluation keeps v4 |
| 8 | Acoustic: commit code, environment, split manifest, metrics, noise-robustness results, and a validated handoff CSV | Mostafa | Nov 13 | 6 | In progress (baselines on `mostafa`; optimized model pending) |
| 8a | Acoustic padding-shortcut check before tuning or export: one-feature trailing-zero baseline, evaluation on unpadded drone windows only, and a fix if confirmed (same padding/cropping for both classes, 0.5 s windows, or noise padding) | Mostafa | Nov 13 | 6, 12 | Done. Trailing-zero test ROC-AUC 0.924. On unpadded drone windows, recall is 0.998 / 0.953 / 0.955 (transformer / CNN / forest). Models remain valid; no retrain |
| 9 | Thermal: link Jacob's repo/PR, commit artifacts, export a validated handoff CSV | Jacob | Nov 13 | 9 | To do |
| 10 | Agent implementation: sequential event runner with deadlines, evidence-request policy, fixed rule-based baseline, code-enforced review gate, audit log | Mostafa | Nov 18 | 11 | To do |
| 11 | Expand the scenario catalog from 5 to about 15–20 scenarios, each with an expected status (late evidence, low quality, uncertain RF, all uncertain, etc.) | Mostafa | Nov 18 | 11, 14 | To do |
| 12 | Evaluation: agent vs baseline on status accuracy, requests per event, time to alert, and violations; replace placeholders with real handoff CSVs as they land | Mostafa | Nov 21 | 14 | To do |
| 13 | Final-report wording check: Lattice wording and the "calibrated" claim must match what tickets 6–9 delivered; agent results in a section separate from per-model metrics | Jacob | Nov 28 | 18 | To do |

## Open decisions for the row 10 review (Nov 12)

1. **Missing sensor when others agree.** Current rule: three agreeing sensors with thermal missing gives `confirmed`, and the gap is listed in the alert. The submitted introduction says "unverified when expected evidence is missing or delayed." Decide whether any expected-sensor gap forces `unverified`.
2. **Confirmed non-drone.** Current rule: two or more sensors agreeing on `not_drone` gives `confirmed` with leading hypothesis `not_drone`, still routed to review. Record this in `docs/agent-coordination.md` if kept.
3. **Decision-log date.** Replace "2026-11 (before plan submission)" in `docs/project-decisions.md` with the exact agent-approval date.
