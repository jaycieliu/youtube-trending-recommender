# YouTube Trending Recommender (Policy-Based Ranking Simulator)

A policy-driven recommendation project that simulates **next-day Top-K slate selection** for trending videos.  
Instead of treating prediction as the final goal, this project focuses on **ranking policy decisions** and the trade-offs they create in business KPIs.

## Why this project matters

Recommendation systems are not just prediction problems — they are **decision systems**.  
A good model score still needs a policy that decides **what gets shown**, **how often**, and **under what trade-offs** (e.g., engagement vs diversity / freshness / risk).

This project evaluates multiple ranking policies offline and compares how they perform under different KPI priorities.

## What I built

- A ranking / policy simulation workflow for next-day Top-K selection
- Multiple policy baselines and rule-based ranking strategies
- Offline evaluation framework to compare policy outcomes
- Result summaries and interpretation for KPI trade-offs

## Policies compared (summary)

- **Random policy** (baseline)
- **Pct-only policy**
- **Watch-only policy**
- **Balanced policy**
- **Two-stage policy**

> See full policy definitions and evaluation details in [Project Details](docs/project-details.md).

## Key outputs

- Policy comparison tables
- Offline evaluation results
- Trade-off interpretation by policy
- Reproducible notebook workflow

## Quick links

- [Project details (full methodology + results)](reports/project-details.md)
- [Notebook](notebooks/01_model_and_policy.ipynb)
- [Policy comparison table](reports/tables/policy_comparison.md)

## How to run (minimal)

1. Open the main notebook
2. Run cells in order to generate predictions / policy simulation / evaluation
3. Review output tables in `reports/`

---

## Tech Stack

- **Languages:** Python
- **Libraries:** pandas, NumPy, scikit-learn, matplotlib
- **Environment:** Jupyter Notebook / VS Code

---

## Repo structure

```text
youtube-trending-recommender/
├── README.md
├── notebooks/
│   ├── 00_load_clean.ipynb
│   └── 01_model_and_policy.ipynb
├── reports/
│   └── project_details.md
│   └── tables/
│       ├── policy_comparison.csv
│       └── policy_comparison.md
```
