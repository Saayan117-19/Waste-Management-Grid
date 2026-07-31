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

- **Genetic Algorithm (full 59-district problem):**
  - **First attempt (pure random-start GA, pop=100, gen=300):** converged smoothly (healthy convergence curve) but to a WORSE result than Nearest Neighbor — 297.98 km vs NN's 238.86 km (-24.7%, i.e. actually worse). Root cause: pure random initialization + insufficient population diversity/generations for a 59-node search space, causing premature convergence to a mediocre local optimum. **Kept as a documented negative result** for the report — demonstrates GA's sensitivity to initialization/hyperparameters, a legitimate DAA talking point.
  - **Fixed version (v2):** seeded initial population with the Nearest Neighbor route + mutated variants of it (instead of 100% random individuals), increased pop_size 100→200 and generations 300→500, and added a **2-opt local refinement pass** on the GA's best result before finalizing.
  - **Result: 218.55 km, 11.15 sec runtime, 8.5% improvement over Nearest Neighbor.** Convergence curve confirms clean convergence (flat by ~generation 130, not still searching at 500). Map shows visibly cleaner route structure vs NN's tangled backtracking (especially the Staten Island loop).
  - Improvement is modest, not dramatic — expected and stated honestly in report: 2-opt refinement already cleans up most of NN's worst mistakes, so GA's remaining improvement margin is naturally smaller than a from-scratch comparison would suggest.

**Final Stage 4 comparison table:**
| Algorithm | Scope | Distance (km) | Runtime (sec) | Complexity | Est. Travel Time (hrs) | Est. Fuel (L) |
|---|---|---|---|---|---|---|
| Nearest Neighbor (Naive) | All 59 districts | 238.86 | 0.0007 | O(n²) | 9.55 | 95.54 |
| Genetic Algorithm + 2-opt | All 59 districts | 218.55 | 11.15 | O(pop×gen×n) | 8.74 | 87.42 |
| Held-Karp (Exact) | Top 10 hotspots only | 101.96 | 45.46 | O(n²·2ⁿ) | 4.08 | 40.78 |

Assumptions stated: avg speed 25 km/h, fuel rate 0.4 L/km (typical urban waste truck estimates — cite as assumptions, not measured data).

**Savings from GA+2-opt vs Naive NN (full 59-district run):** 20.31 km saved (8.5%), ~8.13 L fuel saved, ~0.81 hours saved per collection run.

**Output:** `route_nearest_neighbor.png`, `heldkarp_runtime_growth.png`, `ga_convergence.png`, `route_genetic_algorithm.png`, `route_comparison.csv`

**Not yet done (optional extensions):**
- Sensitivity analysis on GA hyperparameters (population size, mutation rate) for extra report depth.

---

## Stage 4 Extension — Real Road Routing + Multi-Truck VRP

**Motivation:** Original routing used haversine (straight-line) distance, which cuts through water/buildings unrealistically and doesn't reflect real bridge/tunnel infrastructure. Also, single-vehicle TSP framing doesn't reflect how a real city actually collects waste (multiple trucks, capacity limits).

**What we did:**

**1. Switched to real road-network distances via OSRM (public routing API):**
- Built a 59×59 real road distance matrix using OSRM's `/table` endpoint (free public server, driving profile).
- Result: **road distance averages 1.35x longer than haversine** (20.01 km vs 14.92 km avg) — confirms real infrastructure constraints matter. Scatter plot comparison saved as evidence.
- Re-ran Nearest Neighbor and GA+2-opt on real road distances: NN = 330.53 km, GA+2-opt = 315.79 km (9.97 sec runtime).
- Fetched actual road-following polyline geometry via OSRM's `/route` endpoint and plotted it — route now visibly threads through real bridges/tunnels (e.g. Verrazzano Bridge into Staten Island) instead of straight lines over water. This was a specific ask from the project owner — solved naturally as a side effect of switching to real road routing, since a road-network path is physically forced through actual bridge locations.

