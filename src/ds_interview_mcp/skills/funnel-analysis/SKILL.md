---
name: funnel-analysis-conversion
description: Build and analyze conversion funnels for product analytics. Covers funnel construction, drop-off diagnosis, segmented funnel analysis, and conversion optimization frameworks. Includes SQL funnel patterns, Python visualization, and interview approach for funnel questions.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Funnel Analysis & Conversion Optimization

## Overview

Funnel analysis tracks users through a sequence of ordered steps toward a goal (signup, purchase, activation). It answers the most fundamental product question: "Where are we losing users, and why?" Every product analytics interview includes a funnel question, and every product team relies on funnels to prioritize work.

**Key concepts:**
- **Closed funnels**: Users must complete steps in strict order
- **Open funnels**: Users can complete steps in any order (less common, used for feature adoption)
- **Step-to-step vs. overall conversion**: "What % of step 2 users reach step 3" vs. "What % of step 1 users reach step 5"
- **Time-bounded funnels**: Only count conversions within a time window (e.g., 7 days)
- **Segmented funnels**: Breaking down by platform, cohort, geography to find opportunities

## When to Use This Skill

Use this skill when:

- **Identifying the biggest drop-off point** in signup, purchase, or activation flows
- **Prioritizing product work**: A 5% improvement at the biggest drop-off has the most impact
- **A/B test analysis**: Measuring whether a variant improved step-to-step conversion
- **Interview questions**: "Users sign up but don't complete onboarding — diagnose and propose solutions"
- **Quarterly business reviews**: Reporting conversion metrics and identifying bottlenecks

## Funnel Construction

### Framework: Define → Measure → Segment → Diagnose → Act

1. **Define the funnel**: What are the steps, in what order, and within what time window?
2. **Measure overall and step-to-step conversion**: Build the baseline funnel
3. **Segment**: Cut by platform, country, new vs. returning, acquisition source
4. **Diagnose the biggest drop-off**: Where do most users leave, and why?
5. **Act**: Propose specific changes and how to measure their impact

### Common Product Funnels

| Product | Funnel Steps |
|---------|-------------|
| E-commerce | Visit → Product View → Add to Cart → Checkout → Purchase |
| SaaS | Landing Page → Signup → Onboarding → Activation → Subscription |
| Social | App Install → Signup → Profile Setup → First Post → D7 Retained |
| Marketplace | Search → View Listing → Contact Seller → Transaction |
| Content | Page View → Scroll 50% → Scroll 100% → Share/Save |

## SQL Funnel Patterns

### Time-Bounded Sequential Funnel

```sql
-- Users must complete each step within 24 hours of the previous step
WITH step1 AS (
    SELECT user_id, MIN(event_time) AS t1
    FROM events
    WHERE event_name = 'landing_page_view'
      AND event_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY user_id
),
step2 AS (
    SELECT s1.user_id, MIN(e.event_time) AS t2
    FROM step1 s1
    JOIN events e ON s1.user_id = e.user_id
        AND e.event_name = 'signup_complete'
        AND e.event_time > s1.t1
        AND e.event_time < s1.t1 + INTERVAL '24 hours'
    GROUP BY s1.user_id
),
step3 AS (
    SELECT s2.user_id, MIN(e.event_time) AS t3
    FROM step2 s2
    JOIN events e ON s2.user_id = e.user_id
        AND e.event_name = 'onboarding_complete'
        AND e.event_time > s2.t2
        AND e.event_time < s2.t2 + INTERVAL '24 hours'
    GROUP BY s2.user_id
),
step4 AS (
    SELECT s3.user_id, MIN(e.event_time) AS t4
    FROM step3 s3
    JOIN events e ON s3.user_id = e.user_id
        AND e.event_name = 'first_value_action'
        AND e.event_time > s3.t3
        AND e.event_time < s3.t3 + INTERVAL '7 days'
    GROUP BY s3.user_id
)
SELECT
    'Landing Page' AS step, COUNT(*) AS users FROM step1 UNION ALL
SELECT 'Signup', COUNT(*) FROM step2 UNION ALL
SELECT 'Onboarding', COUNT(*) FROM step3 UNION ALL
SELECT 'Activation', COUNT(*) FROM step4;
```

### Time-to-Convert Analysis

```sql
-- How long does each step take? Slow steps → friction → drop-off
WITH funnel_times AS (
    SELECT
        user_id,
        MIN(CASE WHEN event_name = 'signup_start' THEN event_time END) AS signup_start,
        MIN(CASE WHEN event_name = 'signup_complete' THEN event_time END) AS signup_complete,
        MIN(CASE WHEN event_name = 'onboarding_done' THEN event_time END) AS onboarding_done
    FROM events
    GROUP BY user_id
)
SELECT
    PERCENTILE_CONT(0.50) WITHIN GROUP (
        ORDER BY EXTRACT(EPOCH FROM signup_complete - signup_start)
    ) AS median_signup_seconds,
    PERCENTILE_CONT(0.90) WITHIN GROUP (
        ORDER BY EXTRACT(EPOCH FROM signup_complete - signup_start)
    ) AS p90_signup_seconds,
    PERCENTILE_CONT(0.50) WITHIN GROUP (
        ORDER BY EXTRACT(EPOCH FROM onboarding_done - signup_complete) / 3600.0
    ) AS median_onboarding_hours
FROM funnel_times
WHERE signup_complete IS NOT NULL;
```

