---
name: sql-for-product-analytics
description: Write advanced SQL for product analytics including window functions, CTEs, funnel queries, sessionization, cohort analysis, and growth accounting. Covers query optimization, interview patterns, and real-world analytical SQL at scale. For data analyst interviews, dashboard building, and ad-hoc product investigations.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# SQL for Product Analytics

## Overview

SQL is the primary language for product analytics work. Beyond basic SELECT/JOIN/GROUP BY, product analysts need mastery of window functions, recursive CTEs, sessionization logic, and funnel queries. These patterns appear in every product analytics interview and in daily work building dashboards and investigating metric movements.

**Key patterns covered:**
- **Window functions**: ROW_NUMBER, LAG/LEAD, running totals, percentiles
- **CTEs & recursive queries**: Modular query building, hierarchical data
- **Funnel analysis**: Ordered event sequences, drop-off calculations
- **Sessionization**: Grouping events into sessions by inactivity gap
- **Growth accounting**: Quick ratio, new/resurrected/retained/churned framework

## When to Use This Skill

Use this skill when:

- **SQL interviews**: Technical screens at Meta, Google, Airbnb, Netflix, Spotify
- **Building dashboards**: Writing production queries for metrics dashboards
- **Ad-hoc investigations**: Diagnosing metric drops, analyzing user behavior
- **Funnel analysis**: Measuring conversion through multi-step flows
- **Cohort queries**: Tracking user groups over time
- **Performance optimization**: Making slow analytical queries fast

## Window Functions

### Essential Window Functions Reference

```sql
-- ROW_NUMBER: Rank items within groups (dedup, top-N)
SELECT
    user_id,
    event_time,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_time DESC) AS rn
FROM events;

-- LAG/LEAD: Access previous/next row values
SELECT
    user_id,
    event_time,
    LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event,
    event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS time_since_last
FROM events;

-- Running totals and moving averages
SELECT
    event_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY event_date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
    AVG(daily_revenue) OVER (ORDER BY event_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS revenue_7d_avg
FROM daily_metrics;

-- NTILE: Divide users into equal-sized buckets
SELECT
    user_id,
    total_spend,
    NTILE(10) OVER (ORDER BY total_spend DESC) AS spend_decile
FROM user_summary;

-- PERCENT_RANK: Percentile position
SELECT
    user_id,
    session_count,
    PERCENT_RANK() OVER (ORDER BY session_count) AS percentile
FROM user_engagement;
```

## CTE Patterns

### Modular Query Building

```sql
-- Pattern: Break complex analysis into readable steps
WITH daily_active AS (
    SELECT
        DATE(event_time) AS event_date,
        COUNT(DISTINCT user_id) AS dau
    FROM events
    WHERE event_name = 'app_open'
    GROUP BY 1
),
weekly_avg AS (
    SELECT
        event_date,
        dau,
        AVG(dau) OVER (ORDER BY event_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS dau_7d_avg
    FROM daily_active
)
SELECT
    event_date,
    dau,
    dau_7d_avg,
    ROUND(100.0 * (dau - dau_7d_avg) / NULLIF(dau_7d_avg, 0), 2) AS pct_deviation
FROM weekly_avg
ORDER BY event_date DESC;
```

## Funnel Analysis

### Strict Sequential Funnel

```sql
-- E-commerce funnel: homepage → product_view → add_to_cart → checkout → purchase
WITH step_times AS (
    SELECT
        user_id,
        MIN(CASE WHEN event_name = 'homepage_view' THEN event_time END) AS step1_time,
        MIN(CASE WHEN event_name = 'product_view' THEN event_time END) AS step2_time,
        MIN(CASE WHEN event_name = 'add_to_cart' THEN event_time END) AS step3_time,
        MIN(CASE WHEN event_name = 'checkout_start' THEN event_time END) AS step4_time,
        MIN(CASE WHEN event_name = 'purchase' THEN event_time END) AS step5_time
    FROM events
    WHERE event_date = CURRENT_DATE - INTERVAL '7 days'
    GROUP BY user_id
),
sequential_funnel AS (
    SELECT
        user_id,
        1 AS reached_step1,
        CASE WHEN step2_time > step1_time THEN 1 ELSE 0 END AS reached_step2,
        CASE WHEN step3_time > step2_time AND step2_time > step1_time THEN 1 ELSE 0 END AS reached_step3,
        CASE WHEN step4_time > step3_time AND step3_time > step2_time AND step2_time > step1_time THEN 1 ELSE 0 END AS reached_step4,
        CASE WHEN step5_time > step4_time AND step4_time > step3_time AND step3_time > step2_time AND step2_time > step1_time THEN 1 ELSE 0 END AS reached_step5
    FROM step_times
    WHERE step1_time IS NOT NULL
)
SELECT
    'Homepage'     AS step, SUM(reached_step1) AS users, 100.0 AS conversion_pct UNION ALL
SELECT 'Product View', SUM(reached_step2), 100.0 * SUM(reached_step2) / NULLIF(SUM(reached_step1), 0) UNION ALL
SELECT 'Add to Cart',  SUM(reached_step3), 100.0 * SUM(reached_step3) / NULLIF(SUM(reached_step1), 0) UNION ALL
SELECT 'Checkout',     SUM(reached_step4), 100.0 * SUM(reached_step4) / NULLIF(SUM(reached_step1), 0) UNION ALL
SELECT 'Purchase',     SUM(reached_step5), 100.0 * SUM(reached_step5) / NULLIF(SUM(reached_step1), 0)
FROM sequential_funnel;
```

