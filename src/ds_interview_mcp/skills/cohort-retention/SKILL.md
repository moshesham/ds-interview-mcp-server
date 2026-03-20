---
name: cohort-retention-analysis
description: Perform cohort analysis and retention measurement for product analytics. Build retention curves, triangle retention tables, heatmaps, and survival analysis. Covers SQL retention queries, Python visualization, and frameworks for diagnosing and improving retention across different product types.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Cohort Retention Analysis

## Overview

Cohort analysis groups users by a shared characteristic (usually signup date) and tracks their behavior over time. Retention — the percentage of users who come back — is the single most important metric for product-market fit. Without strong retention, growth is a leaky bucket.

**Key concepts:**
- **Cohort definition**: Grouping users by signup week/month, acquisition channel, or first action
- **Retention curves**: Plotting % retained over time — the shape reveals product health
- **Triangle tables**: The classic cohort × period retention matrix
- **Unbounded vs. bounded retention**: "returned at any point in week N" vs. "returned on day N"
- **N-day, N-week, N-month retention**: Different time granularities for different products

## When to Use This Skill

Use this skill when:

- **Measuring product-market fit**: D1, D7, D30 retention are the key signals
- **Evaluating feature impact**: Did a feature launch improve retention for exposed users?
- **Interview questions**: "Calculate Day 7 retention" is a top-5 product analytics interview question
- **Growth modeling**: Retention curves feed into LTV calculations and growth models
- **Comparing segments**: Which acquisition channel, geography, or user type retains best?

## SQL Retention Queries

### Classic N-Day Retention

```sql
-- Day 1, Day 7, Day 30 retention by signup cohort
WITH user_cohorts AS (
    SELECT
        user_id,
        DATE(MIN(event_time)) AS signup_date
    FROM events
    WHERE event_name = 'signup'
    GROUP BY user_id
),
user_activity AS (
    SELECT DISTINCT
        user_id,
        DATE(event_time) AS activity_date
    FROM events
    WHERE event_name IN ('app_open', 'page_view', 'action')
)
SELECT
    DATE_TRUNC('week', c.signup_date) AS cohort_week,
    COUNT(DISTINCT c.user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '1 day' THEN c.user_id END) AS d1_retained,
    COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '7 days' THEN c.user_id END) AS d7_retained,
    COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '30 days' THEN c.user_id END) AS d30_retained,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '1 day' THEN c.user_id END) / COUNT(DISTINCT c.user_id), 1) AS d1_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '7 days' THEN c.user_id END) / COUNT(DISTINCT c.user_id), 1) AS d7_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a.activity_date = c.signup_date + INTERVAL '30 days' THEN c.user_id END) / COUNT(DISTINCT c.user_id), 1) AS d30_pct
FROM user_cohorts c
LEFT JOIN user_activity a ON c.user_id = a.user_id
GROUP BY 1
ORDER BY 1;
```

### Full Triangle Retention Table

```sql
-- Generate the complete cohort × period retention matrix
WITH user_cohorts AS (
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(event_time)) AS cohort_month
    FROM events
    WHERE event_name = 'signup'
    GROUP BY user_id
),
user_activity AS (
    SELECT DISTINCT
        user_id,
        DATE_TRUNC('month', event_time) AS activity_month
    FROM events
),
cohort_activity AS (
    SELECT
        c.cohort_month,
        EXTRACT(YEAR FROM AGE(a.activity_month, c.cohort_month)) * 12 +
            EXTRACT(MONTH FROM AGE(a.activity_month, c.cohort_month)) AS period_number,
        COUNT(DISTINCT c.user_id) AS active_users
    FROM user_cohorts c
    JOIN user_activity a ON c.user_id = a.user_id AND a.activity_month >= c.cohort_month
    GROUP BY 1, 2
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(DISTINCT user_id) AS cohort_size
    FROM user_cohorts
    GROUP BY 1
)
SELECT
    ca.cohort_month,
    cs.cohort_size,
    ca.period_number,
    ca.active_users,
    ROUND(100.0 * ca.active_users / cs.cohort_size, 1) AS retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
ORDER BY ca.cohort_month, ca.period_number;
```

## Python Retention Visualization

### Retention Heatmap

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns


