# Smart Waste Management System — Project Log

Living document. Update after each notebook/milestone. Pull directly from this when writing the report.

---

## Data Source
NYC DSNY Monthly Tonnage Data + NYC Community Districts boundaries (real municipal open data).
- 71 community districts exist geographically; only **59 have tonnage records** (12 excluded codes — non-standard numbering, large area, consistent with parks/airports/non-residential zones — documented as an honest limitation).

---

## Notebook 01 — `01_data_preprocessing.ipynb` (Blueprint Stage 1)

**Goal:** Join raw tonnage CSV with district boundary shapes into one clean table.

**What we did:**
- Parsed `MONTH` string into real dates.
- Built join key: `BoroCd = BOROUGH_ID × 100 + COMMUNITYDISTRICT`.
- Fixed a bug: waste tonnage columns weren't purely numeric (silent bad values) — used `pd.to_numeric(errors="coerce")` before summing into `total_tons`.
- Parsed `the_geom` WKT strings into shapely geometries.
- Joined tonnage ↔ district shapes on `BoroCd`.

**Output:** `tonnage_clean.parquet`, `districts.geojson`

**Note for report:** 59 of 71 districts have tonnage data — documented limitation, not a bug.

---

## Notebook 02 — `02_feature_engineering.ipynb` (Blueprint Stage 1, feature engineering)

**Goal:** Turn clean historical table into ML-ready features.

**What we did:**
- Temporal features: year, month, quarter, is_summer, is_winter.
- Lag features: `total_tons_lag1`, `total_tons_lag12` (per district, using `.groupby().shift()`).
- Rolling features: 3-month and 12-month trailing averages, using `.shift(1).rolling(n).mean()` — shift-before-rolling was deliberate to avoid data leakage (excluding current month from its own rolling average).
- Spatial features: district area, centroid lat/lon (via `.to_crs(epsg=2263)` projection for accurate area/centroid math, then converted back to EPSG:4326 for lat/lon).
- Dropped rows without full lag/rolling history (~708 rows lost — first ~12 months per district).

**Output:** `features.parquet` (24,411 rows × 59 districts)

**Improvised fix:** Path issues — Jupyter's working directory was `notebooks/`, not project root. Solved by hardcoding `PROJECT_ROOT` as an absolute path at the top of every notebook (standard first cell now).

---

## Notebook 03 — `03_prediction_models.ipynb` (Blueprint Stage 2)

**Goal:** Predict future waste generation per district; compare algorithms.

**What we did:**
- Time-based train/test split (train < 2024-01-01, test ≥ 2024-01-01) — explicitly NOT random k-fold, since random splitting would leak future info into training for time-series data.
- Trained 4 models: Naive baseline (predict = last month), Linear Regression, Random Forest, XGBoost.

**Results:**
| Model | RMSE | MAE | MAPE | R² |
|---|---|---|---|---|
| Random Forest | 256.5 | 185.1 | 3.56% | 0.980 |
| XGBoost | 276.8 | 197.5 | 3.86% | 0.977 |
| Linear Regression | 314.4 | 232.9 | 4.55% | 0.970 |
| Naive | 505.3 | 372.2 | 7.30% | 0.923 |

**Winner: Random Forest.** Slightly beat XGBoost — plausible given moderate dataset size (~24K rows) and strong autoregressive features; RF's bagging can outperform boosting under these conditions. Verified NOT overfit: train R² 0.981 vs test R² 0.980 (near-identical).

**Improvised — recursive vs single-step forecasting:** Originally clustered on the model's fitted values across full history (not true forecasting). Corrected to genuine forward forecasting: built a function to construct a real "next month" feature row per district using most recent actuals, predicted via the trained RF model. Chose **single-step** (next month only) over recursive multi-step, per project decision — simpler, avoids compounding recursive error.

**Output:** `models/best_model_rf.pkl`, `predictions.parquet` (full historical fitted values — mostly superseded), `next_month_forecast.parquet` (the one actually used downstream), plus diagnostic plots (actual vs predicted, residuals, feature importance, time-series).

**Environment issues resolved along the way (not project-relevant, but documented in case they recur):**
- Windows Store Python alias intercepting `python` command → used Anaconda Prompt instead.
- `waste` conda env hit Windows Smart App Control blocking pandas DLLs → switched to `base` conda env (already trusted), which resolved it.
- Jupyter kernel mismatch (notebook running on `base` while packages installed to `waste`) → registered/selected correct kernel.

---

## Notebook 04 — `04_clustering_and_hotspots.ipynb` (Blueprint Stage 3)

**Goal:** Cluster districts by waste pattern, detect hotspots, categorize by land-use.

**What we did:**
- Built `district_summary`: **PREDICTED** next-month waste (`predicted_tons`) + **HISTORICAL** volatility (`hist_avg`, `hist_std`, `hist_cv`).
- Standardized features, ran elbow method + silhouette score to pick k.
  - Elbow suggested k=4 visually; **silhouette score showed k=3 was actually better** (0.450 vs 0.366 for k=4) — went with the statistically justified choice (k=3) over the visually-suggested one. Documented as an explicit, honest tradeoff decision.
