# Temporal Robustness of a Static PE Malware Detector on BODMAS

**Research question.** How quickly does a static PE malware detector degrade over time, and is the degradation driven by malware families it has never seen?

**Short answer.** On BODMAS (Aug 2019 – Sep 2020), malware detection did not degrade measurably over 11 months: monthly TPR stayed at 99.4–100% after a strict temporal split. Malware from families absent from training was missed 6.3× as often as malware from seen families (1.35% vs 0.21%), but unseen families never exceeded ~18% of a month's malware, so overall detection barely moved. The operationally significant failures were **false positives driven by changes in how benign data was collected**, not by drift in malware.

---

## Key findings

1. **No measurable time decay in malware detection within one year.** Training on Aug–Oct 2019 and testing on each month from Nov 2019 to Sep 2020, TPR ranged from 99.40% to 100%, matching a size-matched random split (99.49%). A shorter training window (Aug–Sep 2019) gave slightly lower but equally flat TPR (98.8–99.7%).
2. **Unseen families are harder, but rare.** Pooled over all test months:

   | Malware group | n | TPR | 95% CI (Wilson) | Misses |
   |---|---|---|---|---|
   | Families seen in training | 44,767 | 99.79% | [99.74, 99.82] | 96 |
   | Families unseen in training | 3,122 | 98.65% | [98.19, 99.00] | 42 |

   The intervals do not overlap. Miss-rate ratio (unseen / seen): **6.3×**; with the shorter training window: **5.0×** (97.43% vs 99.49%). The unseen share of monthly malware grew from ~1% (Nov–Dec 2019) to 18.1% (Jul 2020).
3. **False-positive spikes come from benign data collection, not gradual drift.** Feb 2020 FPR reached 28.0% (18.7% with the shorter window). Nearly all of those benign files arrived in two batches (Feb 19: 1,259 files, ~26% flagged; Feb 26: 2,382 files, ~30% flagged) immediately after a two-month gap in benign collection. Outside Feb 2020, monthly FPR stayed between 0.6% and 3.0%.
4. **A placeholder date would have produced a fake "January degradation".** 5,877 of January 2020's 5,926 benign samples are timestamped exactly 2020-01-01, 98% at 00:00:00. Left in, they push January FPR to 13.6%. These samples have no real position in time and were excluded.

---

## Data

- **Dataset:** BODMAS (Yang et al., 2021) — 134,435 PE files (77,142 benign, 57,293 malware), EMBER-format feature vectors (2,381 features), first-seen timestamps and family labels. Only the public feature vectors and metadata were used; no binaries were downloaded or executed.
- **Timestamps:** 3,712 timestamps failed pandas' default parser because of mixed string formats; all parse with `format="mixed"`. After re-parsing, rows are time-sorted.
- **Collection window:** 10,731 samples dated before Aug 2019 were all benign (as far back as 2007). Keeping them would let the model associate older timestamps with benign software, so all analysis is restricted to Aug 2019 – Sep 2020 (123,704 samples).
- **Placeholder date:** 5,987 samples dated 2020-01-01 (5,877 benign, 110 malware) were removed from all temporal evaluation and from the clean random-split baseline.
- **Families:** `Unknown` and `unknown` were merged, giving 581 families (matching the dataset paper). The 7 samples labelled `unknown` are excluded from the seen/unseen analysis.

### Benign collection regimes

| Period | Days with samples | Median benign / day | Std | Total |
|---|---|---|---|---|
| Aug–Dec 2019 | 149 | 121 | 68.3 | 20,597 |
| Jan 2 – Feb 25 2020 (excl. batches) | 40 | 2 | 3.5 | 111 |
| Mar 1 – Jul 14 2020 | 136 | 183 | 84.6 | 27,055 |
| Jul 15 – Sep 30 2020 | 63 | 147 | 6.7 | 9,130 |

Benign collection is effectively absent for two months, resumes with two large batches, and becomes near-constant from mid-July. The cause of the July change is not documented; it is reported here as an observation.

---

## Method

- **Model:** LightGBM (`num_leaves=64`, `learning_rate=0.05`, `feature_fraction=0.5`, `bagging_fraction=0.8`, up to 1,000 trees with early stopping after 50 rounds), seed 42. Features are used unnormalised, as is standard for tree models on EMBER features.
- **Validation and threshold:** 10% of each training set is held out at random for early stopping and to set the decision threshold at a target 1% FPR on held-out benign samples. Because this rests on ~1,000 benign validation scores, realised FPR on test data varies around the target and is always reported alongside TPR.
- **Temporal split (main):** train on Aug–Oct 2019 (20,049 samples: 10,759 benign, 9,290 malware; 318 families); test separately on each month Nov 2019 – Sep 2020. No retraining.
- **Temporal split (robustness):** train on Aug–Sep 2019 (11,575 samples; 238 families); test on each month Oct 2019 – Sep 2020.
- **Random-split baseline:** a random training set of the same size (20,049) drawn from the same pool, tested on the rest, so the comparison isolates the effect of temporal ordering from training-set size.
- **Seen vs unseen:** a malware sample is "unseen" if its family label does not occur among the malware in the training window.
- **Metrics:** Monthly malware prevalence varies from ~34% to ~60% (except Jan 2020, which is ~99% malware once placeholder-dated samples are removed, leaving only 49 benign files), so results are reported as TPR, FPR, malware-class F1 and ROC AUC rather than accuracy. Pooled seen/unseen TPR uses Wilson 95% intervals.

