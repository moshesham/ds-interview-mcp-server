# Product Metrics Frameworks

## Overview
Product metrics frameworks provide structured approaches to defining, measuring, and
interpreting the health and success of digital products. Mastering these frameworks is
essential for any product data analyst.

## AARRR (Pirate Metrics) Framework
The AARRR funnel tracks the full user lifecycle:

- **Acquisition** — How users discover your product (channels, CAC, install rate).
- **Activation** — First positive experience (signup completion, onboarding finish, aha moment).
- **Retention** — Users returning over time (D1/D7/D30 retention, DAU/MAU ratio).
- **Referral** — Users inviting others (viral coefficient, invite rate, NPS).
- **Revenue** — Monetization (ARPU, LTV, conversion rate, subscription revenue).

### When to Use
Use AARRR when analyzing the end-to-end user journey, diagnosing which stage leaks
the most users, or presenting funnel health to stakeholders.

### Example Application
For an e-commerce app:
| Stage       | Metric                   | Example Target |
|-------------|--------------------------|----------------|
| Acquisition | Cost per Install         | < $2.00        |
| Activation  | First Purchase within 7d | > 15%          |
| Retention   | 30-day repurchase rate   | > 25%          |
| Referral    | Referral invites sent    | > 0.3 per user |
| Revenue     | LTV / CAC ratio          | > 3.0          |

## HEART Framework (Google)
Developed at Google, HEART focuses on user experience:

- **Happiness** — User satisfaction (NPS, CSAT, app store rating).
- **Engagement** — Depth of interaction (sessions per week, actions per session, time spent).
- **Adoption** — New users of a feature or product (% of users using new feature).
- **Retention** — Continued use over time (weekly/monthly active user retention).
- **Task Success** — Efficiency and effectiveness (task completion rate, error rate, time-on-task).

### Goals-Signals-Metrics (GSM) Process
For each HEART dimension:
1. Define the **Goal** — what does success look like?
2. Identify **Signals** — observable user behaviors indicating progress.
3. Choose **Metrics** — quantifiable measures derived from signals.

## North Star Metric
A single metric that best captures the core value your product delivers:

| Product Type     | Example North Star             |
|------------------|--------------------------------|
| Social Media     | DAU or Time Spent              |
| E-commerce       | Weekly Purchases per User      |
| SaaS             | Weekly Active Teams            |
| Streaming        | Hours Watched per Subscriber   |
| Fintech          | Monthly Transactions per User  |

### Input Metrics
North Star metrics are decomposed into input metrics (drivers):
```
North Star = f(input_1, input_2, ..., input_n)

Example: Weekly Purchases = New Buyers + Repeat Buyers
  - New Buyers = Traffic * Signup Rate * First Purchase Rate
  - Repeat Buyers = Active Users * Repurchase Rate
```

## Counter Metrics & Guardrails
Always pair success metrics with guardrail metrics to catch unintended harm:
- Revenue increase but NPS drops → poor UX or spam
- Click-through improves but bounce rate spikes → clickbait
- DAU rises but session duration drops → superficial engagement

## Practice Questions
1. Define a North Star metric for a food delivery app and decompose it into 3-4 input metrics.
2. Apply the HEART framework to evaluate a new "Stories" feature on a professional networking app.
3. An A/B test shows a 5% lift in click-through rate. What guardrail metrics should you check before shipping?