- K-Means (k=3: Low/Medium/High waste) on PREDICTED + volatility features.
- Built a **parallel HISTORICAL-only clustering** (same k=3, using `hist_avg`/`hist_cv` only) as a comparison baseline — confirms PREDICTED-based clusters aren't wildly different from historical reality (good validation).
- DBSCAN for hotspot/anomaly detection (outlier districts = `-1` label).
- Rule-based land-use categorization: combined PREDICTED waste tier + HISTORICAL volatility (stable vs spiky) → Residential / Institutional / Mixed-Use / Commercial / Industrial / Market Area. ("Public Spaces" category dropped — corresponds to the 12 districts with no tonnage data at all.)

**Results:**
- Category counts: Residential 42, Mixed-Use 11, Commercial 3, Institutional 2, Market Area 1.
- PREDICTED vs HISTORICAL category shares nearly identical (within ~2 percentage points) — good sign, model's short-term forecast is consistent with long-term structural pattern.
- Note: Market Area (n=1) and Institutional (n=2) are small groups — any "average" for them is really describing 1–2 data points, flagged as a limitation.

**Output:** `district_clusters.parquet`, `category_stats_predicted.csv`, `category_stats_historical.csv`, plus maps and charts (all consistently labeled PREDICTED vs HISTORICAL).

**Process note:** Established the convention of always explicitly labeling PREDICTED vs HISTORICAL in every chart title/axis/print statement, since early cells didn't distinguish this clearly and it matters for defending the methodology.

**File hygiene note:** Established the practice of using ONE fixed filename per chart (so `savefig` overwrites automatically rather than accumulating `_v2`, `_final`, etc.), plus a one-time cleanup cell to remove stale files from earlier iterations (e.g. `kmeans_elbow.png`, `kmeans_elbow_forecast.png` — both superseded by `kmeans_k_selection.png`).

---

## Notebook 05 — `05_route_optimization.ipynb` (Blueprint Stage 4) — IN PROGRESS

**Goal:** Optimize waste collection routes; compare algorithms (DAA core).

**What we've done so far:**
- Built distance matrix (59×59) using haversine (straight-line) distance between district centroids. Avg inter-district distance: 14.92 km, max: 47.83 km. **Limitation noted:** haversine ≠ real road distance; a defensible simplification for project scope.
- **Nearest Neighbor (naive baseline):** 238.86 km total route, runtime ~0.0005 sec. Map shows classic NN failure mode — long backtracking lines to reach stranded far-flung districts (Rockaways, Staten Island) left for the end. This visual is a strong "why we need better algorithms" argument.
- **Held-Karp exact DP** (on top-10 highest-waste districts, since full 59-node exact solve is computationally infeasible):
  - Optimal distance: **101.96 km** (ground truth for GA comparison later).
  - Runtime: 45.5 sec for n=10.
  - Empirical growth proof: n=4→10 runtime went from 0.0001s → 50.98s. n=9→10 alone was a ~12x jump. Directly validates O(n²·2ⁿ) complexity claim with real data, not just theory.

**Not yet done:**
- Genetic Algorithm (full 59-district problem).
- 2-opt improvement on NN route.
- Route comparison table (distance, time, fuel, computational complexity, % improvement over naive).
- Possibly: Clarke-Wright/VRP extension for multi-truck realism.

---

## Notebook 06 — Dashboard — NOT STARTED

---

# Feature List (Running Inventory)

## Data / Stage 1
- `total_tons` — engineered target (sum of refuse, paper, MGP, organics categories, NaN→0)
- `date`, `year`, `month_num`, `quarter`, `is_summer`, `is_winter`
- `total_tons_lag1`, `total_tons_lag12`
- `rolling_3m_avg`, `rolling_12m_avg` (leakage-safe, shifted before rolling)
- `area_sqm`, `centroid_lat`, `centroid_lon` (from district geometry, EPSG:2263 projection)

## Prediction / Stage 2
- Model: Random Forest (`n_estimators=200, max_depth=10`)
- Target: `total_tons`
- Forecast horizon: single-step (next month only)
- Output: `predicted_tons`

## Clustering / Stage 3
- Clustering features: `predicted_tons` (PREDICTED), `hist_cv` (HISTORICAL)
- k=3 (Low/Medium/High Waste) — chosen via silhouette score
- DBSCAN: `eps=0.8, min_samples=3` — outlier flag for hotspots
- `land_use_category` — rule-based: waste tier (PREDICTED) × volatility (HISTORICAL, threshold cv<0.20)

## Routing / Stage 4 (in progress)
- Distance metric: haversine (km), from district centroids
- Algorithms implemented so far: Nearest Neighbor, Held-Karp (exact, subset only)
- Algorithms planned: Genetic Algorithm, 2-opt, possibly Clarke-Wright/VRP

---

# Open Decisions / Things to Revisit
- [ ] Confirm final GA parameters (population size, generations, mutation rate) once implemented — needs a sensitivity analysis for report depth.
- [ ] Decide whether to add a multi-truck VRP extension (Clarke-Wright or OR-Tools) — currently single-vehicle TSP framing only.
- [ ] Land-use category thresholds (cv < 0.20) are fixed/arbitrary — worth a one-line sensitivity note in report.
- [ ] Consider whether "Institutional" and "Market Area" categories (n=1–2 districts) should be merged or kept separate given small sample size.
