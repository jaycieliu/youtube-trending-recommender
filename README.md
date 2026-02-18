# YouTube Engagement Recommender
### Next-Day KPI Forecasting + Ranking Policy Simulator

This project builds an **offline recommender-policy simulator** that selects a daily **Top-K slate** of videos to optimize **next-day engagement**.  
We forecast two KPIs (completion depth and total watch time), then compare multiple ranking policies—including a **balanced weighted policy** that explicitly manages KPI trade-offs.

> **Core idea:** prediction is not the end goal—**ranking policy** is.  
> A good model can still produce a bad product outcome if the ranking objective is misaligned.

---

## 1) Problem Definition

Each day we make a product decision:

- On **day _t_**, choose **K videos** to promote (the “slate”)
- Evaluate performance on **day _t+1_** using next-day outcomes

### Targets (Next-Day KPIs)

- **Primary:** `y_next_avg_pct_viewed`  
  Average % of the video watched on the next day (depth/completion).
- **Secondary:** `y_next_watch_time_log`  
  Next-day watch time, modeled in log space for stability.

When reporting watch time in real units, we invert the log transform with:

- `watch_time_real = expm1(y_next_watch_time_log)` (i.e., `np.expm1`)

---

## 2) Data & Features

- **Unit of observation:** (video, date)
- **Features include:** daily views, likes/dislikes, comments, subs change, video length, etc.
- **Prediction goal:** estimate next-day engagement metrics for each (video, day)

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

### (P0) Baseline: Random Slate (sanity baseline)
Randomly sample **K videos per day** (no learning).  
This gives a fair baseline **at the same slate size**.

### (P1) 1-Stage: Pct-Only
Rank by `pred_pct` per day → pick Top-K.  
**Typical behavior:** increases % viewed, but often over-selects short videos → watch time can drop.

### (P2) 1-Stage: Watch-Only
Rank by `pred_watch_log` per day → pick Top-K.  
**Typical behavior:** increases watch time, but often selects longer videos → % viewed can drop.

### (P3) 1-Stage: Balanced Weighted Score ✅ (selected on VALID)
Compute a single score from standardized predictions:

\[
score = w_{pct}\cdot z(pred\_pct)\;+\;(1-w_{pct})\cdot z(pred\_watch\_log)
\]

Rank by `score` per day → pick Top-K.  
**Why z-scores?** `% viewed` and `watch_time_log` are on different scales—z-scoring makes weights interpretable.

### (P4) 2-Stage: Pct → Watch (candidate gating)
Stage 1: per day, keep top **N candidates** by `pred_pct`  
Stage 2: within those candidates, rank by `pred_watch_log` → pick Top-K

**Why it matters:** if **N is too large**, this becomes almost identical to watch-only.  
So **we tune N on VALID**, then lock it for TEST.

---

## 5) Offline Evaluation Protocol

We follow a standard **VALID → TEST** workflow:

### VALID (model/policy selection)
- Use **VALID** to tune:
  - `w_pct` for the balanced policy
  - `N` for the 2-stage policy (candidate pool size)

### TEST (final report)
- After selecting a policy + hyperparameters on VALID, evaluate **once** on TEST.

### Metrics
For each split (VALID / TEST), for each policy:
1. **Select slates using predictions** (`pred_pct`, `pred_watch_log`)
2. **Evaluate outcomes using true labels** (`true_pct`, `true_watch_log`, and `expm1(true_watch_log)`)

Outputs are saved in:
- [`reports/tables/`](reports/tables/)
- [`reports/figures/`](reports/figures/)

---

## 6) Notebooks & Outputs (Click to Open)

- **Notebook 1 (data prep):** [`notebooks/00_load_clean.ipynb`](notebooks/00_load_clean.ipynb)
- **Notebook 2 (model + policy):** [`notebooks/01_model_and_policy.ipynb`](notebooks/01_model_and_policy.ipynb)

Artifacts:
- **Policy comparison (Markdown):** [`reports/tables/policy_comparison.md`](reports/tables/test_policy_comparison.md)
- **Policy comparison (CSV):** [`reports/tables/test_policy_comparison.csv`](reports/tables/test_policy_comparison.csv)

---

## 7) Key Figures (TEST)

### KPI levels (Avg % Viewed + Watch Time)
<img src="reports/figures/kpis_test.png" width="680" />

### KPI uplift vs baseline (Random)
<img src="reports/figures/test_kpis_comparison.png" width="680" />

### Video length distribution (why trade-off happens)
<img src="reports/figures/video_length_distribution_test.png" width="680" />

---

## 8) What the Results Mean

- **Pct-only** strongly increases **completion depth** (% viewed) but reduces **watch time**  
  → it over-selects **short videos** that are easier to finish.

- **Watch-only** increases **watch time** but reduces **% viewed**  
  → it shifts toward **long videos**, increasing minutes but lowering completion fraction.

- **Balanced weighted** aims for a **middle ground**  
  → improves completion while keeping watch-time impact controlled.

- **2-stage (pct→watch)** is a “completion-gated watch-time optimizer”  
  → behavior depends heavily on **N** (candidate pool size).  
  If **N is too large**, it collapses into watch-only.

---

## 9) Why This Trade-Off Happens (Intuition)

Even when KPIs are positively correlated overall, **ranking introduces selection bias**:

- Maximizing `% viewed` tends to favor **shorter videos**  
  → higher completion fraction, but fewer total minutes watched.
- Maximizing watch time tends to favor **longer videos**  
  → higher minutes, but lower completion fraction.

A good ranking policy must either:
- **explicitly optimize one KPI**, or
- **combine KPIs** (weighted score / constraints / Pareto frontier)

---

## 10) Reproduce

```bash
pip install -r requirements.txt

# Run in order:
jupyter notebook notebooks/00_load_clean.ipynb
jupyter notebook notebooks/01_model_and_policy.ipynb
