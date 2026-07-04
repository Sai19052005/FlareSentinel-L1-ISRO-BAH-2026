# FlareSentinel-L1

**Solar flare nowcasting and forecasting from Aditya-L1 soft and hard X-ray data.**

Team Kalam · Bharatiya Antariksh Hackathon (BAH) 2026 · Problem Statement 15

---

FlareSentinel-L1 detects solar flares in Aditya-L1's X-ray payloads, builds a validated flare catalogue, and forecasts flare onset and eventual class from the early rise phase. It is built on the physical coupling between the two payloads the Neupert effect, where the hard X-ray impulsive phase precedes the soft X-ray thermal peak and uses that lead as the basis for early warning.

Everything here runs on real data: 37 days of SoLEXS and 38 days of HEL1OS Level-1 observations from PRADAN/ISSDC, spanning **2 October – 10 November 2024**, during the maximum of Solar Cycle 25. No synthetic data is used at any stage.

## What the system does

The pipeline moves from raw telemetry to an operator-facing forecast in five stages:

**Detection.** A two-tier detector runs on calibrated SoLEXS flux a sensitive statistical tier for weak C-class events and a physics-informed tier (threshold plus impulsive-onset derivative) for M/X events. It produces a 309-flare catalogue with Start/Peak/Stop times and a GOES class for each event, independently reproduced from the data.

**Validation.** Detections are cross-checked against the NOAA/GOES flare event list. The detector recovers **100% of X-class (20/20)** and **98% of M-class (45/46)** events, with **100% class agreement** on matched events and a peak-time agreement of **−0.27 ± 0.90 min**.

**Dual-instrument catalogue.** HEL1OS hard X-rays are detected independently and merged with the SoLEXS catalogue. Matched events give a directly measured Neupert lead time (HXR peak ahead of SXR peak), complementing the SoLEXS-only derivative estimate (~2.8 min median).

**Forecasting.** A three-head XGBoost model, trained on 44 causal physics features with a strictly chronological train/test split, predicts flare onset within a 15-minute horizon, the eventual class, and the time-to-peak. Probabilities are isotonically calibrated.

**Escalation.** A separate classifier answers the operational question *will a flare that has just started grow to M or X?* using only the first five minutes of the rise.

## Key results

| Task | Metric | Result |
|---|---|---|
| Detection vs GOES | X-class / M-class | 100% (20/20) / 98% (45/46) |
| Peak-time accuracy | mean ± σ | −0.27 ± 0.90 min |
| Onset forecast (15 min) | TSS / TPR / FAR | 0.500 / 0.588 / 0.088 |
| Onset forecast | ROC-AUC / Brier | 0.788 / 0.067 |
| Model lead time | median before peak | 54 min |
| Time-to-peak | MAE | 12.5 min |
| Escalation (→ M/X) | TSS | 0.80 |
| Inference | per sample | 6.1 ms |

An ablation study isolates the contribution of the physics features: adding them beyond a statistical/temporal baseline lowers the false-alarm rate (0.102 → 0.083) while holding skill, confirming that the physics-derived quantities carry real forecasting information rather than being redundant.

## Physics-derived analyses

Beyond detection and forecasting, the notebook extracts several quantities directly from the spectra:

- **Neupert Conformity Index (NCI)** — a 0–1 score of how cleanly a flare's impulsive phase follows the Neupert effect. It rises with class (X 0.95, M 0.88, C 0.77); low-NCI gradual flares are, following Veronig et al. (2002), candidate eruptive events.
- **Quasi-periodic pulsations (QPP)** — detected via Lomb-Scargle with a red-noise significance test (>4σ) in two physical period bands (~11 s fast, ~103 s slow). Detected in **~60%** of flares, most strongly in X-class.
- **Temperature and emission measure** — isothermal fits to the 340-channel SoLEXS spectra give T(t) and EM(t), with the T–EM phase diagram tracing the flare's thermal evolution (15.6–58 MK range).
- **Fe XXV line (6.7 keV)** — resolved in the strongest X-class flares (centroid 6.70 keV), independently confirming the >15 MK plasma temperatures from the T/EM fits; the 6.40 keV fluorescence component is not separable at SoLEXS resolution.

## What's in this repository

- The full analysis notebook (Colab-ready), covering the complete pipeline from data loading through forecasting and the physics analyses.
- The validated flare catalogues as CSVs, including the GOES-cross-validated SoLEXS catalogue, the QPP detections, and the escalation-forecasting features and results.

An operations dashboard (multi-page Streamlit app reading these catalogues — live flare replay, filterable catalogue, GOES validation, escalation and QPP panels) is built and functional, and is currently being refined ahead of deployment.

## Data

SoLEXS (SDD2, 2–22 keV, 1 s cadence, 340 channels) and HEL1OS (CdTe, 8–70 keV, 20 s cadence) Level-1 products from PRADAN/ISSDC, 2 Oct – 10 Nov 2024. The notebook expects the day-wise archives in a Drive folder; paths are set at the top of the loading cells.

## Method notes and limitations

Stated plainly, because they matter for reading the results:

- **Flux calibration** is a single-anchor cross-calibration to GOES (using the X9.0 event of 3 Oct 2024), not a full ARF/RMF instrument-response inversion. It is appropriate for tracking relative flux evolution, which is what the forecasting uses.
- **The GOES reference list** is transcribed from NOAA SWPC event reports and covers M-class and above; Colab has no outbound network during processing. C-class detections therefore appear as "extra" against this list rather than as errors.
- **HEL1OS** does not extend the lead time beyond the SoLEXS derivative proxy in this dataset. Its value is independent confirmation (lower false-alarm rate) and the non-thermal energy band, and it is reported as such.
- **Training spans solar maximum only.** The model has not seen solar-minimum conditions; operational use would require multi-cycle training, adaptive baselines, and periodic recalibration. The architecture is designed to support this.
- The **T/EM** fit is isothermal on a multi-thermal disk-integrated spectrum, and the **Fe fluorescence** proxy is SNR-limited to a small sample — both are treated as indicative, not definitive.

## Running it

Open the notebook in Google Colab, point the data paths to the SoLEXS/HEL1OS archives, and run the cells in order. The pipeline is self-contained apart from standard scientific Python packages (`numpy`, `pandas`, `scipy`, `astropy`, `xgboost`, `scikit-learn`, `matplotlib`).

---

*Team Kalam — BAH 2026, Problem Statement 15. Data courtesy ISRO/ISSDC (PRADAN).*
