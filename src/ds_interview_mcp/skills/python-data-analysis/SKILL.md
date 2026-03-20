---
name: python-data-analysis
description: Python patterns for product data analysis using pandas, numpy, and visualization libraries. Covers data wrangling, aggregation, time series, visualization best practices, and common anti-patterns. Practical reference for interview coding rounds and daily analytical work.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Python for Data Analysis

## Overview

Python is the primary programming language for product data analysis beyond SQL. While SQL handles data extraction, Python handles transformation, statistical analysis, visualization, and automation. This skill covers the pandas, numpy, and visualization patterns most commonly used in product analytics work and interviews.

**Key areas:**
- **Pandas data wrangling**: Merge, reshape, groupby, window operations
- **Time series operations**: Resampling, rolling windows, lag features
- **Aggregation patterns**: Cohort analysis, pivot tables, cross-tabulations
- **Visualization**: matplotlib, seaborn patterns for analytical charts
- **Performance**: Avoiding common anti-patterns that make analysis slow

## When to Use This Skill

Use this skill when:

- **Coding interviews**: Python data manipulation rounds at Meta, Google, Netflix, Spotify
- **Ad-hoc analysis**: Quick data exploration and hypothesis testing in Jupyter notebooks
- **Dashboard prototyping**: Building visualizations before converting to dashboarding tools
- **Data pipelines**: Transforming data for models or reports
- **Statistical analysis**: Running tests that aren't available in SQL

## Pandas Essentials

### Data Loading and Inspection

```python
import pandas as pd
import numpy as np

# Load and inspect
df = pd.read_csv("events.csv", parse_dates=["event_time"])

# Quick overview
df.shape              # (rows, columns)
df.dtypes             # Column types
df.describe()         # Summary statistics
df.isna().sum()       # Missing values per column
df.nunique()          # Unique values per column
df["event_name"].value_counts(normalize=True).head(10)  # Top events by frequency
```

### Filtering and Selection

```python
# Boolean indexing
active_users = df[
    (df["event_name"] == "app_open") &
    (df["event_time"] >= "2024-01-01") &
    (df["platform"].isin(["ios", "android"]))
]

# Query syntax (cleaner for complex filters)
active_users = df.query(
    "event_name == 'app_open' and "
    "event_time >= '2024-01-01' and "
    "platform in ['ios', 'android']"
)
```

### GroupBy Patterns

```python
# Basic aggregation
daily_metrics = (
    df.groupby(df["event_time"].dt.date)
    .agg(
        dau=("user_id", "nunique"),
        total_events=("event_id", "count"),
        total_revenue=("revenue", "sum"),
    )
    .reset_index()
)

# Multiple groupby with named aggregation
segment_metrics = (
    df.groupby(["platform", "country"])
    .agg(
        users=("user_id", "nunique"),
        avg_sessions=("session_id", "nunique"),
        median_revenue=("revenue", "median"),
    )
    .reset_index()
    .sort_values("users", ascending=False)
)

# Transform: add group-level stats back to each row
df["user_event_count"] = df.groupby("user_id")["event_id"].transform("count")
df["pct_of_user_events"] = df.groupby("user_id")["event_id"].transform(
    lambda x: 1 / len(x)
)
```

### Merge and Join

```python
# Left join (keep all users, add attributes)
enriched = events_df.merge(users_df, on="user_id", how="left")

# Multiple key join
combined = orders_df.merge(
    products_df,
    left_on=["product_id", "variant_id"],
    right_on=["id", "variant"],
    how="inner"
)

# Time-based join: find each user's first event
first_events = (
    events_df.sort_values("event_time")
    .groupby("user_id")
    .first()
    .reset_index()
)
users_with_first = users_df.merge(
    first_events[["user_id", "event_time", "event_name"]],
    on="user_id",
    how="left"
)
```

### Reshape: Pivot and Melt

```python
# Pivot: rows to columns (cohort retention table)
retention_pivot = retention_df.pivot(
    index="cohort_month",
    columns="period",
    values="retention_pct"
)

# Melt: columns to rows (wide to long)
long_df = pd.melt(
    wide_df,
    id_vars=["user_id", "date"],
    value_vars=["metric_a", "metric_b", "metric_c"],
    var_name="metric_name",
    value_name="metric_value"
)

# Cross-tabulation
pd.crosstab(
    df["platform"],
    df["subscription_tier"],
    values=df["revenue"],
    aggfunc="mean",
    margins=True,
)
```

## Time Series Operations

```python
# Resample to different frequency
daily = df.set_index("event_time").resample("D")["user_id"].nunique()
weekly = daily.resample("W").mean()

# Rolling windows
df["revenue_7d_avg"] = df["daily_revenue"].rolling(window=7).mean()
df["revenue_7d_std"] = df["daily_revenue"].rolling(window=7).std()

# Lag features
df["revenue_prev_day"] = df["daily_revenue"].shift(1)
df["revenue_wow_change"] = df["daily_revenue"] / df["daily_revenue"].shift(7) - 1

# Date parts for seasonality analysis
df["day_of_week"] = df["event_time"].dt.day_name()
df["hour"] = df["event_time"].dt.hour
df["is_weekend"] = df["event_time"].dt.dayofweek >= 5
```

