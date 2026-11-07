# Agent tasks — Mostafa

Human-in-the-loop coordination agent. Plan rows 10, 11, and 14. Spec: `docs/agent-coordination.md`.

**Goal for the technical report:** a Methodology subsection on the agent and a Results subsection comparing the agent with the fixed rule-based baseline. Both must be reported separately from the per-model metrics and labeled as simulated.

**How this file works:** the checkpoints in this file feed the report section at the end. Each report paragraph lists the checkpoints it depends on and carries one status:

- `[pending]` — placeholder; the checkpoints are not done.
- `[draft]` — written; numbers may still change.
- `[final]` — every number traces to a committed file in `results/`.

Only `[final]` paragraphs go to Jacob for the full report (rows 15 and 18). This file is the single source for this report text; it replaces the separate drafts in `reports/`.

## Row 10 — Agent interface (Nov 9–11, review Nov 12 by Tony and Jacob)

- [x] Draft `src/agent_interface.py` and `tests/test_agent_interface.py` (ticket 4)
- [x] Commit both files on `agent-pipeline` (0642645). The 27 tests pass.
- [ ] Open a PR for Tony and Jacob's row-10 review
- [ ] Decide: does any missing or delayed expected sensor force `unverified`, even when two or more sensors agree?
- [ ] Decide: keep `confirmed` + `not_drone` as an allowed outcome? If yes, add it to `docs/agent-coordination.md`
- [ ] Set per-modality score floors from **validation** outputs (ticket 5); record the method and values in the spec
- [ ] Freeze the label mapping after the RF, thermal, and acoustic label names are confirmed by their owners

## Row 11 — Agent implementation (Nov 12–18, review Nov 19)

The committed interface can recommend the next sensors, and it stops when the status is confirmed or conflicting. That recommendation asks for every missing expected sensor, acoustic then radar then thermal, whenever any definitive label is below the score floor. The Benign/Machine sequence below is not implemented yet.

- [ ] Sequential event runner: evidence arrives over time, with one deadline per event (ticket 10)
- [ ] Evidence-request policy, for example:
  - [ ] RF uncertain in the Benign/Machine band → request radar, then acoustic
  - [ ] Radar reports a fast or large drone (D4/D5-like, weak radar class) → request acoustic or thermal
  - [ ] Stop early once the status can no longer change
- [ ] Fixed rule-based baseline: request every sensor, wait until the deadline, apply the same status rules
- [ ] Review gate enforced in code: the agent has no response or actuator tool; every path ends in an alert for review
- [ ] Audit log (JSON lines): every request, arrival, timeout, status decision, and operator disposition, with time and reason
- [ ] CLI script, e.g. `python src/run_agent_replay.py --scenarios ... --output results/agent-replay`
- [ ] Tests: policy order, early stop, timeout handling, no response tool exists, audit log completeness

## Scenarios (ticket 11, by Nov 18)

- [ ] Expand `data/synthetic-scenario-catalog.csv` from 5 to about 15–20 scenarios, each with an expected status
- [ ] Cover: full agreement, uncertain RF, missing thermal, late acoustic, low-quality evidence, full conflict, all uncertain, single supporting sensor, confirmed non-drone, RF jamming (`other`)
- [ ] Keep the `simulated` and placeholder labels on every record

## Row 14 — Evaluation (Nov 16–21, review Nov 22)

- [ ] Replace placeholders with real handoff CSVs as they arrive: radar (available now), RF, acoustic, thermal
- [ ] Record which sensors in each run are real outputs and which are placeholders
- [ ] Metrics, agent vs baseline:
  - [ ] status accuracy against each scenario's expected status
  - [ ] evidence requests per event
  - [ ] time to alert
  - [ ] review-gate and authority violations (target: 0)
- [ ] Save results under `results/agent-replay/` with a summary JSON and CSV

## Technical report content (hand to Jacob for rows 15 and 18)

- [ ] **Methodology — Agentic sensor coordination** (about 0.75 page): purpose, authority boundary, label mapping, status rules, the two policies, audit log
- [ ] **Results — Agent vs baseline** (about 0.5 page): metrics table, status confusion matrix, what the agent saved (requests, time), failure cases
- [ ] **Limitations:** simulated association, unsynchronized datasets, no fused probability, uncalibrated scores where applicable
- [ ] **Future work:** training on synchronized data, LLM operator reports, AWS model-health and IoT device-health integration, graduated autonomy for monitoring tasks only
- [ ] Figures (count toward the 8 presentation visualizations):
  - [ ] Agent architecture / decision-flow diagram
  - [ ] Agent vs baseline bar chart (requests per event, time to alert)
  - [ ] Status confusion matrix (expected vs produced)
  - [ ] One example alert JSON, rendered as a table

---

# Report section — Agentic sensor coordination

*Template sections: Methodology, Results, Limitations, Future work. Status as of 2026-11-07.*

## Methodology

### Purpose `[draft]`
*Depends on: none (scope fixed in the submitted introduction and plan).*

The project's RF, radar, thermal-infrared, and acoustic models are trained and evaluated separately. A human-in-the-loop coordination agent connects their outputs for an operator. Rather than running every model in a fixed sequence, the agent chooses which evidence to request next based on the observations already received. Every event ends in a structured alert for operator review. One motivation is the RF model's Benign/Machine trade-off: when RF evidence is ambiguous, the agent can seek radar or acoustic evidence before presenting an alert, instead of forcing a decision from RF alone.

### Authority boundary `[draft]`
*Depends on: Row 11 — review gate enforced in code.*

The agent's authority is limited to software coordination. It can request model evidence, wait for it until a deadline, and submit an alert. It has no tool to control a sensor or a drone, or to initiate any response, and this boundary is enforced in code by the absence of such tools. The alert schema rejects any alert that is not marked for operator review. A human operator remains responsible for the final disposition.

