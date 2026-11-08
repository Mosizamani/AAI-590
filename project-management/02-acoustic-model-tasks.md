# Acoustic model tasks — Mostafa

Binary acoustic drone detection on DADS (`Z121mm/drone-audio-detection-samples`). Plan row 6 (Nov 9–13, review Nov 14–15 by Tony and Jacob). Notebook: `notebooks/acoustic_drone_detection.ipynb` on the `mostafa` branch.

**Goal for the technical report:** Data Summary, Methodology, and Results content for the acoustic model, with the padding check, noise robustness, and limitations stated clearly.

**How this file works:** the checkpoints in this file feed the report section at the end. Each report paragraph lists the checkpoints it depends on and carries one status:

- `[pending]` — placeholder; the checkpoints are not done.
- `[draft]` — written; numbers may still change.
- `[final]` — every number traces to a committed file in `results/`.

Only `[final]` paragraphs go to Jacob for the full report (rows 15 and 18). This file is the single source for this report text; it replaces the separate drafts in `reports/`.

## Done (baselines)

- [x] Load DADS: 180,320 clips (16,729 no drone, 163,591 drone), mono 16 kHz
- [x] EDA: durations, format checks, duplicates (15 groups), mean spectra
- [x] File-level split with hash deduplication: 0 file overlap; 1 s windows, 0.5 s hop
- [x] Log-mel (128 × 98) and 84-value MFCC features, scaled on training data only
- [x] Spectrogram transformer: test accuracy 0.999, drone F1 0.999, ROC-AUC 1.000
- [x] Spectrogram CNN: test accuracy 0.993, drone F1 0.993, ROC-AUC 0.999
- [x] Random forest (MFCC): test accuracy 0.991, drone F1 0.991, ROC-AUC 1.000
- [x] Thresholds chosen on validation at 1% and 5% false alarm; test scored once

## 1. Padding-shortcut check (ticket 8a) — done

About 162k windows are zero-padded to 1 s. Because most drone clips last 0.5 s and most no-drone clips are much longer, these are expected to be mostly drone windows; report the exact count per class. A model may learn "half silence = drone."

- [x] One-feature baseline: fraction of trailing zero samples per window → test ROC-AUC 0.924
- [x] Re-score all three saved models on **unpadded** drone windows only vs. no-drone windows
- [x] The shortcut is not the models' only cue. The three saved models remain valid on this test set, so the windowing was not changed and the models were not retrained.

## 2. Optimized model (notebook sections 14–15)

