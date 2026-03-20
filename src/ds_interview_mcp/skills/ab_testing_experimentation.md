# A/B Testing & Experimentation

## Overview
A/B testing (controlled experimentation) is the gold standard for measuring the causal
impact of product changes. Product data analysts must design, analyze, and interpret
experiments to guide shipping decisions.

## Experiment Design Checklist

1. **Hypothesis** — State clearly: "We believe [change] will [improve metric] by [amount] because [reason]."
2. **Primary metric** — One success metric the test is powered for (e.g., conversion rate).
3. **Secondary metrics** — Supporting metrics to understand mechanism.
4. **Guardrail metrics** — Metrics that must NOT degrade (latency, crash rate, revenue).
5. **Randomization unit** — User-level (most common), session-level, or device-level.
6. **Sample size** — Calculated from baseline rate, MDE, alpha, power.
7. **Duration** — Long enough for full weekly cycles (min 1-2 weeks).
8. **Segmentation plan** — Pre-register subgroup analyses to avoid p-hacking.

## Sample Size Calculation

### Two-Proportion Z-Test
For conversion metrics:
```
n = (Z_alpha/2 + Z_beta)^2 * (p1*(1-p1) + p2*(1-p2)) / (p1 - p2)^2
```

Where:
- p1 = baseline conversion rate
- p2 = p1 * (1 + MDE) = expected treatment rate
- alpha = 0.05 (significance level)
- beta = 0.20 (1 - power)
- Z_0.025 = 1.96, Z_0.20 = 0.842

### Example
Baseline CVR = 10%, MDE = 5% relative (absolute 0.5pp):
```
p1 = 0.10, p2 = 0.105
n_per_arm = (1.96 + 0.842)^2 * (0.10*0.90 + 0.105*0.895) / (0.005)^2
n_per_arm ≈ 55,736 per variant
Total = ~111,472
```

## Statistical Analysis

### Frequentist Approach
```python
from scipy import stats
import numpy as np

control_cvr = control_conversions / control_total
treatment_cvr = treatment_conversions / treatment_total

# Pooled proportion
p_pool = (control_conversions + treatment_conversions) / (control_total + treatment_total)
se = np.sqrt(p_pool * (1 - p_pool) * (1/control_total + 1/treatment_total))

z_stat = (treatment_cvr - control_cvr) / se
p_value = 2 * (1 - stats.norm.cdf(abs(z_stat)))

# Confidence interval
ci_lower = (treatment_cvr - control_cvr) - 1.96 * se
ci_upper = (treatment_cvr - control_cvr) + 1.96 * se
```

### Interpreting Results
| p-value | Confidence Interval   | Decision               |
|---------|-----------------------|------------------------|
| < 0.05  | Excludes 0, positive  | Ship (statistically significant win) |
| < 0.05  | Excludes 0, negative  | Do NOT ship (significant loss) |
| >= 0.05 | Includes 0            | Inconclusive — need more data or larger MDE |

## Common Pitfalls

### 1. Peeking / Early Stopping
Looking at results too early inflates false positive rate. Use sequential testing or
predetermined analysis windows.

### 2. Multiple Comparisons
Testing many metrics increases false discovery rate. Apply Bonferroni or
Benjamini-Hochberg corrections, or declare one primary metric upfront.

### 3. Network Effects
In social products, control and treatment users interact. Solutions:
- Cluster-based randomization (by geography or social cluster)
- Ego-network randomization
- Use network-adjusted estimators

### 4. Novelty & Primacy Effects
Users may over-engage with new features (novelty) or stick with existing patterns
(primacy). Run tests for at least 2 full weeks to let effects stabilize.

### 5. Simpson's Paradox
Aggregate results can reverse when segmented. Always check results across key segments
(platform, country, new vs existing users).

### 6. Survivorship Bias
Analysis includes only users who remained — those who churned are invisible.
Account for differential attrition between variants.

## Advanced Topics

### Variance Reduction (CUPED)
Use pre-experiment data to reduce variance:
```
Y_adjusted = Y - theta * (X - E[X])
theta = Cov(Y, X) / Var(X)
```
This can reduce required sample size by 30-50%.

### Switchback Experiments
For marketplace/supply-side tests where randomization is at the market level,
alternate treatment and control over time periods.

### Multi-Armed Bandits
When exploring many variants, use Thompson Sampling or UCB to dynamically
allocate traffic to winning variants while still learning.

## Practice Questions
1. You're testing a new checkout flow. Baseline CVR is 3.2%. You want to detect a 10%
   relative lift. Calculate sample size (alpha=0.05, power=0.8).
2. An A/B test shows p=0.03 on the primary metric but the guardrail metric (page load time)
   increased by 200ms. What do you recommend?
3. Explain why you cannot simply run an A/B test on a social feed ranking algorithm.
   What alternative design would you use?
4. Your test ran for 5 days and shows p=0.04. Should you ship? Why or why not?