def build_retention_table(events_df: pd.DataFrame) -> pd.DataFrame:
    """Build a cohort retention table from an events DataFrame.
    
    Args:
        events_df: DataFrame with columns [user_id, event_date, event_name]
    
    Returns:
        Pivot table with cohort_month as rows, period as columns, retention % as values
    """
    # Determine each user's cohort (first activity month)
    cohorts = (
        events_df.groupby("user_id")["event_date"]
        .min()
        .dt.to_period("M")
        .rename("cohort_month")
    )
    
    # Merge cohort back and compute period number
    df = events_df.merge(cohorts, on="user_id")
    df["activity_month"] = df["event_date"].dt.to_period("M")
    df["period"] = (df["activity_month"] - df["cohort_month"]).apply(lambda x: x.n)
    
    # Count distinct users per cohort × period
    cohort_data = (
        df.groupby(["cohort_month", "period"])["user_id"]
        .nunique()
        .reset_index(name="active_users")
    )
    
    # Get cohort sizes
    cohort_sizes = cohort_data[cohort_data["period"] == 0][["cohort_month", "active_users"]].rename(
        columns={"active_users": "cohort_size"}
    )
    
    # Calculate retention percentages
    cohort_data = cohort_data.merge(cohort_sizes, on="cohort_month")
    cohort_data["retention_pct"] = (100.0 * cohort_data["active_users"] / cohort_data["cohort_size"]).round(1)
    
    # Pivot to triangle table
    retention_table = cohort_data.pivot(index="cohort_month", columns="period", values="retention_pct")
    
    return retention_table


def plot_retention_heatmap(retention_table: pd.DataFrame, title: str = "Cohort Retention (%)") -> None:
    """Plot a retention heatmap from a pivot table."""
    fig, ax = plt.subplots(figsize=(14, 8))
    sns.heatmap(
        retention_table,
        annot=True,
        fmt=".0f",
        cmap="YlOrRd_r",
        vmin=0,
        vmax=100,
        ax=ax,
        cbar_kws={"label": "Retention %"},
    )
    ax.set_title(title)
    ax.set_xlabel("Period (months since signup)")
    ax.set_ylabel("Cohort")
    plt.tight_layout()
    plt.show()
```

### Retention Curve Comparison

```python
def plot_retention_curves(retention_table: pd.DataFrame, cohorts_to_plot: list = None) -> None:
    """Overlay retention curves for selected cohorts."""
    if cohorts_to_plot is None:
        cohorts_to_plot = retention_table.index[-6:]  # Last 6 cohorts
    
    fig, ax = plt.subplots(figsize=(12, 6))
    for cohort in cohorts_to_plot:
        if cohort in retention_table.index:
            curve = retention_table.loc[cohort].dropna()
            ax.plot(curve.index, curve.values, marker="o", label=str(cohort))
    
    ax.set_xlabel("Period (months since signup)")
    ax.set_ylabel("Retention (%)")
    ax.set_title("Retention Curves by Cohort")
    ax.legend(title="Cohort")
    ax.set_ylim(0, 105)
    ax.grid(True, alpha=0.3)
    plt.tight_layout()
    plt.show()
```

## Retention Benchmarks

| Product Type | D1 | D7 | D30 | D90 |
|-------------|-----|-----|------|------|
| Social apps | 40-60% | 25-40% | 15-25% | 10-20% |
| E-commerce | 20-30% | 10-20% | 5-15% | 3-10% |
| SaaS (B2B) | 80-95% | 70-85% | 60-80% | 50-70% |
| Gaming (casual) | 30-40% | 15-25% | 5-15% | 2-8% |
| Subscription media | 60-70% | 40-55% | 30-45% | 20-35% |

## Diagnosing Retention Problems

### Retention Curve Shapes

1. **Flattening curve** (good): Steep early drop, then levels off → you have a retained core, optimize onboarding
2. **Continuously declining** (bad): Never flattens → no product-market fit for most users
3. **Smile curve** (rare, great): Drops then recovers → dormant users re-engage (e.g., tax software, seasonal products)
4. **Cliff drop** (concerning): Sudden drop at a specific period → look for trial end, paywall, or content exhaustion

### Investigation Framework

1. **Segment retention** by platform, country, acquisition source, feature usage
2. **Compare power users vs. casual users** — what behaviors correlate with retention?
3. **Identify activation events** — actions in the first session that predict long-term retention
4. **Time-to-value analysis** — how quickly do retained users reach their "aha moment"?

## Best Practices

- **Use unbounded retention for trends** (did the user come back at any point in week N), bounded retention for precision
- **Minimum cohort age**: Don't show D30 retention for a cohort that's only 20 days old
- **Account for seasonality**: Compare same-period cohorts year-over-year
- **Segment before averaging**: Overall retention hides segment differences — always cut by meaningful dimensions
- **Track retention of key actions**, not just logins — revenue retention, content creation retention, etc.

## Additional Resources

- "Retention is King" essay by Brian Balfour
- Amplitude's Retention Playbook
- Reforge Growth Series — Retention & Engagement
- Casey Winters on "Sustainable Growth" (Greylock blog)
- Supplementary: Cohort Analysis Deep Dive (Data-Science-Analytical-Handbook)
