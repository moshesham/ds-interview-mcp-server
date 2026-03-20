# Cohort Analysis & Retention

## Overview
Cohort analysis groups users by a shared characteristic (usually signup date) and
tracks their behavior over time. It is the go-to technique for understanding retention,
engagement decay, and the impact of product changes on user behavior.

## Key Concepts

### Cohort Definition
- **Acquisition cohort** — grouped by signup/install date (most common).
- **Behavioral cohort** — grouped by first action (first purchase, first post).
- **Segment cohort** — grouped by attribute (country, plan, device).

### Retention Metrics
| Metric        | Description                                    |
|---------------|------------------------------------------------|
| D1 retention  | % of users active 1 day after signup           |
| D7 retention  | % of users active 7 days after signup          |
| D30 retention | % of users active 30 days after signup         |
| Rolling ret.  | Active in any window during the period         |
| Bounded ret.  | Active exactly on day N                        |
| DAU/MAU       | Stickiness ratio (higher = more engaged base)  |

### Retention Curve Shapes
- **Flat tail** — healthy product, users who stay tend to stick.
- **Steep early decay** — activation problem; users don't find value quickly.
- **Continuous decline** — product-market fit concern; core loop not sticky.
- **Smile curve** — users come back (re-engagement or seasonal product).

## SQL Patterns for Cohort Analysis

### Basic Cohort Retention Query
```sql
WITH user_cohort AS (
    SELECT
        user_id,
        DATE_TRUNC('month', signup_date) AS cohort_month
    FROM users
),
user_activity AS (
    SELECT
        a.user_id,
        uc.cohort_month,
        DATE_TRUNC('month', a.activity_date) AS activity_month
    FROM activity a
    JOIN user_cohort uc ON a.user_id = uc.user_id
)
SELECT
    cohort_month,
    DATE_DIFF('month', cohort_month, activity_month) AS month_number,
    COUNT(DISTINCT user_id) AS active_users,
    COUNT(DISTINCT user_id)::FLOAT /
        FIRST_VALUE(COUNT(DISTINCT user_id)) OVER (
            PARTITION BY cohort_month ORDER BY activity_month
        ) AS retention_rate
FROM user_activity
GROUP BY cohort_month, activity_month
ORDER BY cohort_month, month_number;
```

### Retention Heatmap Data
```sql
-- Pivot-ready retention data for heatmap visualization
SELECT
    cohort_month,
    month_number,
    ROUND(retention_rate * 100, 1) AS retention_pct
FROM cohort_retention
ORDER BY cohort_month, month_number;
```

## Python Visualization (Seaborn Heatmap)
```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

# Assuming retention_df has columns: cohort_month, month_number, retention_pct
pivot = retention_df.pivot(index='cohort_month', columns='month_number', values='retention_pct')

plt.figure(figsize=(14, 8))
sns.heatmap(pivot, annot=True, fmt='.0f', cmap='YlOrRd_r', vmin=0, vmax=100)
plt.title('Monthly Cohort Retention (%)')
plt.xlabel('Months Since Signup')
plt.ylabel('Cohort Month')
plt.tight_layout()
plt.show()
```

## Product Interpretation Guide
- Compare adjacent cohorts: did a product change improve retention?
- Look at the M1→M2 drop: this is usually the largest and most actionable.
- If D7 retention < 20%, focus on activation before growth.
- Benchmark: consumer apps D1 ~40%, D30 ~15%; SaaS M1 ~80%, M12 ~60%.

## Interview Tips
- Always define your retention window and cohort grain clearly.
- Mention bounded vs unbounded retention — interviewers notice this.
- Tie retention to product decisions: "If M2 drops, I'd investigate onboarding
  changes between those cohorts."

## Practice Questions
1. Build a weekly cohort retention table from raw event data. Interpret the heatmap.
2. D7 retention dropped 3pp across all cohorts last month. List 5 hypotheses.
3. Compare monthly vs weekly cohorts — when would you use each?
