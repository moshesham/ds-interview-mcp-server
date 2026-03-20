---
name: product-sense-frameworks
description: Frameworks and structured approaches for product sense interview questions. Covers CIRCLES, metric definition, root cause analysis, feature prioritization, and product case study approaches. Essential for PM and product analytics interviews at Meta, Google, Airbnb, and other top tech companies.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Product Sense Frameworks

## Overview

Product sense questions test your ability to think like a product analyst or PM — defining metrics, diagnosing problems, evaluating tradeoffs, and making data-driven recommendations. These questions are unstructured by design; frameworks give you structure without being rigid.

**Key frameworks:**
- **CIRCLES**: Comprehensive product design framework
- **Metric Trees**: Decomposing top-level metrics into components
- **Root Cause Analysis**: Systematic diagnosis of metric changes
- **RICE/ICE**: Feature prioritization scoring
- **Jobs-to-be-Done**: Understanding user motivation

## When to Use This Skill

Use this skill when:

- **Product sense interviews**: "How would you measure success for Instagram Reels?"
- **Metric investigation**: "DAU dropped 5% last week — what happened?"
- **Feature proposals**: "Should we add a stories feature to LinkedIn?"
- **Exec presentations**: Building the analytical narrative for product decisions
- **Goal setting**: Defining OKRs, north star metrics, and guardrail metrics

## CIRCLES Framework

For product design and improvement questions:

| Step | Action | Example (Instagram Reels) |
|------|--------|--------------------------|
| **C**omprehend | Clarify the situation, goals, constraints | "Is this about creation or consumption? All markets or specific?" |
| **I**dentify | Identify the customer segments | Creators, casual viewers, power viewers |
| **R**eport | Report customer needs | Creators: reach, monetization. Viewers: entertainment, discovery |
| **C**ut | Cut through prioritization | Focus on viewer engagement (largest segment, biggest growth lever) |
| **L**ist | List solutions | Improved recommendation algorithm, creator incentives, new formats |
| **E**valuate | Evaluate tradeoffs | Algorithm changes: high impact, low cost. New formats: high cost, uncertain impact |
| **S**ummarize | Summarize recommendation | "I'd prioritize recommendation algorithm because it's high impact, measurable via A/B test, and improves experience for all viewers" |

## Metric Trees

### Building a Metric Tree

Decompose any top-level metric into its components:

```
Revenue
├── Users × Revenue per User
│   ├── New Users × Conversion Rate × ARPU
│   └── Existing Users × Retention Rate × ARPU
│       ├── Subscription Revenue
│       │   ├── Subscribers × Price
│       │   └── Churn Rate
│       └── Transaction Revenue
│           ├── Transactions per User
│           └── Average Order Value
│               ├── Items per Order
│               └── Average Item Price
```

### Metric Definition Framework

For any metric, define these components:

```python
def define_metric(metric_name: str) -> dict:
    """Framework for rigorously defining a product metric."""
    return {
        "name": metric_name,
        "definition": "Precise mathematical definition",
        "numerator": "What's counted in the top",
        "denominator": "What's counted in the bottom (if ratio)",
        "unit": "Users, dollars, actions, percentage",
        "granularity": "Daily, weekly, monthly",
        "segments": ["Platform", "Country", "User type"],
        "data_source": "Which table/event",
        "edge_cases": [
            "How to handle bots/spam",
            "How to handle users with zero activity",
            "How to handle timezone differences",
        ],
        "guardrails": ["Metrics that must NOT degrade"],
        "limitations": ["What this metric doesn't capture"],
    }

# Example:
# define_metric("DAU") → {
#     "definition": "Count of unique users who performed at least one
#                    qualifying action in a calendar day (UTC)",
#     "numerator": "DISTINCT user_id with qualifying event",
#     "denominator": None (absolute count),
#     "qualifying_actions": ["app_open", "page_view", "any_action"],
#     ...
# }
```

## Root Cause Analysis

### Metric Investigation Framework

When a metric changes unexpectedly:

