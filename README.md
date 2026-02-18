# YouTube Engagement Recommender
### Next-Day KPI Forecasting + Ranking Policy Simulator

This project builds an **offline recommender-policy simulator** that selects a daily **Top-K slate** of videos to optimize **next-day engagement**.

We model two KPIs (**completion depth** and **total watch time**), then compare multiple ranking policies—including a **balanced weighted policy** that explicitly manages KPI trade-offs.

> **Core idea:** prediction is not the end goal—**ranking policy** is.  
> A good model can still produce a bad product outcome if the ranking objective is misaligned.

---

## 1) Problem Definition

Each day we make a product decision:

- On **day _t_**, choose **K videos** to promote (the “slate”)
- Evaluate performance on **day _t+1_** using next-day outcomes

### Targets (Next-Day KPIs)

- **Primary:** `y_next_avg_pct_viewed`  
  *Average % of the video watched on the next day (depth/completion).*

- **Secondary:** `y_next_watch_time_log`  
  *Next-day watch time, modeled in log space for stability.*

When reporting watch time in real units, we invert the log transform with:

- `watch_time_real = expm1(y_next_watch_time_log)` (i.e., `np.expm1`)

---

## 2) Data & Features ✅

**Unit of observation:** (video, date)

**Features include:** daily views, likes/dislikes, comments, subs change, video length, etc.

**Prediction goal:** estimate next-day engagement metrics for each (video, day).

> Note: some policies heavily prefer **short videos** (higher completion %) even if they reduce total watch time.  
> This is a key reason the KPI trade-off appears.

---

## 3) Method Overview

### Step A — Forecast Next-Day KPIs

We fit regression models to produce:

- `pred_pct` = predicted `y_next_avg_pct_viewed`
- `pred_watch_log` = predicted `y_next_watch_time_log`

### Step B — Policy Simulator (Daily Top-K Slate)

For each day, we rank candidates and take the **Top-K**.
We compare the policies below.

---

## 4) Ranking Policies Compared

### (P0) Random Slate (baseline)

Randomly sample **K videos per day** (no model / no learning).  
This serves as the **true baseline** for uplift comparisons.

---

### (P1) 1-Stage: Pct-Only

Rank by `pred_pct` per day → pick Top-K.

**Expected behavior**
- ↑ Avg % viewed (completion depth)
- ↓ Watch time (often), because it over-selects shorter videos that are easier to finish

---

### (P2) 1-Stage: Watch-Only

Rank by `pred_watch_log` per day → pick Top-K.

**Expected behavior**
- ↑ Watch time (favors longer videos)
- ↓ Avg % viewed (often), because longer videos are harder to complete

---

### (P3) 1-Stage: Balanced Weighted Score ✅ (selected on VALID)

Compute a single score from standardized predictions:

**Score:** `score = w_pct · z(pred_pct) + (1 - w_pct) · z(pred_watch_log)`

Rank by `score` per day → pick Top-K.

**Why z-scores?**  
Because `% viewed` and `watch_time_log` live on different scales—z-scoring makes weights interpretable.

**Chosen weight (VALID):** `w_pct = 0.50`  
This is selected to target a **middle ground** (avoid both extremes).

---

### (P4) 2-Stage: Pct → Watch

Stage 1: per day, keep top **N candidates** by `pred_pct`  
Stage 2: within those candidates, rank by `pred_watch_log` → pick Top-K

**Note:** choose **N on VALID**, then lock it for TEST (do not tune on TEST).

---

## 5) Offline Evaluation Protocol

We simulate the “daily Top-K decision” and evaluate outcomes using **true next-day labels**:

1. **Slate construction (decision rule):** choose videos using predictions (`pred_*`)
2. **Outcome evaluation:** compute KPI using the **true** labels (`y_next_*`) of the chosen slate
3. **Uplift vs baseline:** compare against **random slate baseline (P0)**  
   \[
   uplift = slate\_mean - random\_baseline\_mean
   \]

We report both:
- `y_next_avg_pct_viewed` (higher is better)
- `expm1(y_next_watch_time_log)` (higher is better, in real units)

Outputs are saved in:
- `reports/tables/policy_comparison.csv`
- `reports/tables/policy_comparison.md`

---

## 6) Results (Key Takeaways)

- End-to-end notebook: [`notebooks/01_model_and_policy.ipynb`](notebooks/01_model_and_policy.ipynb)
- Policy comparison (Markdown): [`reports/tables/policy_comparison.md`](reports/tables/policy_comparison.md)
- Policy comparison (CSV): [`reports/tables/policy_comparison.csv`](reports/tables/policy_comparison.csv)

**Main finding:** KPI optimization creates a real trade-off:
- Pct-only maximizes completion but sacrifices watch time
- Watch-only does the opposite
- Balanced weighted score provides a **middle ground**

---

## Key Figures (TEST)

### KPI levels (Avg % Viewed + Watch Time)

<img src="reports/figures/kpis_test.png" width="760" />

### KPI uplift vs random baseline

<img src="reports/figures/test_kpis_comparison.png" width="760" />

### Video length distribution by policy

<img src="reports/figures/video_length_distribution_test.png" width="760" />

### Policy comparison (table)

<img src="reports/figures/test_policy_comparison.png" width="760" />

---

## What the Results Mean

- **Pct-only policy** increases **completion depth** (% viewed) but reduces **total watch time**.  
  *Interpretation:* it over-selects shorter videos that are easier to finish.

- **Watch-only policy** increases **watch time** but reduces **% viewed**.  
  *Interpretation:* it shifts toward longer videos, raising minutes watched but lowering completion fraction.

- **Balanced weighted policy (w_pct = 0.50)** achieves a **middle ground**:
  - meaningful uplift in **% viewed**
  - small positive uplift in **watch time**
  
- **2-stage (pct → watch)** improves watch time modestly while keeping % viewed close to baseline, depending on chosen **N**.

---

## 7) Why This Trade-Off Happens (Intuition)

Even when KPIs are positively correlated overall, **ranking introduces selection bias**:

- Maximizing `% viewed` tends to favor **shorter videos**  
  → higher completion fraction, but lower total minutes watched

- Maximizing watch time tends to favor **longer videos**  
  → higher minutes, but lower completion fraction

A good ranking policy must either:
- **explicitly optimize one KPI**, or
- **combine KPIs** (weighted score / constraints / Pareto frontier)

---

## 8) Reproduce

```bash
pip install -r requirements.txt
jupyter notebook notebooks/01_model_and_policy.ipynb
