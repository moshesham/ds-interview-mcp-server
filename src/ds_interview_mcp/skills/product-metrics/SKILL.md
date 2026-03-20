---
name: product-metrics-frameworks
description: Define, measure, and optimize product metrics. Apply AARRR pirate metrics, HEART framework, North Star metrics, and Goal-Signal-Metric (GSM) process. For product analytics, growth measurement, feature launch evaluation, and KPI design at tech companies.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Product Metrics Frameworks

## Overview

Product metrics are quantitative measures used to track and assess the success of a product. Choosing the right metrics is one of the most critical skills for a product data analyst—poor metric choices lead to misaligned incentives, wasted engineering effort, and incorrect product decisions.

**Core frameworks covered:**
- **AARRR (Pirate Metrics)**: Acquisition → Activation → Retention → Revenue → Referral
- **HEART**: Happiness, Engagement, Adoption, Retention, Task Success
- **North Star Metric**: The single metric that best captures core product value
- **GSM (Goal-Signal-Metric)**: Structured process for deriving metrics from business goals

## When to Use This Skill

Use this skill when:

- **Feature launch evaluation**: Defining success metrics and guardrail metrics for a new feature
- **Product health monitoring**: Building dashboards that track the right KPIs
- **Interview preparation**: Answering "How would you measure success of X?" questions
- **Metric investigation**: Diagnosing why a metric moved up or down
- **Executive reporting**: Choosing metrics that tell a clear business story
- **OKR/goal setting**: Translating business objectives into measurable outcomes

## AARRR Pirate Metrics Framework

The AARRR framework covers the full customer lifecycle:

| Stage | Question | Example Metrics |
|-------|----------|----------------|
| **Acquisition** | How do users find us? | Signups, app installs, landing page visits |
| **Activation** | Do users have a great first experience? | Onboarding completion rate, time-to-first-value |
| **Retention** | Do users come back? | D1/D7/D30 retention, WAU/MAU ratio |
| **Revenue** | Can we monetize? | ARPU, conversion to paid, LTV |
| **Referral** | Do users tell others? | Invite rate, viral coefficient (k-factor) |

```python
import pandas as pd
from datetime import datetime, timedelta

def calculate_aarrr_metrics(events_df: pd.DataFrame) -> dict:
    """
    Calculate AARRR funnel metrics from an events table.

    Args:
        events_df: DataFrame with columns [user_id, event_name, timestamp, revenue]
                   event_name values: 'signup', 'activation', 'purchase', 'referral'

    Returns:
        Dict with AARRR stage counts and conversion rates
    """
    total_signups = events_df[events_df['event_name'] == 'signup']['user_id'].nunique()
    activated = events_df[events_df['event_name'] == 'activation']['user_id'].nunique()
    purchasers = events_df[events_df['event_name'] == 'purchase']['user_id'].nunique()
    referrers = events_df[events_df['event_name'] == 'referral']['user_id'].nunique()

    # Retention: users active in week 2 who were active in week 1
    events_df['week'] = (events_df['timestamp'] - events_df['timestamp'].min()).dt.days // 7
    week0_users = set(events_df[events_df['week'] == 0]['user_id'])
    week1_users = set(events_df[events_df['week'] == 1]['user_id'])
    retained = len(week0_users & week1_users)

    return {
        'acquisition': total_signups,
        'activation': activated,
        'activation_rate': activated / total_signups if total_signups else 0,
        'retention_week1': retained / len(week0_users) if week0_users else 0,
        'revenue_users': purchasers,
        'monetization_rate': purchasers / activated if activated else 0,
        'referral_users': referrers,
        'viral_coefficient': referrers / total_signups if total_signups else 0,
    }
```

## HEART Framework (Google)

Developed at Google for measuring UX quality at scale:

| Dimension | What It Measures | Signal Examples | Metric Examples |
|-----------|-----------------|-----------------|-----------------|
| **Happiness** | User satisfaction | Survey responses, NPS | CSAT score, NPS |
| **Engagement** | Depth of usage | Actions per session | DAU/MAU, sessions/week |
| **Adoption** | New user uptake | New feature usage | % users adopting feature in 7d |
| **Retention** | Users coming back | Return visits | D7 retention, churn rate |
| **Task Success** | Efficiency & effectiveness | Task completion | Success rate, time-on-task |

### Applying GSM (Goal-Signal-Metric) Process

```python
def define_heart_metrics(feature_name: str) -> dict:
    """
    Template for applying the GSM process to HEART dimensions.

    For each HEART dimension:
        Goal   → What is the desired user outcome?
        Signal → What user behavior indicates progress toward the goal?
        Metric → How do we quantify the signal?

    Returns:
        Dict mapping each HEART dimension to {goal, signal, metric}
    """
    template = {
        'Happiness': {
            'goal': f'Users are satisfied with {feature_name}',
            'signal': 'Post-interaction survey responses, support tickets',
            'metric': 'Average CSAT score (1-5), NPS'
        },
        'Engagement': {
            'goal': f'Users actively interact with {feature_name}',
            'signal': 'Frequency and depth of feature usage',
            'metric': 'Actions per session, DAU/WAU ratio'
        },
        'Adoption': {
            'goal': f'New users discover and try {feature_name}',
            'signal': 'First-time usage events',
            'metric': '% of eligible users who used feature within 7 days of launch'
        },
        'Retention': {
            'goal': f'Users continue using {feature_name} over time',
            'signal': 'Repeat usage after first interaction',
            'metric': 'D7/D28 feature retention rate'
        },
        'Task Success': {
            'goal': f'Users accomplish their goal efficiently with {feature_name}',
            'signal': 'Task completion, error rates',
            'metric': 'Completion rate, median time-to-complete'
        }
    }
    return template
```

