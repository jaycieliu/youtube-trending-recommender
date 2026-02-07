# YouTube Engagement Recommender  
### Next-Day KPI Forecasting + Ranking Policy Simulator

Build an industry-style **policy simulator** that selects a daily **Top-K slate** of videos to optimize **next-day engagement**.  
The project predicts two next-day KPIs and compares multiple ranking policies (1-stage, 2-stage, weighted/balanced).

---

## Problem Setup

We simulate a product decision made each day:

> **On day _t_**, choose **K videos** to promote (the “slate”)  
> and evaluate how that slate performs on **day _t+1_**.

### KPIs (Targets)
- **Primary KPI:** `y_next_avg_pct_viewed` (next-day average % viewed)
- **Secondary KPI:** `y_next_watch_time_log` (next-day watch time, log-transformed for modeling)

We report watch time in **real units** using `exp(x) - 1` (via `np.expm1`) when evaluating uplift.

---

## Approach

### 1) Train Predictive Models (Next-Day Forecast)
We build regression models to predict:
- `pred_pct` → next-day avg % viewed
- `pred_watch_log` → next-day watch time (log)

### 2) Rank and Select Daily Top-K (Policy Simulator)
We compare multiple ranking policies that produce a **daily Top-K slate**:

**(A) 1-stage (pct-only)**  
Rank videos by predicted next-day % viewed (`pred_pct`), pick Top-K.

**(B) 2-stage (pct → watch)**  
Stage 1: for each day, keep top **N candidates** by `pred_pct`  
Stage 2: within those candidates, rank by `pred_watch_log`, pick Top-K

**(C) 1-stage (balanced weighted score)** ✅ recommended  
Create a single score that trades off both KPIs:
\[
score = w\_{pct}\cdot z(pred\_pct)\;+\;(1-w\_{pct})\cdot z(pred\_watch\_log)
\]
Rank by `score` and pick Top-K.

Why the weighted score matters: optimizing only `% viewed` tends to favor **shorter videos** (higher completion rate), which can reduce total watch time.

---

## Key Results (Offline Policy Evaluation)

### Policy Comparison Table
Full metrics are saved here:
- `reports/tables/policy_comparison.csv`
- `reports/tables/policy_comparison.md`

### Visuals
**Test KPIs (absolute)**
- ![kpis_test](reports/figures/kpis_test.png)

**Test KPI uplift vs baseline**
- ![test_kpis_comparison](reports/figures/test_kpis_comparison.png)

---

## Interpretation

### Trade-off: % Viewed vs Watch Time
Across the dataset, next-day `% viewed` and next-day watch time are positively correlated, but **ranking can still create a trade-off** because the policy may systematically prefer **short videos**:

- **Pct-only policy:** selects videos people finish (high %), often short → **watch time decreases**
- **2-stage policy (pct → watch):** forces watch time up in stage 2 → **% viewed decreases**
- **Balanced policy (weighted):** finds a middle ground → modest improvement in both KPIs (or at least avoids large losses)

### Balanced Policy Choice
We tune `w_pct` on the validation set (sweep from 0.10 → 0.90).  
A practical choice is the smallest `w_pct` that:
- keeps `% viewed` uplift **positive**, and  
- makes watch time uplift **non-negative** (or acceptable).

Example outcome (your run): `w_pct ≈ 0.55` improved both metrics on test.

---

## Repository Structure

```text
youtube-trending-recommender/
├── data/                         # (local) raw/processed data (not pushed if large)
├── notebooks/
│   ├── 00_load_clean.ipynb
│   └── 00_end_to_end.ipynb       # end-to-end modeling + policy simulation
├── reports/
│   ├── figures/                  # saved plots for README/report
│   └── tables/                   # policy comparison tables (csv/md)
├── src/                          # (optional) reusable functions later
└── README.md