**2. Implemented Clarke-Wright Savings Algorithm (capacitated multi-truck VRP):**
- Classic greedy-merge heuristic: starts with one truck per district, iteratively merges routes with the highest "savings" score, subject to a capacity constraint.
- **First attempt used truck capacity = 8000 tons — too close to individual district demand (mean ~5,287 tons/district), leaving almost no room to merge routes.** Result: 46 trucks, mostly single-stop round trips, 1677 km total. Documented as a lesson in capacity-parameter sensitivity, not a bug.
- **Corrected: capacity raised to 25,000 tons** (~3-4x average district demand) to allow genuine consolidation. Result: **14 trucks, 653.12 km total distance, most trucks running near-full (22,000-24,700 tons out of 25,000 capacity)** — a well-utilized, realistic-looking solution.
- Fetched real road-following geometry for all 14 truck routes individually via OSRM (one request per truck) and plotted each in a distinct color — final map shows geographically coherent truck zones following real streets.

**Honesty note for report:** truck capacity here is illustrative (monthly tonnage treated as a single VRP solve) rather than a literal single-trip truck capacity (~8-20 tons in reality); a real deployment would calibrate this against actual per-trip capacity and split monthly tonnage across many trips.

**Final comparison table (Stage 4 complete):**
| Approach | Vehicles | Scope | Distance (km) | Runtime (sec) | Complexity | Fuel (L) | Travel Time (hrs, per vehicle) |
|---|---|---|---|---|---|---|---|
| Nearest Neighbor (Naive, real roads) | 1 | All 59 | 330.53 | ~0 | O(n²) | 132.21 | 13.22 |
| Genetic Algorithm + 2-opt (real roads) | 1 | All 59 | 315.79 | 9.97 | O(pop×gen×n) | 126.32 | 12.63 |
| Held-Karp (Exact, haversine) | 1 | Top 10 hotspots | 101.96 | 45.46 | O(n²·2ⁿ) | 40.78 | 4.08 |
| **Clarke-Wright VRP (real roads, multi-truck)** | **14** | **All 59** | **653.12** | **0.004** | **O(n² log n)** | **261.25** | **1.87 (avg per truck)** |

**Key operational insight:** VRP's total distance (653 km) is higher than single-vehicle routes because it's summed across 14 trucks, but per-truck distance (~46.6 km) and per-truck time (~1.87 hrs) are dramatically lower than a single truck attempting the full city (12.6+ hrs) — this is the real-world payoff of multi-vehicle routing: parallelized, faster completion, even though aggregate distance is higher.

**Output:** `road_vs_haversine_comparison.png`, `route_real_roads.png`, `route_vrp_clarke_wright.png` (straight-line version), `route_vrp_real_roads.png` (final, road-following, one color per truck), `final_route_comparison.csv`, `vrp_truck_summary.csv`

**Stage 4 fully complete.**

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

## Routing / Stage 4
- Distance metrics: haversine (km, initial/Held-Karp benchmark) AND real road-network distance via OSRM public API (final, used for all reported single/multi-vehicle results)
- Algorithms implemented: Nearest Neighbor (naive baseline), Held-Karp (exact, top-10 hotspot subset only, haversine), Genetic Algorithm (NN-seeded, pop=200, gen=500, tournament selection, ordered crossover, swap mutation) + 2-opt local refinement, Clarke-Wright Savings Algorithm (capacitated multi-truck VRP)
- Cost assumptions: avg speed 25 km/h, fuel rate 0.4 L/km, VRP truck capacity 25,000 tons (illustrative, see honesty note)
- Real road route geometry fetched via OSRM `/route` endpoint — visibly shows bridges/tunnels (e.g. Verrazzano Bridge) instead of straight lines over water

---

# Open Decisions / Things to Revisit
- [x] GA parameters finalized: pop=200, gen=500, NN-seeded, +2-opt refinement.
- [x] Real road distances (OSRM) implemented, replacing haversine for final results — also solves "show bridges" request.
- [x] Multi-truck VRP (Clarke-Wright) implemented — 14 trucks, 653 km total, real road geometry per truck.
- [ ] Optional: sensitivity analysis on GA hyperparameters (population size, mutation rate) for extra report depth — nice-to-have.
- [ ] VRP truck capacity (25,000 tons) is illustrative, not a literal per-trip capacity — documented as a limitation; could be refined by rescaling monthly demand to per-trip demand for a more literal VRP framing.
- [ ] Land-use category thresholds (cv < 0.20) are fixed/arbitrary — worth a one-line sensitivity note in report.
- [ ] Consider whether "Institutional" and "Market Area" categories (n=1–2 districts) should be merged or kept separate given small sample size.
- [ ] Stage 5 (dashboard) not started yet.