## Visualization Patterns

### Distribution Analysis

```python
import matplotlib.pyplot as plt
import seaborn as sns

def plot_metric_distribution(
    data: pd.Series,
    title: str,
    log_scale: bool = False,
    percentiles: list[float] = [0.25, 0.50, 0.75, 0.90, 0.95, 0.99]
) -> None:
    """Plot distribution with key percentile markers."""
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))
    
    plot_data = np.log1p(data) if log_scale else data
    xlabel = f"log(1 + {title})" if log_scale else title
    
    # Histogram
    axes[0].hist(plot_data.dropna(), bins=50, edgecolor="white", alpha=0.7)
    for p in percentiles:
        val = plot_data.quantile(p)
        axes[0].axvline(val, color="red", linestyle="--", alpha=0.5)
        axes[0].text(val, axes[0].get_ylim()[1] * 0.9, f"P{int(p*100)}", fontsize=8, rotation=90)
    axes[0].set_xlabel(xlabel)
    axes[0].set_title(f"Distribution of {title}")
    
    # Box plot
    axes[1].boxplot(plot_data.dropna(), vert=False)
    axes[1].set_xlabel(xlabel)
    axes[1].set_title(f"Box Plot of {title}")
    
    plt.tight_layout()
    plt.show()
    
    # Print summary statistics
    print(f"\n{title} Summary:")
    for p in percentiles:
        print(f"  P{int(p*100):>2}: {data.quantile(p):,.2f}")
    print(f"  Mean: {data.mean():,.2f}, Std: {data.std():,.2f}")
```

### Time Series Chart

```python
def plot_metric_over_time(
    df: pd.DataFrame,
    date_col: str,
    value_col: str,
    title: str,
    rolling_window: int = 7,
) -> None:
    """Plot a metric over time with rolling average and annotation."""
    fig, ax = plt.subplots(figsize=(14, 6))
    
    ax.plot(df[date_col], df[value_col], alpha=0.3, label="Daily")
    ax.plot(
        df[date_col],
        df[value_col].rolling(rolling_window).mean(),
        linewidth=2,
        label=f"{rolling_window}-day avg",
    )
    
    ax.set_xlabel("Date")
    ax.set_ylabel(value_col)
    ax.set_title(title)
    ax.legend()
    ax.grid(True, alpha=0.3)
    plt.tight_layout()
    plt.show()
```

### Segment Comparison

```python
def plot_segment_comparison(
    df: pd.DataFrame,
    segment_col: str,
    value_col: str,
    title: str,
) -> None:
    """Compare a metric across segments with box plots and means."""
    fig, axes = plt.subplots(1, 2, figsize=(14, 6))
    
    # Box plot
    order = df.groupby(segment_col)[value_col].median().sort_values(ascending=False).index
    sns.boxplot(data=df, x=segment_col, y=value_col, order=order, ax=axes[0])
    axes[0].set_title(f"{title} — Distribution by {segment_col}")
    axes[0].tick_params(axis="x", rotation=45)
    
    # Bar chart of means with error bars
    summary = df.groupby(segment_col)[value_col].agg(["mean", "std", "count"]).loc[order]
    summary["se"] = summary["std"] / np.sqrt(summary["count"])
    axes[1].bar(range(len(summary)), summary["mean"], yerr=1.96 * summary["se"],
                capsize=4, alpha=0.7)
    axes[1].set_xticks(range(len(summary)))
    axes[1].set_xticklabels(summary.index, rotation=45)
    axes[1].set_title(f"{title} — Mean by {segment_col} (95% CI)")
    
    plt.tight_layout()
    plt.show()
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|----------------|
| Iterating rows with `for i, row in df.iterrows()` | 100-1000x slower than vectorized | Use vectorized operations or `.apply()` |
| Appending to DataFrame in a loop | Quadratic memory allocation | Build a list of dicts, then `pd.DataFrame(list)` |
| `df[df['col'] == val]` inside a loop | Repeated full-table scan | Use `.groupby()` or set index |
| Not using categorical dtype | High memory for repeated strings | `df['col'] = df['col'].astype('category')` |
| Chained indexing `df['a']['b']` | Unpredictable SettingWithCopy | Use `.loc[row, col]` |
| Loading full dataset when sampling works | Slow iteration, OOM | `df.sample(frac=0.1)` for exploration |

## Best Practices

- **Chain operations** using parentheses for readable pipelines: `(df.query(...).groupby(...).agg(...).reset_index())`
- **Name your aggregations** — `.agg(users=("user_id", "nunique"))` is clearer than `.agg({"user_id": "nunique"})`
- **Use `.assign()`** for adding computed columns in a chain without modifying the original DataFrame
- **Profile before optimizing** — use `%%timeit` and `df.memory_usage(deep=True)` to identify actual bottlenecks
- **Validate assumptions** — check for duplicates, nulls, and unexpected values before analysis
- **Reproducibility** — set random seeds, version your notebooks, document data sources

## Additional Resources

- "Python for Data Analysis" by Wes McKinney (pandas creator)
- pandas documentation — User Guide section
- Real Python — pandas tutorials
- Jake VanderPlas — "Python Data Science Handbook" (free online)
- Supplementary: Python for Data Science (Data-Science-Analytical-Handbook)
