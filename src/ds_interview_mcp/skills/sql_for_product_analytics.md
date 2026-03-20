# SQL for Product Analytics

## Overview
SQL is the primary tool for product data analysts. This skill covers the patterns most
frequently tested in interviews and used in day-to-day product analytics work.

## Essential SQL Patterns

### 1. Window Functions
Window functions are the most tested SQL concept in analytics interviews.

```sql
-- Running total of revenue by user
SELECT
    user_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date) AS cumulative_revenue
FROM orders;

-- Rank users by engagement within each country
SELECT
    user_id,
    country,
    engagement_score,
    RANK() OVER (PARTITION BY country ORDER BY engagement_score DESC) AS engagement_rank,
    NTILE(10) OVER (PARTITION BY country ORDER BY engagement_score DESC) AS decile
FROM user_metrics;

-- Compare each day's DAU to the previous day
SELECT
    date,
    dau,
    LAG(dau) OVER (ORDER BY date) AS prev_day_dau,
    ROUND((dau - LAG(dau) OVER (ORDER BY date))::NUMERIC
          / LAG(dau) OVER (ORDER BY date) * 100, 2) AS pct_change
FROM daily_active_users;
```

### 2. CTEs (Common Table Expressions)
Use CTEs to break complex queries into readable stages:

```sql
WITH daily_metrics AS (
    SELECT
        DATE_TRUNC('day', event_time) AS date,
        COUNT(DISTINCT user_id) AS dau,
        COUNT(*) AS total_events
    FROM events
    WHERE event_time >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY 1
),
rolling_avg AS (
    SELECT
        date,
        dau,
        AVG(dau) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS dau_7d_avg
    FROM daily_metrics
)
SELECT * FROM rolling_avg ORDER BY date;
```

### 3. Funnel Analysis
```sql
WITH funnel AS (
    SELECT
        user_id,
        MAX(CASE WHEN event = 'page_view' THEN 1 ELSE 0 END) AS step_1_view,
        MAX(CASE WHEN event = 'add_to_cart' THEN 1 ELSE 0 END) AS step_2_cart,
        MAX(CASE WHEN event = 'checkout_start' THEN 1 ELSE 0 END) AS step_3_checkout,
        MAX(CASE WHEN event = 'purchase' THEN 1 ELSE 0 END) AS step_4_purchase
    FROM events
    WHERE event_date = '2025-01-15'
    GROUP BY user_id
)
SELECT
    COUNT(*) AS total_users,
    SUM(step_1_view) AS viewed,
    SUM(step_2_cart) AS added_to_cart,
    SUM(step_3_checkout) AS started_checkout,
    SUM(step_4_purchase) AS purchased,
    ROUND(SUM(step_2_cart)::NUMERIC / NULLIF(SUM(step_1_view), 0) * 100, 1) AS view_to_cart_pct,
    ROUND(SUM(step_4_purchase)::NUMERIC / NULLIF(SUM(step_1_view), 0) * 100, 1) AS overall_cvr_pct
FROM funnel;
```

### 4. Sessionization
```sql
WITH events_with_gap AS (
    SELECT
        user_id,
        event_time,
        LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time,
        CASE
            WHEN event_time - LAG(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time
            ) > INTERVAL '30 minutes' THEN 1
            ELSE 0
        END AS new_session_flag
    FROM events
),
sessions AS (
    SELECT
        user_id,
        event_time,
        SUM(new_session_flag) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS session_id
    FROM events_with_gap
)
SELECT
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS events_in_session,
    MAX(event_time) - MIN(event_time) AS session_duration
FROM sessions
GROUP BY user_id, session_id;
```

### 5. Growth Accounting
```sql
WITH monthly_active AS (
    SELECT DISTINCT
        DATE_TRUNC('month', activity_date) AS month,
        user_id
    FROM user_activity
),
classified AS (
    SELECT
        curr.month,
        curr.user_id,
        CASE
            WHEN prev.user_id IS NOT NULL THEN 'retained'
            WHEN first_month.month = curr.month THEN 'new'
            WHEN resurrected.user_id IS NOT NULL THEN 'resurrected'
        END AS user_type
    FROM monthly_active curr
    LEFT JOIN monthly_active prev
        ON curr.user_id = prev.user_id
        AND curr.month = prev.month + INTERVAL '1 month'
    LEFT JOIN (
        SELECT user_id, MIN(month) AS month FROM monthly_active GROUP BY user_id
    ) first_month ON curr.user_id = first_month.user_id
    LEFT JOIN monthly_active resurrected
        ON curr.user_id = resurrected.user_id
        AND resurrected.month < curr.month - INTERVAL '1 month'
)
SELECT
    month,
    COUNT(CASE WHEN user_type = 'new' THEN 1 END) AS new_users,
    COUNT(CASE WHEN user_type = 'retained' THEN 1 END) AS retained_users,
    COUNT(CASE WHEN user_type = 'resurrected' THEN 1 END) AS resurrected_users,
    COUNT(*) AS total_mau
FROM classified
GROUP BY month
ORDER BY month;
```

## Query Optimization Tips
- Use indexes on columns in WHERE and JOIN conditions.
- Avoid `SELECT *` — specify only needed columns.
- Use `EXISTS` instead of `IN` for large subqueries.
- Pre-aggregate in CTEs before joining.
- Avoid functions on indexed columns in WHERE clauses.

## Interview SQL Strategies
1. **Clarify first** — ask about table schema, data volume, expected output.
2. **Start with CTEs** — build incrementally, name each step clearly.
3. **Talk through your logic** — explain before writing.
4. **Check edge cases** — NULLs, duplicates, empty sets.
5. **Optimize last** — get correctness first, then discuss performance.

## Practice Questions
1. Write a query to find the top 3 products by revenue for each category, including ties.
2. Calculate 7-day rolling average DAU and flag days with >10% drop from the rolling average.
3. Build a conversion funnel from 'visit' -> 'signup' -> 'first_purchase' with step-over-step drop-off rates.
4. Identify users who were active in January but not active in February (churned users).
