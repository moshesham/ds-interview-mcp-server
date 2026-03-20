---
name: ab-testing-experimentation
description: Design, analyze, and interpret A/B tests and controlled experiments. Calculate sample sizes, run power analysis, detect common pitfalls (peeking, multiple comparisons, network effects), and apply advanced techniques like CUPED variance reduction and switchback experiments. For product experimentation, feature launches, and causal inference.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# A/B Testing & Experimentation

## Overview

A/B testing (online controlled experimentation) is the gold standard for measuring causal impact of product changes. A well-designed experiment randomly assigns users to treatment and control groups, measures a pre-defined metric, and uses statistical inference to determine whether the treatment had a real effect.

**Key concepts covered:**
- **Experiment design**: Hypothesis, randomization unit, metric selection, guardrails
- **Sample size & power analysis**: Minimum detectable effect, statistical power, significance level
- **Analysis methods**: Frequentist (z-test, t-test), sequential testing, CUPED
- **Common pitfalls**: Peeking, multiple comparisons, novelty effects, interference

## When to Use This Skill

Use this skill when:

- **Launching a new feature**: Measuring causal impact on engagement, revenue, or retention
- **Optimizing user flows**: Testing a checkout redesign, onboarding variant, or pricing page
- **Interview preparation**: A/B testing design questions are common at every FAANG company
- **Reviewing experiment results**: Checking for validity, significance, and practical importance
- **Resolving debates**: Replacing opinions with data when stakeholders disagree
- **Building experimentation platforms**: Designing assignment, logging, and analysis pipelines

## Experiment Design Checklist

Follow these 8 steps before launching any experiment:

1. **State the hypothesis**: "Changing X will improve Y by at least Z%"
2. **Choose the randomization unit**: User-level (most common), session, device, or region
3. **Define primary metric**: One metric that directly measures the hypothesis
4. **Define guardrail metrics**: Metrics that must NOT degrade (e.g., crash rate, latency)
5. **Calculate sample size**: Based on baseline rate, MDE, power, and significance level
6. **Set experiment duration**: Account for weekly cycles (run at least 1-2 full weeks)
7. **Check for interference**: Network effects, shared resources, marketplace dynamics
8. **Pre-register the analysis plan**: Specify segmentation, metric definitions, decision rules

## Sample Size Calculation

### Two-Proportion Z-Test

The most common case: comparing conversion rates between control and treatment.

```python
import numpy as np
from scipy import stats

def calculate_sample_size(
    baseline_rate: float,
    minimum_detectable_effect: float,
    alpha: float = 0.05,
    power: float = 0.80,
    two_sided: bool = True
) -> dict:
    """
    Calculate required sample size per group for a two-proportion z-test.

    Args:
        baseline_rate: Current conversion rate (e.g., 0.10 for 10%)
        minimum_detectable_effect: Relative change to detect (e.g., 0.05 for 5% relative lift)
        alpha: Significance level (Type I error rate)
        power: Statistical power (1 - Type II error rate)
        two_sided: Whether the test is two-sided

    Returns:
        Dict with sample size per group, total, and parameters
    """
    p1 = baseline_rate
    p2 = baseline_rate * (1 + minimum_detectable_effect)

    # Pooled proportion
    p_bar = (p1 + p2) / 2

    # Z-scores
    z_alpha = stats.norm.ppf(1 - alpha / (2 if two_sided else 1))
    z_beta = stats.norm.ppf(power)

    # Sample size formula (per group)
    numerator = (z_alpha * np.sqrt(2 * p_bar * (1 - p_bar)) +
                 z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2
    denominator = (p2 - p1) ** 2

    n_per_group = int(np.ceil(numerator / denominator))

    return {
        'n_per_group': n_per_group,
        'n_total': n_per_group * 2,
        'baseline_rate': p1,
        'expected_treatment_rate': p2,
        'absolute_difference': p2 - p1,
        'relative_mde': minimum_detectable_effect,
        'alpha': alpha,
        'power': power,
    }

# Example: 10% baseline conversion, detect 5% relative lift
result = calculate_sample_size(baseline_rate=0.10, minimum_detectable_effect=0.05)
print(f"Need {result['n_per_group']:,} users per group ({result['n_total']:,} total)")
print(f"Detecting {result['baseline_rate']:.1%} → {result['expected_treatment_rate']:.1%}")
```

### Continuous Metrics (T-Test)

```python
def sample_size_continuous(
    baseline_mean: float,
    baseline_std: float,
    minimum_detectable_effect: float,
    alpha: float = 0.05,
    power: float = 0.80
) -> int:
    """
    Sample size for detecting a change in a continuous metric (e.g., revenue per user).

    Args:
        baseline_mean: Current metric mean
        baseline_std: Current metric standard deviation
        minimum_detectable_effect: Relative change to detect

    Returns:
        Sample size per group
    """
    delta = baseline_mean * minimum_detectable_effect
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)

    n = int(np.ceil(2 * ((z_alpha + z_beta) * baseline_std / delta) ** 2))
    return n

# Example: revenue per user, mean=$5.00, std=$12.00, detect 3% lift
n = sample_size_continuous(5.0, 12.0, 0.03)
print(f"Need {n:,} users per group")
```

## Analyzing Experiment Results

### Frequentist Analysis