## Python Funnel Visualization

```python
import pandas as pd
import matplotlib.pyplot as plt


def plot_funnel(steps: list[str], users: list[int], title: str = "Conversion Funnel") -> None:
    """Plot a horizontal funnel chart with conversion rates.
    
    Args:
        steps: List of step names in order
        users: List of user counts at each step
        title: Chart title
    """
    fig, ax = plt.subplots(figsize=(10, 6))
    
    # Compute metrics
    overall_conv = [100.0 * u / users[0] for u in users]
    step_conv = [100.0] + [100.0 * users[i] / users[i-1] for i in range(1, len(users))]
    
    # Horizontal bar chart
    y_pos = range(len(steps) - 1, -1, -1)
    colors = plt.cm.Blues([(0.3 + 0.7 * c / 100) for c in overall_conv])
    
    bars = ax.barh(y_pos, users, color=colors, edgecolor="white", height=0.7)
    
    # Annotate bars
    for i, (bar, step, count, oc, sc) in enumerate(
        zip(bars, steps, users, overall_conv, step_conv)
    ):
        label = f"{count:,}  ({oc:.1f}% overall"
        if i > 0:
            label += f", {sc:.1f}% from prev"
        label += ")"
        ax.text(bar.get_width() + max(users) * 0.01, bar.get_y() + bar.get_height() / 2,
                label, va="center", fontsize=10)
    
    ax.set_yticks(y_pos)
    ax.set_yticklabels(steps, fontsize=12)
    ax.set_xlabel("Users")
    ax.set_title(title)
    ax.set_xlim(0, max(users) * 1.5)
    plt.tight_layout()
    plt.show()


def funnel_comparison(segments: dict[str, list[int]], steps: list[str]) -> pd.DataFrame:
    """Compare funnel conversion across segments.
    
    Args:
        segments: Dict mapping segment name to list of user counts per step
        steps: List of step names
    
    Returns:
        DataFrame with step-to-step and overall conversion per segment
    """
    rows = []
    for segment, users in segments.items():
        for i, step in enumerate(steps):
            rows.append({
                "segment": segment,
                "step": step,
                "users": users[i],
                "overall_pct": round(100.0 * users[i] / users[0], 1),
                "step_pct": round(100.0 * users[i] / users[i-1], 1) if i > 0 else 100.0,
            })
    return pd.DataFrame(rows)
```

## Diagnosing Drop-offs

### The Drop-off Investigation Checklist

When a funnel step has an unexpectedly high drop-off:

1. **Is it real?** Check tracking — are events firing correctly for all platforms?
2. **Is it new?** Compare to the previous period — sudden vs. gradual change
3. **Segment it**: Which user segments have the worst conversion? (platform, country, device, acquisition channel)
4. **Time-to-convert**: Are users who drop off spending too long on the step (friction) or leaving immediately (unclear value)?
5. **Error rates**: Are there technical errors (crashes, timeouts, API errors) at this step?
6. **User feedback**: Qualitative data from support tickets, session recordings, surveys

### Common Drop-off Causes by Step

| Funnel Stage | Common Causes |
|-------------|--------------|
| Landing → Signup | Unclear value prop, too many fields, no social login |
| Signup → Onboarding | Onboarding too long, unclear next steps, permission walls |
| Onboarding → Activation | User doesn't reach "aha moment", no personalization |
| Activation → Conversion | Pricing friction, no urgency, unclear upgrade path |

## Interview Approach

For a funnel interview question ("Instagram stories views are down — diagnose this"):

1. **Clarify the metric**: How is "views" defined? Unique viewers or total views? What time period?
2. **Decompose**: Views = DAU × % who view stories × stories viewed per viewer. Which component changed?
3. **Build the funnel**: App open → feed view → story tray visible → first story tap → subsequent stories → story completion
4. **Segment**: Platform, country, new vs. existing users, story creators vs. consumers
5. **Check supply**: Fewer stories created → fewer to view (supply-side issue)
6. **Check demand**: Same stories available but fewer taps (demand-side issue)
7. **Propose hypothesis and test**: "We think the story tray redesign reduced discoverability. We can A/B test reverting the position."

## Best Practices

- **Define conversion window**: Always specify the time window for conversion — "within 7 days" vs. "ever" gives very different numbers
- **Use consistent entry criteria**: Filter to users who actually had a chance to convert (e.g., only count users who saw the paywall)
- **Watch for survivorship bias**: Completing step N means the user survived until step N — don't attribute step N's drop-off to something that happened at step 1
- **Track both leads and rates**: A 50% conversion rate with 100 users is less important than a 40% rate with 10,000 users
- **Iterate on the biggest lever**: Improve the step with the biggest absolute user loss first, not necessarily the worst percentage

## Additional Resources

- Amplitude's Guide to Funnel Analysis
- "The Lean Startup" by Eric Ries — Build-Measure-Learn funnel thinking
- Google Analytics Funnel Exploration documentation
- Reforge: Conversion Rate Optimization
- Supplementary: Funnel Analysis Deep Dive (Data-Science-Analytical-Handbook)
