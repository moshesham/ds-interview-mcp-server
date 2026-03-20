# Churn Analytics & Customer Retention

## Overview
Churn analysis identifies why and when users leave a product, enabling proactive
retention strategies. This is a critical skill for product data analysts, especially
at subscription and SaaS companies.

## Defining Churn

### Contractual vs Non-Contractual Churn
| Type            | Examples                  | How to Measure                    |
|-----------------|---------------------------|-----------------------------------|
| Contractual     | SaaS subscription cancel  | Explicit cancellation event       |
| Non-contractual | E-commerce, social media  | Inactivity threshold (e.g., no activity in 28 days) |

### Voluntary vs Involuntary Churn
- **Voluntary** — user actively decides to leave (dissatisfaction, competitor, no longer needed).
- **Involuntary** — payment failure, expired credit card, account lockout.

## Churn Rate Calculation

### Simple Churn Rate
```
Churn Rate = (Users Lost During Period) / (Users at Start of Period) × 100
```

### Cohort-Based Churn
More accurate — tracks each cohort's survival independently:
```sql
SELECT
    cohort_month,
    month_number,
    1 - (active_users::FLOAT / cohort_size) AS cumulative_churn_rate
FROM cohort_retention;
```

### Revenue Churn (Net Revenue Retention)
```
Net Revenue Retention = (MRR_start + Expansion - Contraction - Churn) / MRR_start × 100
```
NRR > 100% means growth from existing customers alone (expansion > churn).

## Identifying Churn Drivers

### Data Sources
1. **Usage data** — Login frequency, feature usage, time in app.
2. **Support data** — Ticket volume, resolution time, complaint themes.
3. **Billing data** — Payment failures, downgrades, plan changes.
4. **Survey data** — Exit surveys, NPS, CSAT scores.
5. **Product data** — Feature adoption, error rates, load times.

### Analytical Approaches

#### Survival Analysis
Model time-to-churn using Kaplan-Meier curves or Cox proportional hazards:
```python
from lifelines import KaplanMeierFitter, CoxPHFitter

# Kaplan-Meier survival curve
kmf = KaplanMeierFitter()
kmf.fit(durations=df['tenure_days'], event_observed=df['churned'])
kmf.plot_survival_function()

# Cox model to find risk factors
cph = CoxPHFitter()
cph.fit(df[['tenure_days', 'churned', 'support_tickets',
            'login_frequency', 'feature_usage_score']], 
        duration_col='tenure_days', event_col='churned')
cph.print_summary()
```

#### Logistic Regression for Churn Prediction
```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, roc_auc_score

features = ['login_freq_30d', 'feature_count', 'support_tickets',
            'days_since_last_activity', 'plan_type_encoded']

model = LogisticRegression()
model.fit(X_train[features], y_train)

# Evaluate
y_pred_proba = model.predict_proba(X_test[features])[:, 1]
print(f"AUC-ROC: {roc_auc_score(y_test, y_pred_proba):.3f}")
print(classification_report(y_test, model.predict(X_test[features])))
```

#### RFM Segmentation for Retention
| Segment          | Recency | Frequency | Monetary | Action                       |
|------------------|---------|-----------|----------|------------------------------|
| Champions        | High    | High      | High     | Reward, upsell               |
| At Risk          | Low     | High      | High     | Re-engage urgently           |
| Hibernating      | Low     | Low       | Low      | Win-back campaign or accept  |
| New Customers    | High    | Low       | Low      | Onboarding and activation    |

## Retention Strategies by Churn Driver

| Churn Driver           | Strategy                                      |
|------------------------|-----------------------------------------------|
| Poor onboarding        | Guided tours, checklist, early value delivery  |
| Feature confusion      | In-app tooltips, usage nudges                  |
| Support frustration    | Faster resolution, proactive outreach          |
| Price sensitivity      | Downgrade path, annual discount                |
| Competitor switch      | Feature parity analysis, switching cost        |
| Involuntary (payments) | Retry logic, dunning emails, card update flow  |

## Key Benchmarks
| Industry     | Monthly Churn | Annual Churn | Good NRR   |
|--------------|---------------|--------------|------------|
| B2B SaaS     | 1-2%          | 10-20%       | 110-130%   |
| B2C SaaS     | 3-7%          | 30-60%       | 90-110%    |
| E-commerce   | Varies        | Varies       | N/A        |
| Consumer App | 5-10% monthly | 50-70%       | N/A        |

## Practice Questions
1. Calculate monthly churn rate and net revenue retention for a SaaS product given subscription data.
2. Build a churn prediction model. Which features would you prioritize and why?
3. A product manager asks: "Users are churning after the free trial." How would you investigate?
4. Design a retention strategy for a cohort with 60% D30 churn rate on a social app.