---

## Results

| Setup | AUC | F1 | TPR | FPR |
|---|---|---|---|---|
| Random split, placeholder included | 0.9997 | 0.9904 | 0.9965 | 0.0136 |
| Random split, placeholder removed | 0.9997 | 0.9927 | 0.9949 | 0.0090 |
| Temporal, per month (Aug–Oct train, placeholder removed) | ≥ 0.9966 | 0.89–0.999 | 0.994–1.000 | 0.006–0.280 |

The low end of the temporal AUC and F1 and the high end of FPR all come from Feb 2020. Jan 2020's F1 (0.999) and FPR (2.0%) rest on only 49 benign samples and should not be read as a reliable estimate.

### Figures

- `figs/fig1_temporal_tpr_fpr.png` — monthly TPR and FPR for both training windows against the random-split reference. Hollow markers indicate months with fewer than 100 benign samples (Jan 2020).
- `figs/fig2_seen_vs_unseen.png` — monthly TPR for seen vs unseen families, with the unseen-family share of malware as bars. Hollow markers indicate months with fewer than 100 unseen-family samples; counts are annotated.
- `figs/fig3_benign_daily.png` — benign samples per day (log scale), showing the Jan 1 placeholder, the Feb 19/26 batches, the Jan–Feb collection gap and the mid-July change.

Tables: `results/temporal_aug-oct.csv`, `results/temporal_aug-sep.csv`, `results/seen_vs_unseen.csv`.

---

## Limitations

- **Short horizon.** One year of data. TESSERACT reported clear time decay on Android over multi-year horizons; the defensible claim here is "no measurable decay within 12 months on BODMAS", not that static PE detectors do not decay.
- **Single source.** All samples come from one vendor's collection pipeline, which may produce more consistent features than a real deployment would see.
- **Benign sampling.** Benign collection changes over the year (gap, batches, fixed-rate sampling), so benign-side results reflect collection practice as much as software change. This is the sampling-bias problem Arp et al. describe.
- **Family labels.** "Unseen" means unseen *label*. Some unseen families may be close variants of seen ones; this was not tested. Early-month unseen estimates rest on small counts (33 samples in Nov and Dec 2019).
- **Single model and seed.** One model class, one seed, fixed hyperparameters, no retraining or threshold recalibration over time.
- **Threshold estimation.** The 1% FPR threshold is estimated from ~1,000 benign validation samples, so realised FPR varies around the target.

**Hypothesis, not tested here:** EMBER-style static features (imports, section entropy, packing indicators) may capture generic malicious traits rather than family-specific signatures, which would explain why unseen families are still detected at ~98.6%.

---

## Reproducing

Runs on a standard Google Colab CPU runtime; each model trains in a few minutes.

1. Download `bodmas.npz`, `bodmas_metadata.csv` and `bodmas_malware_category.csv` from the authors' public Google Drive folder (linked on the BODMAS website) using `gdown`.
2. Run the notebook cells in order: load and parse timestamps → restrict to the collection window → random-split baseline → temporal split → false-positive diagnosis → seen/unseen analysis with Wilson intervals → robustness checks → figures.

Dependencies: `numpy`, `pandas`, `lightgbm`, `scikit-learn`, `statsmodels`, `matplotlib`, `gdown`.

---

## References

- Yang, L., Ciptadi, A., Laziuk, I., Ahmadzadeh, A., & Wang, G. (2021). BODMAS: An Open Dataset for Learning based Temporal Analysis of PE Malware. *4th Deep Learning and Security Workshop (DLS), co-located with IEEE S&P 2021.*
- Pendlebury, F., Pierazzi, F., Jordaney, R., Kinder, J., & Cavallaro, L. (2019). TESSERACT: Eliminating Experimental Bias in Malware Classification across Space and Time. *USENIX Security Symposium.*
- Arp, D., Quiring, E., Pendlebury, F., Warnecke, A., Pierazzi, F., Wressnegger, C., Cavallaro, L., & Rieck, K. (2022). Dos and Don'ts of Machine Learning in Computer Security. *USENIX Security Symposium.*
- Anderson, H. S., & Roth, P. (2018). EMBER: An Open Dataset for Training Static PE Malware Machine Learning Models. *arXiv:1804.04637.*