- [ ] Tune with Optuna on validation only: learning rate, batch size, dropout, weight decay, augmentation strength, window length
- [ ] Keep a results table of every run; change one thing at a time
- [ ] Temperature scaling on validation; report expected calibration error before and after (needed for the agent's `score_calibrated`)
- [ ] Re-select the 1% and 5% false-alarm thresholds on validation
- [ ] Optional: distillation, quantization, or ONNX export; measure latency and model size
- [ ] Score the optimized model once on the same test windows; compare with the three saved models
- [ ] Bootstrap 95% confidence intervals **over files**, not windows

## 3. Noise robustness (plan row 6)

- [ ] Choose noise sources (e.g. ESC-50, UrbanSound8K subsets) and record their licenses
- [ ] Mix noise only into test-time copies of held-out windows at fixed SNRs (e.g. 20, 10, 5, 0, −5 dB)
- [ ] Keep the clean-validation threshold fixed; report recall, false-alarm rate, and F1 per SNR
- [ ] Report the robustness results separately from the clean results

## 4. Repository artifacts (ticket 8)

- [ ] Commit the code (notebook and/or `src/acoustic_*.py`), environment/dependency file, and split manifest (`windows.csv` or file-level list)
- [ ] Save metrics JSON/CSV and figures under `results/acoustic/`; move them off Google Drive only
- [ ] Remove the hard-coded Drive folder IDs and the Colab-only cells, or isolate them
- [ ] Clear progress-bar outputs to shrink the notebook (currently 5.7 MB)
- [ ] Export a held-out handoff CSV (`sensor_type=acoustic`, `split=test`), add `calibrated_score` if calibrated, and run `python src/validate_model_outputs.py`
- [ ] Update `data/dataset-manifest.csv` (license, checksum) and `docs/acoustic-model-evidence.md` (CNN, RF, padding check, robustness)

## Technical report content (hand to Jacob for rows 15 and 18)

- [ ] **Data Summary:** DADS source and license, class and duration imbalance, padding and silence handling, no source/site label
- [ ] **Methodology:** windowing, features, three models plus optimized model, training setup, validation-only threshold selection, calibration
- [ ] **Results:** comparison table (four models plus majority baseline), detection at 1% and 5% false alarm, confidence intervals, padding check, robustness by SNR
- [ ] **Limitations:** no source/site hold-out, possible microphone-on-drone recordings, unknown distances, no field recordings
- [ ] Figures:
  - [ ] class/duration distribution
  - [ ] mean spectrograms (drone vs no drone)
  - [ ] ROC overlay of all models
  - [ ] recall vs SNR

---

# Report section — Acoustic drone detection

*Template sections: Data Summary, Methodology, Results, Limitations. Baseline numbers are from `notebooks/acoustic_drone_detection.ipynb` (branch `mostafa`). Status as of 2026-11-07. No acoustic checkpoint changed on this date.*

## Data summary `[draft]`
*Depends on: Done (baselines); ticket 2 (license recorded in the manifest).*

The acoustic model uses the Drone Audio Detection Samples (DADS) dataset hosted on Hugging Face (Z121mm, n.d.). DADS contains 180,320 WAV clips with binary labels: 16,729 no-drone clips (33.8 hours) and 163,591 drone clips (27.1 hours).

The classes differ strongly in clip length. Most drone clips last 0.5 s (mean 0.6 s), while no-drone clips have a median of 4.0 s and run up to 16 s.

Every file was readable and already mono, 16-bit, and 16 kHz, so no resampling was applied. The audit flagged 1,056 clips quieter than −60 dBFS and 1,103 clips with full-scale clipping. It also found 15 groups of byte-identical audio, none with conflicting labels.

The dataset does not record which original corpus or recording site each clip came from; files are named only `drone-N.wav` and `no-drone-N.wav`. A whole source therefore cannot be held out, and the evaluation cannot measure generalization to a new site or microphone.

## Methodology

### Splitting and windowing `[draft]`
*Depends on: Done (baselines). The padding check did not change this split.*

Each file, with byte-identical duplicates merged, is one unit, and each unit is assigned entirely to training, validation, or test. No file appears in more than one split.

- Clips shorter than 0.5 s (176 files) were dropped.
- Clips of 0.5–1 s become one window zero-padded to 1 s.
- Longer clips are cut into 1 s windows with a 0.5 s hop.
- Windows quieter than −60 dBFS (30,204) were removed, which removed 804 files entirely.

| Split | No-drone windows (files) | Drone windows (files) |
| --- | ---: | ---: |
| Train | 136,639 (11,057) | 137,181 (114,470) |
| Validation | 29,468 (2,369) | 27,536 (24,529) |
| Test | 28,334 (2,370) | 28,681 (24,530) |

Because long no-drone clips yield many windows, the window counts are nearly balanced without resampling.

### Features `[draft]`
*Depends on: Done (baselines).*

Each window becomes a 128-bin log-mel spectrogram (25 ms window, 10 ms hop, 20–8,000 Hz), giving 128 × 98 values. For the random forest, each window is summarized as an 84-value vector: the mean and standard deviation of 13 MFCCs plus energy, and of their first and second differences. Both feature types are standardized using training windows only.

### Models and training `[draft]`
*Depends on: Done (baselines); section 2 adds the optimized model.*

Three models are compared against a majority-class baseline:

- **Spectrogram transformer:** trained from scratch; 16 × 7 patches, 112 patch tokens plus a class token, 4 layers, 4 heads, width 128; 559,490 parameters. It follows the patch-based design of the Audio Spectrogram Transformer (Gong et al., 2021) at a much smaller scale.
- **Spectrogram CNN:** three layers with 32, 64, and 128 filters; 92,930 parameters; same log-mel input.
- **Random forest:** trained on the MFCC vectors.

The two neural networks are trained with:

- weighted cross-entropy (class weights near 1.0);
- AdamW (learning rate 3 × 10⁻⁴, weight decay 0.01);
- gradient clipping, a one-epoch warmup, and cosine decay;
- a batch size of 64;
- training-only augmentation: gain, time shift, drone-on-background mixing, and SpecAugment (Park et al., 2019);
- up to 20 epochs, with early stopping on validation ROC-AUC.

The forest's tree count and depth are chosen from a small grid by validation ROC-AUC.

**[pending: optimized model, including the Optuna search space, selected settings, and temperature-scaling calibration with expected calibration error before and after.]**

### Operating thresholds `[draft]`
*Depends on: Done (baselines).*

False alarms are the costly error for operators. Each model's thresholds are therefore chosen on validation data to give 1% and 5% false-alarm rates, then applied once to the test set. Each model scored the test split a single time.

## Results

### Baseline comparison `[draft]`
*Depends on: Done (baselines); becomes `[final]` after the artifacts are committed under `results/acoustic/` (section 4).*

All models were evaluated on 57,015 held-out test windows (28,681 drone, 28,334 no drone).

| Model | Accuracy | Drone precision | Drone recall | Drone F1 | ROC-AUC | Detection at 1% FA | Detection at 5% FA |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Majority class (always drone) | 0.503 | 0.503 | 1.000 | 0.669 | 0.500 | 0.010 | 0.050 |
| Spectrogram transformer | **0.999** | 0.998 | **0.999** | **0.999** | 1.000 | **1.000** | 1.000 |
| Spectrogram CNN | 0.993 | 0.995 | 0.990 | 0.993 | 0.999 | 0.993 | 0.998 |
| Random forest (MFCC) | 0.991 | 0.998 | 0.984 | 0.991 | 1.000 | 0.993 | 0.998 |

At the validation-selected 1% false-alarm threshold, the realized test false-alarm rates were 0.8% (transformer), 0.8% (CNN), and 0.9% (random forest), so the validation thresholds transferred well. The CNN and the forest use different features (log-mel image vs MFCC summary), so the gap between them partly reflects the feature as well as the model.

### Padding check `[draft]`
*Depends on: section 1 — done. The saved models remain valid; no retrain.*

On the test set, 24,341 of 28,681 drone windows (84.9%) are zero-padded, against 47 of 28,334 no-drone windows (0.2%). The fraction of trailing zeros has a test ROC-AUC of 0.924 and, at a 1% false-alarm limit, detects 84.9% of test drone windows. It misses all 4,340 unpadded drone windows.

Scored on those unpadded drone windows plus every no-drone window, the saved models still detect drones: transformer recall 0.998 (ROC-AUC 1.000), CNN recall 0.953 (ROC-AUC 0.996), and random forest recall 0.955 (ROC-AUC 0.997), with false-alarm rates of 0.8–0.9%. The published scores stand, and the models were not retrained.

### Optimized model `[pending]`
*Depends on: section 2 — all checkpoints.*

**[pending: optimized-model row added to the table; file-level bootstrap 95% confidence intervals; latency and model size.]**

### Noise robustness `[pending]`
*Depends on: section 3 — all checkpoints.*

**[pending: noise sources and licenses; recall, false-alarm rate, and F1 at 20, 10, 5, 0, and −5 dB SNR with the clean-validation threshold held fixed.]**

Figures: class/duration distribution `[draft]` · mean spectrograms `[draft]` · ROC overlay `[draft]` · recall vs SNR `[pending]`

## Limitations `[draft]`
*Depends on: padding check is done; revise again after the robustness results.*

- The results measure performance on DADS only. Without source or site labels, they do not establish performance on new microphones, sites, distances, or drone types.
- Windows from the same file are correlated, so window-level metrics overstate the effective sample size; confidence intervals should be computed over files.
- Scores are concentrated near 0 and 1 and are not calibrated probabilities unless the calibration checkpoint is completed.
- The acoustic recordings are not synchronized with the RF, radar, or thermal data. Acoustic outputs enter the coordination agent only through clearly labeled simulated replay scenarios and do not demonstrate real-world sensor fusion.

## References

Gong, Y., Chung, Y.-A., & Glass, J. (2021). AST: Audio spectrogram transformer. *Proceedings of Interspeech 2021*, 571–575. https://doi.org/10.21437/Interspeech.2021-698

Park, D. S., Chan, W., Zhang, Y., Chiu, C.-C., Zoph, B., Cubuk, E. D., & Le, Q. V. (2019). SpecAugment: A simple data augmentation method for automatic speech recognition. *Proceedings of Interspeech 2019*, 2613–2617. https://doi.org/10.21437/Interspeech.2019-2680

Z121mm. (n.d.). *Drone-audio-detection-samples* [Data set]. Hugging Face. https://huggingface.co/datasets/Z121mm/drone-audio-detection-samples