```
Step 1: Verify the data
  └─ Is the metric calculated correctly? Any logging/tracking issues?

Step 2: Quantify the change
  └─ How big is the change? Is it within normal variance?
  └─ When exactly did it start?

Step 3: Decompose the metric
  └─ metric = component_a × component_b × component_c
  └─ Which component changed?

Step 4: Segment analysis
  └─ Cut by: platform, country, user segment, new vs. existing
  └─ Is it concentrated in one segment or broad-based?

Step 5: Check for external factors
  └─ Product changes / deployments
  └─ Marketing campaigns starting/ending
  └─ Competitor actions
  └─ Seasonality / holidays
  └─ iOS/Android OS updates

Step 6: Correlate with other metrics
  └─ What other metrics moved at the same time?
  └─ Do the correlated changes tell a consistent story?

Step 7: Hypothesize and validate
  └─ Form 2-3 hypotheses that explain the data
  └─ What additional data would confirm/reject each?
```

### Interview Example: "Facebook likes are down 10%"

```
1. Verify: Is the metric correctly computed? Any tracking changes?
2. Decompose: Likes = DAU × % who like × likes per liker
   - DAU stable? → If down, it's an engagement issue
   - % who like down? → Content quality or UI change
   - Likes per liker down? → Feed algorithm or content mix
3. Segment:
   - By country: Did it drop everywhere or just one region?
   - By platform: iOS vs. Android vs. web
   - By content type: Photos, videos, text posts, shared links
   - By user tenure: New users vs. long-time users
4. Timeline: Did it coincide with a product release?
5. Hypotheses:
   - H1: Recent feed algorithm change reduced like-worthy content visibility
   - H2: New double-tap UI caused accidental likes to decrease
   - H3: Shift in content mix toward video (which gets fewer likes but more comments)
6. Validate: Check each hypothesis against segmented data
```

## Feature Prioritization

### RICE Framework

| Factor | Definition | Scale |
|--------|-----------|-------|
| **R**each | How many users affected per quarter | Exact number estimate |
| **I**mpact | Effect on each user | 3=massive, 2=high, 1=medium, 0.5=low, 0.25=minimal |
| **C**onfidence | How sure are you? | 100%=high, 80%=medium, 50%=low |
| **E**ffort | Person-months to build | Exact estimate |

**RICE Score = (Reach × Impact × Confidence) / Effort**

### Prioritization Decision Framework

```python
def prioritize_features(features: list[dict]) -> list[dict]:
    """Score and rank features using RICE framework.
    
    Each feature dict should have: name, reach, impact, confidence, effort
    """
    for f in features:
        f["rice_score"] = (f["reach"] * f["impact"] * f["confidence"]) / f["effort"]
    
    return sorted(features, key=lambda x: x["rice_score"], reverse=True)

# Example usage:
features = [
    {"name": "Push notification optimization", "reach": 50000, "impact": 1, "confidence": 0.8, "effort": 2},
    {"name": "Onboarding redesign", "reach": 20000, "impact": 2, "confidence": 0.6, "effort": 4},
    {"name": "Search improvements", "reach": 30000, "impact": 1, "confidence": 0.9, "effort": 3},
]
# prioritize_features(features) → ranked by RICE score
```

## Common Interview Question Types

### 1. "How would you measure X?"

**Structure**: Define the goal → Choose primary metric → Add supporting metrics → Add guardrail metrics → Discuss tradeoffs

### 2. "Metric X dropped — what happened?"

**Structure**: Verify → Decompose → Segment → Timeline → External factors → Hypothesize → Validate

### 3. "Should we build feature X?"

**Structure**: Who is it for → What problem does it solve → How would we measure success → What are the risks → Prioritize against alternatives

### 4. "Design a metric for X"

**Structure**: Define precisely → State numerator/denominator → Edge cases → Limitations → How it can be gamed → Complementary metrics

## Best Practices

- **Always start by clarifying** — restate the problem, ask about scope, define terms
- **Decompose before diagnosing** — break metrics into components before jumping to hypotheses
- **Be hypothesis-driven** — state your hypothesis, then explain how you'd validate it with data
- **Quantify impact** — "this is important" is weak; "this affects 20% of users and $2M ARR" is strong
- **Acknowledge tradeoffs** — every decision has costs; showing you understand them demonstrates maturity
- **Connect to business goals** — tie everything back to revenue, growth, or user value

## Additional Resources

- "Cracking the PM Interview" by Gayle Laakmann McDowell & Jackie Bavaro
- Lenny's Newsletter — Product management and metrics
- Exponent — Product sense interview practice
- "Measure What Matters" by John Doerr (OKR framework)
- Supplementary: Product Analytics Frameworks (Data-Science-Analytical-Handbook)