## North Star Metric

The **North Star Metric (NSM)** is the single metric that best captures the core value your product delivers to customers.

### Choosing a North Star Metric

| Product Type | Example NSM | Why |
|-------------|-------------|-----|
| Social media | Daily Active Users who post or comment | Captures engagement, not passive browsing |
| E-commerce | Weekly purchases per active buyer | Captures repeat purchase behavior |
| SaaS B2B | Weekly active teams completing workflows | Captures team-level adoption |
| Streaming | Hours streamed per subscriber per week | Captures content consumption depth |
| Marketplace | Transactions completed per week | Captures supply-demand matching |

### Guardrail / Counter-Metrics

Always pair your North Star with guardrail metrics to catch unintended consequences:

```python
def north_star_dashboard(primary_metric: float, guardrails: dict) -> dict:
    """
    Evaluate North Star metric alongside guardrail metrics.

    Args:
        primary_metric: Current value of the North Star metric
        guardrails: Dict of {metric_name: (current_value, threshold, direction)}
                    direction: 'above' means healthy if current >= threshold

    Returns:
        Dashboard summary with alerts
    """
    alerts = []
    for name, (value, threshold, direction) in guardrails.items():
        if direction == 'above' and value < threshold:
            alerts.append(f"⚠️ {name}: {value:.2f} below threshold {threshold:.2f}")
        elif direction == 'below' and value > threshold:
            alerts.append(f"⚠️ {name}: {value:.2f} above threshold {threshold:.2f}")

    return {
        'north_star': primary_metric,
        'guardrails': guardrails,
        'alerts': alerts,
        'healthy': len(alerts) == 0
    }

# Example usage for a social media product
result = north_star_dashboard(
    primary_metric=1_250_000,  # DAU who post or comment
    guardrails={
        'content_quality_score': (0.82, 0.75, 'above'),
        'spam_rate': (0.03, 0.05, 'below'),
        'avg_session_duration_min': (12.5, 8.0, 'above'),
        'uninstall_rate': (0.02, 0.03, 'below'),
    }
)
```

## Metric Investigation Framework

When a metric moves unexpectedly, follow this structured approach:

1. **Verify the data**: Is the change real? Check for logging issues, ETL delays, duplicates
2. **Quantify the change**: How big is it? Is it statistically significant?
3. **Segment the data**: Break down by platform, region, user cohort, device
4. **Check for external factors**: Holidays, competitor launches, press coverage, outages
5. **Trace the funnel**: Which part of the funnel changed? Upstream or downstream?
6. **Formulate hypotheses**: Generate 3-5 possible explanations, rank by likelihood
7. **Test hypotheses**: Use data to validate or eliminate each one

```sql
-- Example: Investigate a metric drop by segmenting
WITH daily_metrics AS (
    SELECT
        DATE(event_time) AS event_date,
        platform,
        country,
        COUNT(DISTINCT user_id) AS dau,
        COUNT(*) AS total_events
    FROM events
    WHERE event_date BETWEEN CURRENT_DATE - INTERVAL '14 days' AND CURRENT_DATE
    GROUP BY 1, 2, 3
)
SELECT
    event_date,
    platform,
    country,
    dau,
    total_events,
    LAG(dau) OVER (PARTITION BY platform, country ORDER BY event_date) AS prev_day_dau,
    ROUND(100.0 * (dau - LAG(dau) OVER (PARTITION BY platform, country ORDER BY event_date))
        / NULLIF(LAG(dau) OVER (PARTITION BY platform, country ORDER BY event_date), 0), 2) AS dau_pct_change
FROM daily_metrics
ORDER BY event_date DESC, ABS(dau - LAG(dau) OVER (PARTITION BY platform, country ORDER BY event_date)) DESC NULLS LAST;
```

## Best Practices

- **Start with the user outcome**, not the metric. Ask "what does success look like for the user?" before choosing numbers
- **Every primary metric needs a counter-metric** to detect gaming or unintended side effects
- **Prefer ratio metrics** (rates, per-user averages) over absolute counts for comparing across segments
- **Use leading indicators** (e.g., activation rate) alongside lagging indicators (e.g., revenue) for early signal
- **Define metrics precisely** in a data dictionary: numerator, denominator, time window, filters, edge cases
- **Avoid vanity metrics**: total signups, page views, and downloads tell you very little about product health

## Additional Resources

- Croll, A. & Yoskovitz, B. — *Lean Analytics*
- McClure, D. — "Startup Metrics for Pirates" (AARRR)
- Rodden, K. et al. — "Measuring the User Experience on a Large Scale" (HEART Framework, Google)
- Amplitude — "North Star Playbook"
- Reforge — "Growth Series: Engagement & Retention"
