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

## 1b. Noise × padding split (ticket 8b) — done

At 20 dB (noise about 1% of the window's power), drone recall falls from about 1.0 to 0.308 / 0.445 / 0.148 (transformer / CNN / forest). The forest's 0.148 is close to the unpadded share of test drone windows (15.1%), and recall is not monotonic in SNR. Noise is added across the whole second, so it fills the zero-padded tail. Hypothesis: on padded windows the models rely on the digital-silence tail, and the drop is mostly that artifact rather than noise level.

- [x] Re-use the section 13.2 noisy mixes: same ESC-50 draw, same SNRs, same seed. No new mixing. Overall recall matches `noise_robustness.csv` at every SNR
- [x] Keep each model's clean-validation 1% false-alarm threshold fixed; do not refit
- [x] Split drone windows by the existing padded flag (test: 24,341 padded, 4,340 unpadded)
- [x] Per model and SNR (clean, 20, 10, 5, 0, −5 dB): recall on padded drone windows, recall on unpadded drone windows. The false-alarm rate stays on all no-drone windows (only 47 are padded)
- [x] Saved `results/acoustic/noise_robustness_by_padding.csv` and `results/acoustic/recall_vs_snr_by_padding.png`
- [x] Outcome: padding artifact confirmed. At 20 dB, unpadded recall is 0.991 / 0.936 / 0.912 and padded recall is 0.186 / 0.357 / 0.012 (transformer / CNN / forest). Every gap is above 0.20
- [x] Unpadded-window recall is the deployment-relevant robustness figure. Real microphones never produce a digital-zero tail

Short clips are kept and rewindowed before section 2. Dropping them would remove about 85% of the drone windows. The three saved models stay as the "before" models.

## 2. Optimized CNN (ticket 8c, notebook sections 14–15)

Fix the two problems ticket 8b found. The three saved baselines stay as they are and become the "before" models; they are rescored, not retrained.

The spectrogram CNN (92,930 parameters) is the model that is optimized, because it is the cheapest to retrain many times and the most practical for real-time use on a sensor. This choice is size and latency, not the noise test. On unpadded noisy windows the transformer scores higher (0.991 vs 0.936 at 20 dB, 0.412 vs 0.306 at −5 dB). The transformer is a candidate for the same treatment later.

### 2.1 Write-up corrections from 8b

- [x] Noise paragraph: the two findings replace "not robust to this ESC-50 noise" (padding artifact at moderate noise, real loss on full-length audio near 0 to −5 dB). "The CNN keeps the highest recall" is limited to all windows. The transformer leads on unpadded windows
- [x] Padding paragraph: clean scores stand, and the noise test showed that padding matters, so short clips are rewindowed before optimization
- [x] Full unpadded table (clean, 20, 10, 5, 0, −5 dB) is the headline robustness figure

### 2.2 Rewindow the short clips

- [x] Replace zero-padding: each short window keeps its samples, and the tail points at a training no-drone clip scaled to 20 dB below the clip when the waveform is built. Short clips were not dropped. `dataset2-processed/windows_v2.csv` has 225,361 `none` and 162,478 `background`
- [x] The file-level split is unchanged. `fill` is `none` or `background`
- [x] No accepted background crop has 160 or more consecutive digital zeros (440 crops were redrawn). The old trailing-zero score is 0 on all 57,015 test windows, so its ROC-AUC is 0.500
- [ ] Rebuild log-mel features and the training-only scale from the new windows

### 2.3 Noise augmentation (train only)

- [ ] Keep augmentation noise separate from the test noise: augment with ESC-50 folds 1–4 and rebuild the robustness test mix from fold 5 only (or use a different noise source for training)
- [ ] Mix noise into training windows at random SNRs (for example −5 to 20 dB) with a set probability. Validation and test stay clean except for the robustness mix
- [ ] Build a noisy validation copy (fold 1–4 noise, fixed seed) for model selection

### 2.4 Ablation, then tuning

- [ ] Run A: CNN on rewindowed data, original section 8 settings, no augmentation (one run)
- [ ] Run B: run A plus noise augmentation (one run)
- [ ] Run C: Optuna on run B, validation only: learning rate, batch size, dropout, weight decay, augmentation probability and SNR range. Objective: clean validation ROC-AUC plus noisy validation recall at the 1% false-alarm threshold
- [ ] Keep a results table of every run
- [ ] Temperature scaling on clean validation; report expected calibration error before and after (needed for the agent's `score_calibrated`)
- [ ] Re-select the 1% and 5% false-alarm thresholds on clean validation
- [ ] Optional: quantization or ONNX export; measure latency and model size

### 2.5 Evaluation (test once)

- [ ] Score runs A, B, C and the three saved baselines on the same rewindowed test windows. The baselines are rescored only; a drop shows how much they relied on the zeros
- [ ] Report the clean comparison, then the fold-5 noise test by SNR, split by original padded / unpadded windows
- [ ] Report the step effects: baseline → A (rewindowing) → B (augmentation) → C (tuning)
- [ ] Bootstrap 95% confidence intervals over files, not windows
- [ ] State in the report that the baselines were trained on zero-padded data and the optimized CNN was not

## 3. Noise robustness (plan row 6) — done

- [x] Choose noise sources (e.g. ESC-50, UrbanSound8K subsets) and record their licenses. ESC-50 subset, CC BY-NC 3.0, 400 clips from 10 categories
- [x] Mix noise only into test-time copies of held-out windows at fixed SNRs (20, 10, 5, 0, −5 dB)
- [x] Keep the clean-validation threshold fixed; report recall, false-alarm rate, and F1 per SNR
- [x] Report the robustness results separately from the clean results

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

Scored on those unpadded drone windows plus every no-drone window, the saved models still detect drones: transformer recall 0.998 (ROC-AUC 1.000), CNN recall 0.953 (ROC-AUC 0.996), and random forest recall 0.955 (ROC-AUC 0.997), with false-alarm rates of 0.8–0.9%. The published clean scores stand, and the three saved models were not retrained. The noise test showed that padding does matter once the zero tail is filled, so the short clips are rewindowed before optimization.

### Optimized model `[pending]`
*Depends on: section 2 — runs A, B, and C. The CNN is optimized for size and latency. The transformer remains a candidate for the same treatment later.*

**[pending: runs A, B, and C on rewindowed windows; clean comparison; fold-5 noise test split by the original padded flag; file-level bootstrap 95% confidence intervals; latency and model size. The baselines were trained on zero-padded data and are rescored only.]**

### Noise robustness `[draft]`
*Depends on: section 3 — done. Reported separately from the clean comparison.*

Noise was mixed only into copies of the 57,015 test windows. The source is 400 ESC-50 clips (Piczak, 2015) from rain, sea waves, wind, thunderstorm, engine, train, helicopter, airplane, crackling fire, and chainsaw, under CC BY-NC 3.0. Each copy was scored at 20, 10, 5, 0, and −5 dB. The threshold is the clean-validation 1% false-alarm point and was not refit.

At 20 dB, drone recall on all windows falls from the clean operating point to 0.308 (transformer), 0.445 (CNN), and 0.148 (random forest). At −5 dB those all-window recalls are 0.190, 0.267, and 0.075. On all windows, the CNN keeps the highest recall at both ends. False-alarm rates rise with the noise: 0.105 for the transformer at −5 dB, at most 0.050 for the CNN, and 0.087 for the forest at 0 dB. The clean scores in the comparison table still stand.

Two findings sit under that all-window table. At moderate noise the drop is a padding artifact: the mix fills the digital-zero tail of the short clips. On full-length audio the loss is real near 0 dB to −5 dB, where the noise is about as loud as the window. The transformer leads on unpadded windows (0.991 vs 0.936 at 20 dB, 0.412 vs 0.306 at −5 dB). The CNN is still the model that is optimized, because it has 92,930 parameters and is the practical real-time model, not because it wins this noise test.

Unpadded recall is the headline robustness figure. A real microphone does not leave a stretch of digital zeros. The false-alarm rate stays on all no-drone windows.

| Model | Clean | 20 dB | 10 dB | 5 dB | 0 dB | −5 dB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Spectrogram transformer | 0.998 | 0.991 | 0.943 | 0.848 | 0.659 | 0.412 |
| Spectrogram CNN | 0.953 | 0.936 | 0.820 | 0.662 | 0.475 | 0.306 |
| Random forest | 0.955 | 0.912 | 0.770 | 0.631 | 0.450 | 0.268 |

The decibel number is the signal-to-noise ratio: the power of the original window compared with the power of the noise mixed into it. At 0 dB the two powers are equal. A positive value means the window is louder than the noise, and a negative value means the noise is louder. At 20 dB the window has about 100 times the noise power. At 10 dB it has about 10 times, and at 5 dB about 3 times. At −5 dB the noise has about 3.2 times the power of the window, so the clip is mostly noise.

Figures: class/duration distribution `[draft]` · mean spectrograms `[draft]` · ROC overlay `[draft]` · recall vs SNR `[draft]` · recall vs SNR by padding `[draft]`

## Limitations `[draft]`
*Depends on: padding check, noise robustness, and the noise × padding split are done.*

- The results measure performance on DADS only. Without source or site labels, they do not establish performance on new microphones, sites, distances, or drone types. Added ESC-50 noise is a separate test-time mix, not a field recording, and some of those clips (helicopter, airplane, engine) can themselves resemble a drone.
- The clean comparison still stands. The 20 dB recall below 0.45 is mostly the 24,341 padded drone windows: noise fills their digital-zero tail. On the 4,340 unpadded drone windows, 20 dB recall stays at 0.991 / 0.936 / 0.912. That unpadded figure is the one that applies to a real microphone. Windowing still has to be fixed, and the models retrained, before the CNN fine-tune is run.
- Windows from the same file are correlated, so window-level metrics overstate the effective sample size; confidence intervals should be computed over files.
- Scores are concentrated near 0 and 1 and are not calibrated probabilities unless the calibration checkpoint is completed.
- The acoustic recordings are not synchronized with the RF, radar, or thermal data. Acoustic outputs enter the coordination agent only through clearly labeled simulated replay scenarios and do not demonstrate real-world sensor fusion.

## References

Gong, Y., Chung, Y.-A., & Glass, J. (2021). AST: Audio spectrogram transformer. *Proceedings of Interspeech 2021*, 571–575. https://doi.org/10.21437/Interspeech.2021-698

Park, D. S., Chan, W., Zhang, Y., Chiu, C.-C., Zoph, B., Cubuk, E. D., & Le, Q. V. (2019). SpecAugment: A simple data augmentation method for automatic speech recognition. *Proceedings of Interspeech 2019*, 2613–2617. https://doi.org/10.21437/Interspeech.2019-2680

Piczak, K. J. (2015). ESC: Dataset for environmental sound classification. *Proceedings of the 23rd ACM International Conference on Multimedia*, 1015–1018. https://doi.org/10.1145/2733373.2806390

Z121mm. (n.d.). *Drone-audio-detection-samples* [Data set]. Hugging Face. https://huggingface.co/datasets/Z121mm/drone-audio-detection-samples
