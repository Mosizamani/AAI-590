## Report: Evaluation (section 15)

All six models were scored once on the same rewindowed test set: 57,015 windows (28,681 drone, 28,334 no drone). In this test set, the tail of each short clip is filled with background audio instead of zeros. The three saved models (CNN, Transformer, Random forest) were trained on zero-padded windows and were only rescored. CNN-Rewindowed, CNN-NoiseAug, and CNN-Tuned were trained on the rewindowed windows. Each model uses its own 1% false-alarm threshold, chosen on clean validation and not refit on noisy copies. The noise test mixes ESC-50 fold 5 into copies of the test windows at 20, 10, 5, 0, and −5 dB. Training noise came only from folds 1–4.

**The saved models depended on the zero tail.** Once the zeros are replaced, clean recall at the 1% threshold falls to 0.342 for the CNN, 0.293 for the Transformer, and 0.145 for the Random forest. Clean ROC-AUC falls to 0.914, 0.823, and 0.545. On full-length windows, which never had padding, their recall is still 0.954 to 0.998. Their earlier near-perfect scores came mostly from the trailing silence, not from the drone sound.

**Full-length recall is the headline.** Full-length windows (4,340 drone windows) are the case a real microphone produces. Recall at the 1% threshold:

| Model | Clean | 20 dB | 10 dB | 5 dB | 0 dB | −5 dB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| CNN (saved) | 0.954 | 0.937 | 0.827 | 0.688 | 0.500 | 0.336 |
| Transformer (saved) | **0.998** | **0.989** | 0.937 | 0.855 | 0.679 | 0.460 |
| Random forest (saved) | 0.956 | 0.921 | 0.814 | 0.707 | 0.553 | 0.366 |
| CNN-Rewindowed | 0.970 | 0.958 | 0.853 | 0.717 | 0.516 | 0.332 |
| CNN-NoiseAug | 0.945 | 0.936 | 0.874 | 0.791 | 0.630 | 0.421 |
| CNN-Tuned | 0.977 | 0.971 | **0.949** | **0.901** | **0.775** | **0.518** |

**What each step changed:**
- Rewindowing (CNN-Rewindowed) removed the dependence on the zero tail: clean recall on all windows is 0.995. It did not add noise robustness; at 0 dB it is level with the saved CNN.
- Noise augmentation (CNN-NoiseAug) added the robustness: 0.516 to 0.630 at 0 dB and 0.332 to 0.421 at −5 dB, for a small clean loss (0.970 to 0.945).
- Tuning (CNN-Tuned) recovered the clean loss and added more: 0.775 at 0 dB and 0.518 at −5 dB.

CNN-Tuned is the best model at 10 dB and below. The saved Transformer is slightly better on clean audio and at 20 dB. Its false-alarm rate also rises more with noise: 0.128 at −5 dB, against 0.071 for CNN-Tuned.

**Do not use all-window or originally-short recall as the robustness figure.** For CNN-NoiseAug and CNN-Tuned, recall on originally short windows stays near 1.0 down to 0 dB and is 0.971 and 0.847 at −5 dB, above full-length recall at every level, although those windows hold only about half a second of drone sound. CNN-Rewindowed does not show this (0.315 short against 0.332 full-length at −5 dB). Noise augmentation appears to have taught the models a new cue: drone sound in part of the window and noise alone in the rest. In this dataset that pattern occurs almost only in drone windows. About 85% of drone windows are originally short, so this inflates all-window recall.

## Report: CNN-Tuned metrics and results

**Model.** CNN-Tuned is the three-layer spectrogram CNN (92,930 parameters) trained on rewindowed windows with ESC-50 noise augmentation (folds 1–4). Its settings are the best of 5 Optuna trials, scored on validation only (clean ROC-AUC plus noisy recall at the 1% threshold, objective 1.972):

| Setting | Value |
| --- | --- |
| Learning rate | 8.7 × 10⁻⁴ |
| Batch size | 32 |
| Dropout | 0.09 |
| Weight decay | 1.6 × 10⁻⁵ |
| Noise-mix probability | 0.56 |
| Training SNR range | 2.1 to 10.2 dB |

Temperature scaling fit a temperature of 1.000. Expected calibration error is 0.001 before and after, so the raw scores were already calibrated on clean validation. The decision threshold is 0.344, the 1% false-alarm point on clean validation.

**Clean test.** On all 57,015 test windows: accuracy 0.992, precision 0.988, recall 0.996, F1 0.992, false-alarm rate 0.012, ROC-AUC 0.999, average precision 0.999. The 95% file-bootstrap interval for clean recall is 0.994 to 0.998. On full-length windows: recall 0.977, precision 0.925, F1 0.951, ROC-AUC 0.997.

**Noise test, full-length windows** (4,340 drone windows, 28,287 no-drone windows, fixed threshold 0.344):

| SNR | Recall | Precision | F1 | False-alarm rate | ROC-AUC | False share of alerts |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Clean | 0.977 | 0.925 | 0.951 | 0.012 | 0.997 | 7% |
| 20 dB | 0.971 | 0.913 | 0.941 | 0.014 | 0.996 | 9% |
| 10 dB | 0.949 | 0.878 | 0.913 | 0.020 | 0.993 | 12% |
| 5 dB | 0.901 | 0.830 | 0.864 | 0.028 | 0.987 | 17% |
| 0 dB | 0.775 | 0.716 | 0.744 | 0.047 | 0.962 | 28% |
| −5 dB | 0.518 | 0.529 | 0.524 | 0.071 | 0.884 | 47% |

The 95% file-bootstrap interval for full-length recall at 20 dB is 0.951 to 0.983.

**Findings:**
- Detection holds through moderate noise. Full-length recall stays at or above 0.90 down to 5 dB and falls to 0.775 at 0 dB and 0.518 at −5 dB. SNR compares the drone sound with the added noise: at 0 dB they are equally loud, and at −5 dB the noise has about 3 times the drone's power.
- False alarms rise as noise gets louder. The threshold holds 1.2% on clean audio but reaches 4.7% at 0 dB and 7.1% at −5 dB. Some ESC-50 categories (helicopter, airplane, engine) resemble drone sound.
- For an operator, the share of alerts that are false rises from 7% clean to 47% at −5 dB. This test set has about 6.5 no-drone windows per drone window. Where drones are rarer, the false share would be higher.
- The model's ranking survives better than its threshold. At −5 dB full-length ROC-AUC is still 0.884 while recall at the fixed threshold is 0.518. A threshold set on clean validation does not fit loud noise.
- Training noise covered 2 to 10 dB, so −5 dB lies outside the tuned training range. That may explain why CNN-Tuned scores below CNN-NoiseAug (trained on −5 to 20 dB) on all windows at −5 dB.

**Limitations:**
- Scores are for the DADS file split. DADS has no source or site label, so these results do not show performance on new microphones, sites, distances, or drone types.
- ESC-50 noise is a test-time mix, not a field recording.
- Originally short windows still carry a partial-window cue (see the evaluation report), so their noisy recall overstates robustness. A later fix is to give no-drone windows the same short-clip-and-fill structure.
- The 17% false-alarm rate on originally short no-drone windows comes from 8 of 47 windows and is not a reliable rate.
- The Optuna search was light: 5 trials of at most 8 epochs.

**Use in the system.** CNN-Tuned is the acoustic model for the handoff. Acoustic alerts are reliable in quiet to moderate noise. Near 0 dB and below, about one alert in three to one in two is false, so the agent should treat a lone acoustic alert in loud conditions as a cue for radar or thermal confirmation, not as a confirmed detection.