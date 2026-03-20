---
name: churn-analytics-prediction
description: Churn analysis and prediction for product analytics. Covers churn definition frameworks, survival analysis, RFM segmentation, predictive modeling, and retention intervention strategies. Includes SQL churn queries, Python survival curves, and feature engineering for churn prediction models.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Churn Analytics & Prediction

## Overview

Churn — the rate at which users stop using a product — is one of the most critical metrics for any subscription or engagement-based business. Understanding, measuring, and predicting churn enables proactive retention strategies that are far more cost-effective than acquiring new users.

**Key concepts:**
- **Churn rate calculation**: Contractual vs. non-contractual churn definitions
- **Survival analysis**: Time-to-event modeling for churn
- **RFM segmentation**: Recency, Frequency, Monetary value for user scoring
- **Predictive modeling**: Feature engineering and models for churn prediction
- **Intervention design**: Targeted retention campaigns based on churn signals

## When to Use This Skill

Use this skill when:

- **Measuring user health**: Tracking churn rates across cohorts and segments
- **Building early warning systems**: Identifying at-risk users before they leave
- **Interview questions**: "How would you predict churn for a subscription product?"
- **ROI analysis**: Quantifying the value of retention improvements
- **Growth accounting**: Decomposing user base changes into churn/resurrection/new

## Churn Definitions

### Contractual vs. Non-Contractual

| Type | Examples | Churn Definition | Challenge |
|------|---------|-----------------|-----------|
| **Contractual** | Netflix, Spotify, SaaS | User cancels subscription | Clear signal; predict before cancellation |
| **Non-contractual** | E-commerce, social media, gaming | User stops engaging | No explicit signal; must define inactivity threshold |

### Defining Non-Contractual Churn

```sql
-- Define churn as no activity in the past 28 days
-- Choose threshold based on natural usage frequency of your product
WITH user_last_activity AS (
    SELECT
        user_id,
        MAX(event_date) AS last_active_date,
        MIN(event_date) AS first_active_date,
        COUNT(DISTINCT event_date) AS active_days
    FROM events
    GROUP BY user_id
)
SELECT
    user_id,
    last_active_date,
    first_active_date,
    active_days,
    CURRENT_DATE - last_active_date AS days_since_last_active,
    CASE
        WHEN CURRENT_DATE - last_active_date > 28 THEN 'churned'
        WHEN CURRENT_DATE - last_active_date > 14 THEN 'at_risk'
        ELSE 'active'
    END AS user_status
FROM user_last_activity;
```

### Choosing the Inactivity Threshold

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt


def find_churn_threshold(events_df: pd.DataFrame, user_col: str = "user_id",
                         date_col: str = "event_date") -> None:
    """Analyze inter-visit gaps to choose an appropriate churn threshold.
    
    Plot the distribution of days between consecutive visits;
    the natural break point suggests the churn threshold.
    """
    # Calculate gaps between consecutive visits per user
    sorted_events = events_df.sort_values([user_col, date_col])
    sorted_events["prev_date"] = sorted_events.groupby(user_col)[date_col].shift(1)
    sorted_events["gap_days"] = (sorted_events[date_col] - sorted_events["prev_date"]).dt.days
    gaps = sorted_events["gap_days"].dropna()
    
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))
    
    # Distribution of gaps
    axes[0].hist(gaps[gaps <= 90], bins=90, edgecolor="white", alpha=0.7)
    axes[0].set_xlabel("Days between visits")
    axes[0].set_ylabel("Count")
    axes[0].set_title("Distribution of Inter-Visit Gaps")
    
    # Cumulative distribution
    sorted_gaps = np.sort(gaps[gaps <= 90])
    axes[1].plot(sorted_gaps, np.arange(1, len(sorted_gaps) + 1) / len(sorted_gaps))
    axes[1].set_xlabel("Days between visits")
    axes[1].set_ylabel("Cumulative proportion")
    axes[1].set_title("CDF of Inter-Visit Gaps")
    for threshold in [7, 14, 28, 60]:
        pct = (gaps <= threshold).mean()
        axes[1].axvline(threshold, color="red", linestyle="--", alpha=0.5)
        axes[1].text(threshold + 1, pct - 0.05, f"{threshold}d: {pct:.0%}", fontsize=9)
    
    plt.tight_layout()
    plt.show()
    
    # Print percentile analysis
    print("\nGap Analysis:")
    for p in [50, 75, 90, 95, 99]:
        print(f"  P{p}: {gaps.quantile(p/100):.0f} days")
```

## Survival Analysis

### Kaplan-Meier Survival Curves

```python
from lifelines import KaplanMeierFitter