### Inputs `[draft]`
*Depends on: Row 10 — interface committed on `agent-pipeline` (0642645); PR still open.*

The agent receives one observation per sensor for each event. An observation combines a saved model output from the repository's replay contract with:

- the model version and a reference to the source output;
- whether the evidence was available, missing, delayed, or low quality;
- its arrival time after the event opened; and
- whether its score is calibrated.

Real model outputs enter through the shared model-output handoff format, which contains held-out test predictions only. Because the RF, radar, thermal, and acoustic datasets record different events and are not synchronized, the scenario author assigns outputs from different sensors to one event, and every event is marked as simulated.

### Label mapping `[draft]`
*Depends on: Row 10 — freeze label mapping; set score floors (ticket 5).*

Each modality keeps its own class labels. For status decisions only, the agent maps them to a common hypothesis:

| Modality | Drone-like | Not drone | Other |
| --- | --- | --- | --- |
| RF | Machine | Benign, Human, Wildlife | Broadband or narrowband jamming |
| Radar | drone | bird, human | — |
| Thermal/vision | drone | no drone, bird | — |
| Acoustic | drone | no drone | — |

An observation whose score falls below its modality's floor is treated as uncertain rather than as evidence for either hypothesis. The committed interface still uses the development placeholder of 0.60 for every modality. Floors must be set from validation outputs, never from test data: **[pending: floor values and method]**. Labels with no defined mapping are rejected rather than guessed. The thermal `bird` label is mapped in code because Dataset 2 uses birds as hard negatives; `docs/agent-coordination.md` still lists only drone and no drone for thermal, so the mapping is not frozen.

### Alert status `[draft]`
*Depends on: Row 10 — the two open decisions.*

The status rules are applied in priority order:

1. **Conflicting:** usable sensors disagree between drone-like and not drone.
2. **Confirmed:** at least two usable sensors support the same definitive hypothesis. The alert states which hypothesis was confirmed and still requires review.
3. **Unverified:** an expected sensor is missing, arrived after the deadline, or was low quality, and the remaining evidence does not reach confirmation.
4. **Unknown:** the usable evidence is uncertain or non-definitive, or only one sensor supports a hypothesis.

**[pending decision: whether a missing or delayed sensor forces "unverified" even when two or more sensors agree; whether "confirmed not drone" is kept.]**

Each alert lists the status, the leading hypothesis, and for each sensor its prediction, score, calibration flag, model version, evidence reference, and arrival state. It also lists the missing or delayed sensors, the reasons for the status, and an empty field for the operator's disposition. The agent does not calculate or report a fused detection probability.

### Coordination policies `[pending]`
*Depends on: Row 11 — runner, evidence-request policy, baseline, audit log.*

Both policies use the same status rules and alert format:

- **Fixed rule-based baseline:** request every sensor, wait until the event deadline, then apply the status rules.
- **Coordination agent:** request evidence one sensor at a time, choosing the next request from the current evidence. For example:
  - when RF is uncertain in the Benign/Machine region, request radar and then acoustic;
  - when radar reports a drone class with a weaker radar signature, request acoustic or thermal;
  - stop once additional evidence could no longer change the status.

Every request, arrival, timeout, status decision, and operator disposition is written to an audit log with its time and reason. **[pending: final policy rules as implemented]**

### Evaluation design `[pending]`
*Depends on: Scenarios (ticket 11); Row 14 — real handoff CSVs.*

Both policies run on **[pending: number]** simulated replay scenarios, each with an expected status. The scenarios cover agreement, full conflict, uncertain RF, missing or late evidence, low-quality evidence, all-uncertain evidence, single-sensor support, confirmed non-drone activity, and RF jamming. Placeholder records are replaced with saved held-out model outputs as they become available. Each run records which sensors used real outputs: **[pending: real sensors used]**.

## Results `[pending]`
*Depends on: Row 14 — all metric checkpoints; results saved under `results/agent-replay/`.*

| Measure | Rule-based baseline | Coordination agent |
| --- | ---: | ---: |
| Status accuracy vs expected | [pending] | [pending] |
| Evidence requests per event | [pending] | [pending] |
| Time to alert | [pending] | [pending] |
| Review-gate or authority violations (target 0) | [pending] | [pending] |

**[pending: one paragraph on what the agent saved in requests and time, where its statuses differed from the baseline, and the failure cases.]**

Figures: decision-flow diagram `[pending]` · agent vs baseline bar chart `[pending]` · status confusion matrix `[pending]` · example alert table `[pending]`

## Limitations `[draft]`
*Depends on: none; revise after Results.*

- The scenarios are simulated. Agreement between outputs from unsynchronized datasets does not show that the sensors observed the same object, and these results do not measure real-world sensor-fusion performance.
- Event association is defined by the scenario author, not inferred by the agent.
- Scores are raw model outputs unless the model's owner provides calibration evidence; each alert records which scores are calibrated.
- Status accuracy measures agreement with expected statuses written by the team, not ground truth about a physical event.

## Future work `[draft]`
*Depends on: none.*

- The evidence-combination component is kept separate so it can later be trained on synchronized multi-sensor recordings and refined with real-world operator feedback, using the audit log as a record of operator decisions.
- A later phase could add a large language model that writes concise operator reports, combining sensor evidence with model-health information from the AWS monitoring pipeline inherited from AAI-540 and device-health telemetry from sensor nodes.
- The audit log could also support graduated autonomy for low-risk monitoring actions only, such as requesting evidence or prioritizing alerts, justified by shadow-mode comparison against independent ground truth.
- Any counter-UAS response would remain the decision of an authorized person.
