# Funnel Analysis & Conversion Optimization

## Overview
Funnel analysis tracks user progression through a sequence of steps toward a desired
outcome (purchase, signup, activation). Identifying where users drop off enables
targeted optimization.

## Common Product Funnels

### E-Commerce Funnel
```
Landing Page → Product View → Add to Cart → Checkout → Purchase
```

### SaaS Signup Funnel
```
Website Visit → Start Signup → Complete Signup → Onboarding → Activation (Aha Moment)
```

### Social App Engagement Funnel
```
App Open → Content View → Interaction (Like/Comment) → Content Creation → Share
```

## Funnel Metrics

| Metric                 | Formula                                     |
|------------------------|---------------------------------------------|
| Step Conversion Rate   | Users at Step N / Users at Step N-1         |
| Overall Conversion     | Users at Final Step / Users at First Step   |
| Drop-off Rate          | 1 - Step Conversion Rate                    |
| Time Between Steps     | Median time from Step N-1 to Step N         |

## SQL: Building a Funnel

### Sequential Funnel (Strict Order)
```sql
WITH step_events AS (
    SELECT
        user_id,
        MIN(CASE WHEN event = 'page_view' THEN event_time END) AS step1_time,
        MIN(CASE WHEN event = 'add_to_cart' THEN event_time END) AS step2_time,
        MIN(CASE WHEN event = 'checkout' THEN event_time END) AS step3_time,
        MIN(CASE WHEN event = 'purchase' THEN event_time END) AS step4_time
    FROM events
    WHERE event_date BETWEEN '2025-01-01' AND '2025-01-31'
    GROUP BY user_id
),
ordered_funnel AS (
    SELECT
        user_id,
        CASE WHEN step1_time IS NOT NULL THEN 1 ELSE 0 END AS reached_step1,
        CASE WHEN step2_time > step1_time THEN 1 ELSE 0 END AS reached_step2,
        CASE WHEN step3_time > step2_time THEN 1 ELSE 0 END AS reached_step3,
        CASE WHEN step4_time > step3_time THEN 1 ELSE 0 END AS reached_step4
    FROM step_events
)
SELECT
    SUM(reached_step1) AS step1_users,
    SUM(reached_step2) AS step2_users,
    SUM(reached_step3) AS step3_users,
    SUM(reached_step4) AS step4_users,
    ROUND(SUM(reached_step2)::NUMERIC / NULLIF(SUM(reached_step1), 0) * 100, 1) AS step1_to_2_pct,
    ROUND(SUM(reached_step3)::NUMERIC / NULLIF(SUM(reached_step2), 0) * 100, 1) AS step2_to_3_pct,
    ROUND(SUM(reached_step4)::NUMERIC / NULLIF(SUM(reached_step3), 0) * 100, 1) AS step3_to_4_pct,
    ROUND(SUM(reached_step4)::NUMERIC / NULLIF(SUM(reached_step1), 0) * 100, 1) AS overall_cvr
FROM ordered_funnel;
```

### Segmented Funnel
```sql
-- Break funnel by platform to find where drop-off is device-specific
SELECT
    u.platform,
    SUM(reached_step1) AS step1,
    SUM(reached_step4) AS step4,
    ROUND(SUM(reached_step4)::NUMERIC / NULLIF(SUM(reached_step1), 0) * 100, 1) AS cvr_pct
FROM ordered_funnel f
JOIN users u ON f.user_id = u.user_id
GROUP BY u.platform;
```

## Diagnostic Framework: Why Are Users Dropping Off?

### Step 1: Quantify the Drop-Off
- Which step has the highest absolute drop-off?
- Which step has the highest relative drop-off?

### Step 2: Segment the Drop-Off
- By platform (iOS vs Android vs Web)
- By user type (new vs returning)
- By geography / language
- By traffic source / acquisition channel

### Step 3: Investigate Causes
| Possible Cause       | How to Diagnose                          |
|----------------------|------------------------------------------|
| UX friction          | Session recordings, time-on-step analysis|
| Technical errors     | Error logs, crash rates by step          |
| Pricing shock        | Time spent on pricing page, exit rate    |
| Missing trust signals| Compare CVR with/without reviews/badges  |
| Form complexity      | Field-level abandonment rates            |

### Step 4: Prioritize & Test
Use ICE framework (Impact × Confidence × Ease) to prioritize experiments.
A/B test each change to measure causal lift.

## Python: Funnel Visualization
```python
import plotly.express as px

stages = ['Page View', 'Add to Cart', 'Checkout', 'Purchase']
values = [10000, 3500, 1800, 900]

fig = px.funnel(
    x=values, y=stages,
    title='E-Commerce Conversion Funnel (Jan 2025)'
)
fig.show()
```

## Benchmarks
| Funnel Step             | Typical Rate (E-Commerce) |
|-------------------------|---------------------------|
| View → Add to Cart      | 8-15%                     |
| Add to Cart → Checkout  | 30-50%                    |
| Checkout → Purchase     | 50-70%                    |
| Overall Visit → Purchase| 2-5%                      |

## Practice Questions
1. Build a checkout funnel grouped by device type. Where is the biggest drop-off for mobile?
2. The product team redesigned the signup flow. How would you measure if it improved conversion?
3. A PM says "only 2% of visitors convert." Walk through how you'd diagnose and prioritize.
4. Explain the difference between a strict-order funnel and an any-order funnel. When would each be appropriate?