def plot_survival_curves(
    df: pd.DataFrame,
    duration_col: str = "tenure_days",
    event_col: str = "churned",
    segment_col: str = None,
    title: str = "User Survival Curve"
) -> dict:
    """Plot Kaplan-Meier survival curves, optionally segmented.
    
    Args:
        df: DataFrame with tenure and churn columns
        duration_col: Column with time-to-event (days since signup)
        event_col: Column with event indicator (1=churned, 0=censored/active)
        segment_col: Optional column to segment curves
    
    Returns:
        Dictionary mapping segment to median survival time
    """
    fig, ax = plt.subplots(figsize=(12, 6))
    results = {}
    
    if segment_col is None:
        kmf = KaplanMeierFitter()
        kmf.fit(df[duration_col], event_observed=df[event_col])
        kmf.plot_survival_function(ax=ax)
        results["overall"] = kmf.median_survival_time_
    else:
        for segment in sorted(df[segment_col].unique()):
            mask = df[segment_col] == segment
            kmf = KaplanMeierFitter()
            kmf.fit(df.loc[mask, duration_col], event_observed=df.loc[mask, event_col], label=str(segment))
            kmf.plot_survival_function(ax=ax)
            results[segment] = kmf.median_survival_time_
    
    ax.set_xlabel("Days since signup")
    ax.set_ylabel("Survival probability")
    ax.set_title(title)
    ax.grid(True, alpha=0.3)
    plt.tight_layout()
    plt.show()
    
    return results
```

### Cox Proportional Hazards

```python
from lifelines import CoxPHFitter


def fit_cox_model(df: pd.DataFrame, duration_col: str, event_col: str,
                  covariates: list[str]) -> CoxPHFitter:
    """Fit a Cox proportional hazards model to identify churn risk factors.
    
    Returns the fitted model with hazard ratios for each covariate.
    Hazard ratio > 1 means higher churn risk.
    """
    cph = CoxPHFitter()
    cph.fit(df[[duration_col, event_col] + covariates],
            duration_col=duration_col, event_col=event_col)
    
    cph.print_summary()
    cph.plot()
    plt.title("Cox Model — Hazard Ratios")
    plt.tight_layout()
    plt.show()
    
    return cph

# Example usage:
# covariates = ["completed_onboarding", "connected_friends", "platform_ios",
#                "signup_source_organic", "first_week_sessions"]
# model = fit_cox_model(users_df, "tenure_days", "churned", covariates)
```

## RFM Segmentation

```python
def rfm_segmentation(
    transactions_df: pd.DataFrame,
    user_col: str = "user_id",
    date_col: str = "event_date",
    value_col: str = "revenue",
    reference_date: str = None
) -> pd.DataFrame:
    """Compute RFM scores and segments for all users.
    
    Recency: Days since last activity (lower = better)
    Frequency: Number of active days (higher = better)
    Monetary: Total revenue/value (higher = better)
    """
    if reference_date is None:
        ref = transactions_df[date_col].max() + pd.Timedelta(days=1)
    else:
        ref = pd.to_datetime(reference_date)
    
    rfm = (
        transactions_df.groupby(user_col)
        .agg(
            recency=(date_col, lambda x: (ref - x.max()).days),
            frequency=(date_col, "nunique"),
            monetary=(value_col, "sum"),
        )
        .reset_index()
    )
    
    # Score each dimension 1-5 using quintiles
    for col in ["frequency", "monetary"]:
        rfm[f"{col}_score"] = pd.qcut(rfm[col], q=5, labels=[1, 2, 3, 4, 5], duplicates="drop")
    # Recency: lower is better, so reverse
    rfm["recency_score"] = pd.qcut(rfm["recency"], q=5, labels=[5, 4, 3, 2, 1], duplicates="drop")
    
    # Combined RFM score
    rfm["rfm_score"] = (
        rfm["recency_score"].astype(int) +
        rfm["frequency_score"].astype(int) +
        rfm["monetary_score"].astype(int)
    )
    
    # Segment labels
    def label_segment(row):
        r, f, m = int(row["recency_score"]), int(row["frequency_score"]), int(row["monetary_score"])
        if r >= 4 and f >= 4:
            return "Champions"
        elif r >= 3 and f >= 3:
            return "Loyal Customers"
        elif r >= 4 and f <= 2:
            return "New Customers"
        elif r <= 2 and f >= 3:
            return "At Risk"
        elif r <= 2 and f <= 2:
            return "Lost"
        else:
            return "Need Attention"
    
    rfm["segment"] = rfm.apply(label_segment, axis=1)
    return rfm