### Segmented Funnel

```sql
-- Break funnel down by platform to identify where the biggest drop-off occurs
WITH funnel AS (
    -- (same CTE as above but adding platform column)
    SELECT user_id, platform, reached_step1, reached_step2, reached_step3
    FROM sequential_funnel
    JOIN users USING (user_id)
)
SELECT
    platform,
    SUM(reached_step1) AS step1_users,
    SUM(reached_step2) AS step2_users,
    ROUND(100.0 * SUM(reached_step2) / NULLIF(SUM(reached_step1), 0), 1) AS step1_to_2_pct,
    SUM(reached_step3) AS step3_users,
    ROUND(100.0 * SUM(reached_step3) / NULLIF(SUM(reached_step2), 0), 1) AS step2_to_3_pct
FROM funnel
GROUP BY platform
ORDER BY step1_users DESC;
```

## Sessionization

Grouping raw events into sessions based on an inactivity threshold (typically 30 minutes):

```sql
WITH events_with_gap AS (
    SELECT
        user_id,
        event_time,
        event_name,
        LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time,
        EXTRACT(EPOCH FROM (event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time))) / 60.0 AS gap_minutes
    FROM events
),
session_starts AS (
    SELECT
        *,
        CASE WHEN gap_minutes IS NULL OR gap_minutes > 30 THEN 1 ELSE 0 END AS is_new_session
    FROM events_with_gap
),
sessions AS (
    SELECT
        *,
        SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
    FROM session_starts
)
SELECT
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS events_in_session,
    EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))) / 60.0 AS session_duration_min
FROM sessions
GROUP BY user_id, session_id;
```

## Growth Accounting

Decompose user base changes into new, resurrected, retained, and churned:

```sql
WITH monthly_users AS (
    SELECT DISTINCT
        DATE_TRUNC('month', event_date) AS month,
        user_id
    FROM events
),
user_status AS (
    SELECT
        c.month,
        c.user_id,
        CASE
            WHEN p.user_id IS NULL AND pp.user_id IS NULL THEN 'new'
            WHEN p.user_id IS NULL AND pp.user_id IS NOT NULL THEN 'resurrected'
            ELSE 'retained'
        END AS status
    FROM monthly_users c
    LEFT JOIN monthly_users p ON c.user_id = p.user_id AND c.month = p.month + INTERVAL '1 month'
    LEFT JOIN monthly_users pp ON c.user_id = pp.user_id AND pp.month < c.month - INTERVAL '1 month'
),
churned AS (
    SELECT
        p.month + INTERVAL '1 month' AS month,
        p.user_id,
        'churned' AS status
    FROM monthly_users p
    LEFT JOIN monthly_users c ON p.user_id = c.user_id AND c.month = p.month + INTERVAL '1 month'
    WHERE c.user_id IS NULL
)
SELECT
    month,
    status,
    COUNT(*) AS users
FROM (
    SELECT month, user_id, status FROM user_status
    UNION ALL
    SELECT month, user_id, status FROM churned
) combined
GROUP BY month, status
ORDER BY month, status;

-- Quick Ratio = (new + resurrected) / churned
-- Quick Ratio > 1  →  growing user base
-- Quick Ratio > 4  →  strong growth
```

## Query Optimization Tips

| Technique | When to Use | Impact |
|-----------|------------|--------|
| Filter early with WHERE | Always | Reduces data scanned |
| Use approximate functions (APPROX_COUNT_DISTINCT) | Large-scale counts where exact precision isn't needed | 10-100x faster |
| Avoid SELECT * | Always | Reduces I/O |
| Partition pruning | Date-partitioned tables | Huge I/O savings |
| Materialize intermediate CTEs | When the same CTE is referenced multiple times | Prevents re-computation |
| Use EXISTS instead of IN for subqueries | Large subquery result sets | Better query plan |

## Interview Strategy

1. **Clarify**: Ask about table schema, time range, edge cases, what "active" means
2. **Outline**: State your approach before writing SQL ("I'll use a CTE to get daily active users, then a window function for 7-day rolling average")
3. **Write incrementally**: Build from inner query outward, test each CTE mentally
4. **Handle edge cases**: NULLs, division by zero (use NULLIF), users with no events
5. **Optimize last**: Get correctness first, then discuss optimization

## Best Practices

- **Always use CTEs** to make complex queries readable and debuggable
- **Avoid correlated subqueries** — rewrite as JOINs or window functions
- **Use COALESCE and NULLIF** to handle NULLs and division by zero explicitly
- **Date filter every query** — unbounded scans on event tables are expensive
- **Name your columns clearly** — `signup_date` not `d`, `daily_active_users` not `cnt`
- **Comment non-obvious logic** — especially business rules embedded in CASE statements

## Additional Resources

- Mode Analytics SQL Tutorial
- StrataScratch — SQL interview questions from FAANG companies
- LeetCode Database Section
- "SQL for Data Scientists" by Renee Teate
- Supplementary: Advanced SQL Patterns & Techniques (Data-Science-Analytical-Handbook)