```python
def analyze_ab_test(
    control_conversions: int,
    control_total: int,
    treatment_conversions: int,
    treatment_total: int,
    alpha: float = 0.05
) -> dict:
    """
    Analyze an A/B test result for a binary outcome.

    Returns:
        Dict with rates, lift, confidence interval, p-value, and decision
    """
    p_c = control_conversions / control_total
    p_t = treatment_conversions / treatment_total
    lift = (p_t - p_c) / p_c if p_c > 0 else float('inf')

    # Standard error of the difference
    se = np.sqrt(p_c * (1 - p_c) / control_total + p_t * (1 - p_t) / treatment_total)

    # Z-statistic and p-value
    z_stat = (p_t - p_c) / se
    p_value = 2 * (1 - stats.norm.cdf(abs(z_stat)))

    # Confidence interval for the difference
    z_crit = stats.norm.ppf(1 - alpha / 2)
    ci_lower = (p_t - p_c) - z_crit * se
    ci_upper = (p_t - p_c) + z_crit * se

    return {
        'control_rate': p_c,
        'treatment_rate': p_t,
        'absolute_difference': p_t - p_c,
        'relative_lift': lift,
        'standard_error': se,
        'z_statistic': z_stat,
        'p_value': p_value,
        'ci_95': (ci_lower, ci_upper),
        'significant': p_value < alpha,
        'decision': 'SHIP IT' if p_value < alpha and lift > 0 else
                    'DO NOT SHIP' if p_value < alpha and lift < 0 else
                    'NO SIGNIFICANT DIFFERENCE'
    }

# Example
result = analyze_ab_test(
    control_conversions=4900, control_total=50000,
    treatment_conversions=5150, treatment_total=50000
)
for k, v in result.items():
    print(f"  {k}: {v}")
```

## Common Pitfalls

### 1. Peeking (Repeated Significance Testing)

Checking results daily and stopping when p < 0.05 inflates false positive rates up to 30%.

**Solution**: Use sequential testing methods (e.g., always-valid p-values, group sequential designs) or pre-commit to a fixed end date.

### 2. Multiple Comparisons

Testing 20 metrics at α=0.05 means ~1 will be "significant" by chance alone.

**Solution**: Bonferroni correction (divide α by number of tests) or designate ONE primary metric.

```python
def bonferroni_correction(p_values: list, alpha: float = 0.05) -> list:
    """Apply Bonferroni correction: adjust alpha by number of comparisons."""
    adjusted_alpha = alpha / len(p_values)
    return [{'p_value': p, 'significant': p < adjusted_alpha} for p in p_values]
```

### 3. Network Effects / Interference (SUTVA Violation)

In social products, treating user A changes the experience of user A's untreated friends.

**Solution**: Cluster randomization (randomize by social cluster or geo region), or ego-network randomization.

### 4. Novelty & Primacy Effects

Users may engage more with any change simply because it's new, or stick with the old design out of habit.

**Solution**: Run experiments for 2-4 weeks; segment by "saw treatment on day 1" vs. "saw treatment on day 7+" to detect decay.

### 5. Simpson's Paradox

A treatment can appear better overall but be worse when you segment by a confounding variable (e.g., platform mix shift).

**Solution**: Always check key segments (platform, country, new vs. existing users).

### 6. Survivorship Bias

Analyzing only users who stayed through the entire experiment ignores those who churned because of the treatment.

**Solution**: Use intent-to-treat analysis—include all randomized users regardless of whether they completed the experience.

## Advanced Techniques

### CUPED (Controlled-experiment Using Pre-Experiment Data)

Reduces variance by adjusting for pre-experiment behavior, increasing experiment sensitivity by 20-50%.

```python
def cuped_adjustment(
    y_post: np.ndarray,
    y_pre: np.ndarray
) -> np.ndarray:
    """
    Apply CUPED variance reduction.

    Uses pre-experiment metric values as a control variate to reduce
    variance of the post-experiment metric estimate.

    Args:
        y_post: Post-experiment metric values per user
        y_pre: Pre-experiment metric values per user (same period length)

    Returns:
        Adjusted metric values with reduced variance
    """
    # Compute theta (optimal coefficient)
    cov = np.cov(y_post, y_pre)[0, 1]
    var_pre = np.var(y_pre)
    theta = cov / var_pre if var_pre > 0 else 0

    # Adjusted metric
    y_adjusted = y_post - theta * (y_pre - np.mean(y_pre))
    return y_adjusted
```

### Switchback Experiments

For marketplace/logistics products where user-level randomization causes interference:

| Period | Region A | Region B |
|--------|----------|----------|
| Hour 1 | Treatment | Control |
| Hour 2 | Control | Treatment |
| Hour 3 | Treatment | Control |
| ... | ... | ... |

### Multi-Armed Bandits

When you want to minimize regret during the experiment (e.g., pricing, recommendations):

- **Thompson Sampling**: Bayesian approach, balances exploration vs. exploitation
- **UCB (Upper Confidence Bound)**: Optimistic exploration
- **Trade-off**: Bandits optimize for reward during the experiment but provide weaker statistical guarantees

## Best Practices

- **Always run an A/A test first** on a new platform to validate that the randomization and logging work correctly
- **Pre-register your analysis plan**: primary metric, guardrails, sample size, duration, segmentation
- **Run for at least 1-2 full weeks** to capture weekly seasonality
- **Check for Sample Ratio Mismatch (SRM)**: If control/treatment split deviates significantly from 50/50, your randomization is broken
- **Distinguish statistical vs. practical significance**: A p-value of 0.03 with a 0.01% lift is real but irrelevant
- **Document everything**: hypothesis, design, results, decision, and learnings for institutional memory

## Additional Resources

- Kohavi, R., Tang, D., & Xu, Y. — *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing*
- Deng, A. et al. — "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data" (CUPED paper)
- Microsoft ExP Platform — "Online Controlled Experiments at Scale"
- Uber Engineering Blog — "Under the Hood of Uber's Experimentation Platform"
- Netflix Tech Blog — "Experimentation is a Major Focus at Netflix"
