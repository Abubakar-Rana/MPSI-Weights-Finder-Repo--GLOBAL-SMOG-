# MPSI Gases Weights Finder

Derives the per-gas weights of the **Multi-Pollutant Smog Index (MPSI)** from data,
using logistic regression, instead of assigning them by hand.

The equal-weighted index averages five Sentinel-5P pollutant Z-scores:

```
MPSI = (Z_NO2 + Z_CO + Z_SO2 + Z_O3 + Z_UVAI) / 5
```

This repository replaces the equal weights with coefficients fitted from the data,
producing the weighted index

```
MPSI* = 0.185 Z_NO2 + 0.288 Z_CO + 0.630 Z_SO2 - 0.388 Z_O3 - 0.319 Z_UVAI
```

---

## How the weights are obtained

1. Monthly city Z-scores for the five pollutants (NO2, CO, SO2, O3, UVAI) are built
   from the combined Sentinel-5P record.
2. A logistic regression is fitted to separate **winter months (Oct–Feb) from the
   rest of the year**, using the five Z-scores as predictors:
   `LogisticRegression(C=np.inf, max_iter=5000, class_weight='balanced')`.
   Grouped 5-fold cross-validation (`StratifiedGroupKFold`, grouped by city) checks
   that the fit generalises across cities rather than memorising any one of them.
3. The coefficients are normalised so that `|w|` sums to 1, then divided by the
   standard deviation of the resulting score, so that **MPSI\* has unit variance**.

> **The label is seasonal, not the formula's own output.** The regression is trained
> to recognise *winter*, not to reproduce the rule-based smog flag. This matters for
> interpretation: the fitted weights describe the pollutant signature that
> distinguishes the winter smog season, which is why O3 and UVAI carry **negative**
> coefficients — both are comparatively *low* in winter relative to their own annual
> cycle, so low values are evidence *for* the winter regime.

Reported skill: **AUC 0.951 in-sample, 0.938 under grouped cross-validation.**

### Fitted coefficients

| Gas | Coefficient | Weight share | Final weight |
|---|---|---|---|
| NO2 | 0.6748 | 0.1023 | **0.1851** |
| CO | 1.0488 | 0.1590 | **0.2878** |
| SO2 | 2.2967 | 0.3482 | **0.6301** |
| O3 | −1.4150 | 0.2145 | **−0.3882** |
| UVAI (AerosolIndex) | −1.1612 | 0.1760 | **−0.3186** |

`sigma_w = 0.5525`

### Detection threshold

Because the weights rescale the index to unit variance, the detection threshold is
**TAU = 1.8**, not 1.0 — 1.8 reproduces the strictness of the original rule in SD
units (the equal-weighted index had SD 0.557). A month is flagged when

```
MPSI* >= 1.8   AND   at least 2 pollutants elevated   AND   month in the seasonal window
```

For comparison, the notebook also computes three alternative weighting schemes
(PCA, entropy and CRITIC) alongside equal weights; see `Results/MPSI_gas_weights.csv`.

---

## Repository layout

```
MPSI-Gases-Weights-Finder/
├── Data/
│   └── AllCities_combined_data.csv     input: monthly/weekly S5P record per city
├── Code/
│   └── mpsi_weights_finder.ipynb       the full pipeline
├── Results/                            all outputs are written here
│   ├── MPSI_gas_weights.csv            equal / PCA / entropy / CRITIC weights
│   ├── smog_month_refined.csv          detections under the fitted weights
│   ├── smog_month_weight_comparison.csv  detections under each weighting scheme
│   ├── smog_month_detection.csv        baseline (equal-weighted) detections
│   ├── smog_month_detection_v2.csv     revised baseline detections
│   └── Figures/                        generated figures
├── requirements.txt
└── README.md
```

The files already present in `Results/` are the outputs of a previous run, kept as a
reference so a fresh run can be checked against them. Re-running the notebook
overwrites them.

---

## Running it

```bash
pip install -r requirements.txt
jupyter lab Code/mpsi_weights_finder.ipynb
```

Run the cells top to bottom. All paths are relative to `Code/`, so the notebook works
from a fresh clone with no edits.

---

## Notes and caveats

- **Weights do not sum to 1.** They sum to 0.396, because MPSI\* is a weighted
  *contrast* (two terms are negative), not an average. Thresholds calibrated for the
  equal-weighted index therefore do not transfer directly; use TAU = 1.8.
- **SO2 dominates.** Its weight (0.630) exceeds NO2 and CO combined, so MPSI\* behaves
  close to an SO2 anomaly detector with supporting terms.
- **The seasonal label is a design choice.** Fitting against Oct–Feb defines "smog" as
  whatever distinguishes winter. A city whose pollution season falls outside that
  window will be described poorly by these weights.
- Z-scores are computed per city, so the index measures anomalies relative to each
  city's own baseline, not absolute concentrations. A detected month means "unusual
  for this city", not "worse than some absolute standard".

---

## Provenance

Code and data copied unchanged from the `CCAi-2026-SMOG` project, except that
machine-specific absolute paths were replaced with relative ones and figure output was
routed into `Results/Figures/` so the notebook is reproducible from a clone.
