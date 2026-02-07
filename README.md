# YouTube Engagement Recommender (Next-Day KPI Forecast + Ranking Policy)

## Goal
Build an industry-style recommender **policy simulator** that selects a daily Top-K slate of videos to optimize next-day engagement.

**Primary KPI:** Average % viewed (next day)  
**Secondary KPI:** Watch time (next day, log-transformed for modeling)

## Dataset
Kaggle YouTube Analytics engagement dataset (daily performance + video-level metadata).

## Approach
### 1) Problem framing (what we predict)
For each video-day (t), predict next-day outcomes (t+1):
- `y_next_avg_pct_viewed`
- `y_next_watch_time_log = log1p(next_avg_watch_time)`

### 2) Modeling
Train two regression models using time-based split:
- Model A: predict next-day avg % viewed
- Model B: predict next-day watch time (log)

Metrics reported on validation/test: MAE / RMSE / R².

### 3) Ranking policies (what we deploy)
We compare 3 candidate policies:

- **1-stage (pct-only):** rank by predicted next-day avg % viewed  
- **2-stage (pct → watch):** stage 1 selects N candidates by predicted pct, stage 2 reranks by predicted watch time  
- **Balanced 1-stage (weighted):** `score = w * z(pred_pct) + (1-w) * z(pred_watch_log)`; pick Top-K by score

## Results
See table: `reports/tables/policy_comparison.csv` (and `.md` if exported).

Key takeaway:
- pct-only improves depth (% viewed) but can reduce total watch time (shorter videos bias).
- 2-stage improves watch time but may reduce avg % viewed.
- balanced weighted score can trade off both (tunable weight).

## Repo structure
- `notebooks/00_load_clean.ipynb` — data loading/cleaning + feature/label creation  
- `notebooks/00_end_to_end.ipynb` — modeling + policy simulation + evaluation  
- `reports/tables/` — exported metrics tables  
- `reports/figures/` — exported figures  

## How to run
1. Put raw data under `data/raw/...` (ignored by git)
2. Run notebooks in order:
   - `00_load_clean.ipynb`
   - `00_end_to_end.ipynb`
## Key Figures (VALID / TEST)

## Results (Policy Comparison)

We evaluated ranking policies using **true next-day KPIs** on VALID / TEST splits (offline replay).

**Slate size:** K = 100  
**2-stage candidate pool:** N = 1000 → final K = 100

### Key takeaway
- **Pct-only** (optimize avg % viewed) increases **Avg % Viewed** but hurts **Watch Time** (shorter videos).
- **2-stage (pct → watch)** increases **Watch Time** but reduces **Avg % Viewed**.
- A **balanced weighted policy** provides a trade-off between the two KPIs.

### Policy comparison table
- Markdown: [`reports/tables/policy_comparison.md`](reports/tables/policy_comparison.md)  
- CSV: [`reports/tables/policy_comparison.csv`](reports/tables/policy_comparison.csv)