```

## Feature Engineering for Churn Prediction

### Key Feature Categories

```python
def engineer_churn_features(events_df: pd.DataFrame, users_df: pd.DataFrame,
                            prediction_date: str) -> pd.DataFrame:
    """Build churn prediction features from event and user data.
    
    Features are computed as of prediction_date looking backward.
    """
    pred_date = pd.to_datetime(prediction_date)
    
    # Filter to events before prediction date
    df = events_df[events_df["event_date"] < pred_date].copy()
    
    features = users_df[["user_id"]].copy()
    
    # === ENGAGEMENT FEATURES ===
    user_engagement = df.groupby("user_id").agg(
        total_events=("event_id", "count"),
        active_days=("event_date", "nunique"),
        last_active=(("event_date", "max")),
        first_active=("event_date", "min"),
    ).reset_index()
    
    user_engagement["days_since_last_active"] = (pred_date - user_engagement["last_active"]).dt.days
    user_engagement["tenure_days"] = (pred_date - user_engagement["first_active"]).dt.days
    user_engagement["activity_rate"] = user_engagement["active_days"] / user_engagement["tenure_days"].clip(lower=1)
    
    features = features.merge(user_engagement, on="user_id", how="left")
    
    # === TREND FEATURES (recent vs. historical) ===
    recent = df[df["event_date"] >= pred_date - pd.Timedelta(days=7)]
    historical = df[(df["event_date"] >= pred_date - pd.Timedelta(days=28)) &
                    (df["event_date"] < pred_date - pd.Timedelta(days=7))]
    
    recent_activity = recent.groupby("user_id")["event_id"].count().rename("events_last_7d")
    hist_activity = historical.groupby("user_id")["event_id"].count().rename("events_prev_21d")
    
    features = features.merge(recent_activity, on="user_id", how="left")
    features = features.merge(hist_activity, on="user_id", how="left")
    features["events_last_7d"] = features["events_last_7d"].fillna(0)
    features["events_prev_21d"] = features["events_prev_21d"].fillna(0)
    features["activity_trend"] = (
        features["events_last_7d"] / features["events_prev_21d"].clip(lower=1) * 3 - 1
    )  # Normalize: 0 = same rate, positive = increasing, negative = decreasing
    
    return features
```

## Churn Rate Calculation (SQL)

```sql
-- Monthly churn rate for subscription product
WITH monthly_status AS (
    SELECT
        DATE_TRUNC('month', activity_date) AS month,
        user_id,
        subscription_status
    FROM user_daily_status
    WHERE subscription_status IN ('active', 'cancelled')
),
transitions AS (
    SELECT
        c.month AS churn_month,
        COUNT(DISTINCT CASE WHEN p.subscription_status = 'active' THEN p.user_id END) AS start_active,
        COUNT(DISTINCT CASE WHEN p.subscription_status = 'active' AND c.subscription_status = 'cancelled' THEN c.user_id END) AS churned
    FROM monthly_status c
    JOIN monthly_status p ON c.user_id = p.user_id
        AND c.month = p.month + INTERVAL '1 month'
    GROUP BY c.month
)
SELECT
    churn_month,
    start_active,
    churned,
    ROUND(100.0 * churned / NULLIF(start_active, 0), 2) AS churn_rate_pct
FROM transitions
ORDER BY churn_month;
```

## Intervention Strategies

| User Segment | Signal | Intervention |
|-------------|--------|-------------|
| At-risk (high-value) | Activity declining, was previously engaged | Personal outreach, exclusive offers, account manager |
| At-risk (low-value) | Low recent engagement | Re-engagement email, push notification with personalized content |
| Newly churned | Just cancelled or went inactive | Win-back campaign, exit survey, discount offer |
| Long-term churned | Inactive 90+ days | Brand awareness, major feature announcements |
| Trial users | Trial ending without activation | In-app nudges, feature tutorials, extend trial |

## Best Practices

- **Define churn precisely** before building models — ambiguous definitions lead to useless predictions
- **Use the right time windows**: Features should be computed from a window before the prediction date; label should be computed after
- **Avoid leakage**: Never include features that are consequences of churning (e.g., cancellation page views)
- **Combine quantitative and qualitative**: High churn risk scores + NPS/survey data → actionable insights
- **Measure intervention impact**: A/B test retention campaigns against a holdout group
- **Focus on preventable churn**: Some churn is natural (user's need was fulfilled) — focus modeling on users you can actually retain

## Additional Resources

- "Fighting Churn with Data" by Carl Gold
- lifelines documentation — Survival analysis in Python
- Customer Lifetime Value and Churn Prediction — Coursera specialization
- Supplementary: Churn Analytics & Retention (Data-Science-Analytical-Handbook)
