---
name: statistics-for-product-analytics
description: Applied statistics for product data analytics including hypothesis testing, confidence intervals, power analysis, distributions, Bayesian methods, and common statistical pitfalls. Covers practical application of statistics in A/B testing, metric analysis, and data-driven decision making for product teams.
license: MIT
metadata:
    skill-author: ds-interview-mcp
---

# Statistics for Product Analytics

## Overview

Product analysts use statistics daily — from interpreting A/B test results to building confidence intervals around metric estimates. This skill covers the practical statistics needed for product analytics work and interviews, focused on correct application rather than mathematical proofs.

**Key areas:**
- **Hypothesis testing**: z-tests, t-tests, chi-squared, when to use each
- **Confidence intervals**: Construction, interpretation, common misunderstandings
- **Power analysis**: Sample size planning, minimum detectable effect
- **Distributions**: Normal, binomial, Poisson — which metrics follow which
- **Bayesian thinking**: Priors, posteriors, and when Bayesian approaches help
- **Statistical pitfalls**: P-hacking, multiple comparisons, Simpson's paradox

## When to Use This Skill

Use this skill when:

- **Analyzing A/B test results**: Is the observed difference statistically significant?
- **Planning experiments**: How many users do we need? How long should we run the test?
- **Building dashboards**: Adding confidence intervals to metric estimates
- **Interviewing**: "Is this result significant?" is asked in every product analytics interview
- **Communicating uncertainty**: Helping stakeholders understand what the data does and doesn't tell us

## Hypothesis Testing

### Framework

```
1. State H₀ (null hypothesis) and H₁ (alternative hypothesis)
2. Choose significance level α (typically 0.05)
3. Choose the test statistic based on data type and assumptions
4. Compute the test statistic and p-value
5. Compare p-value to α → reject or fail to reject H₀
6. Interpret in business terms
```

### Choosing the Right Test

| Scenario | Test | Python Function |
|----------|------|----------------|
| Compare two proportions (conversion rates) | Two-proportion z-test | `statsmodels.stats.proportion.proportions_ztest` |
| Compare two means (revenue per user) | Two-sample t-test | `scipy.stats.ttest_ind` |
| Compare means, paired data (before/after) | Paired t-test | `scipy.stats.ttest_rel` |
| Compare proportions across categories | Chi-squared test | `scipy.stats.chi2_contingency` |
| Non-normal data, compare medians | Mann-Whitney U test | `scipy.stats.mannwhitneyu` |
| Compare more than 2 groups | ANOVA / Kruskal-Wallis | `scipy.stats.f_oneway` / `kruskal` |

### Python Implementation

```python
import numpy as np
from scipy import stats


def two_proportion_ztest(
    successes_a: int, n_a: int,
    successes_b: int, n_b: int,
    alpha: float = 0.05
) -> dict:
    """Two-proportion z-test for comparing conversion rates.
    
    Args:
        successes_a: Number of conversions in control
        n_a: Total users in control
        successes_b: Number of conversions in treatment
        n_b: Total users in treatment
        alpha: Significance level
    
    Returns:
        Dictionary with test results and interpretation
    """
    p_a = successes_a / n_a
    p_b = successes_b / n_b
    p_pool = (successes_a + successes_b) / (n_a + n_b)
    
    se = np.sqrt(p_pool * (1 - p_pool) * (1/n_a + 1/n_b))
    z_stat = (p_b - p_a) / se
    p_value = 2 * (1 - stats.norm.cdf(abs(z_stat)))  # two-tailed
    
    # Confidence interval for the difference
    se_diff = np.sqrt(p_a * (1 - p_a) / n_a + p_b * (1 - p_b) / n_b)
    z_crit = stats.norm.ppf(1 - alpha / 2)
    ci_lower = (p_b - p_a) - z_crit * se_diff
    ci_upper = (p_b - p_a) + z_crit * se_diff
    
    return {
        "control_rate": round(p_a, 4),
        "treatment_rate": round(p_b, 4),
        "absolute_lift": round(p_b - p_a, 4),
        "relative_lift_pct": round(100 * (p_b - p_a) / p_a, 2) if p_a > 0 else None,
        "z_statistic": round(z_stat, 3),
        "p_value": round(p_value, 4),
        "significant": p_value < alpha,
        "ci_95": (round(ci_lower, 4), round(ci_upper, 4)),
    }


def two_sample_ttest(
    values_a: np.ndarray,
    values_b: np.ndarray,
    alpha: float = 0.05,
    equal_var: bool = False
) -> dict:
    """Two-sample t-test for comparing means (e.g., revenue per user).
    
    Uses Welch's t-test (equal_var=False) by default since equal
    variance assumption rarely holds in practice.
    """
    t_stat, p_value = stats.ttest_ind(values_a, values_b, equal_var=equal_var)
    
    mean_a, mean_b = np.mean(values_a), np.mean(values_b)
    se_diff = np.sqrt(np.var(values_a, ddof=1)/len(values_a) + np.var(values_b, ddof=1)/len(values_b))
    t_crit = stats.t.ppf(1 - alpha/2, df=min(len(values_a), len(values_b)) - 1)
    
    return {
        "mean_control": round(float(mean_a), 4),
        "mean_treatment": round(float(mean_b), 4),
        "difference": round(float(mean_b - mean_a), 4),
        "t_statistic": round(float(t_stat), 3),
        "p_value": round(float(p_value), 4),
        "significant": p_value < alpha,
        "ci_95": (
            round(float((mean_b - mean_a) - t_crit * se_diff), 4),
            round(float((mean_b - mean_a) + t_crit * se_diff), 4),
        ),
    }
```

## Power Analysis

### Sample Size Calculation

```python
def required_sample_size(
    baseline_rate: float,
    minimum_detectable_effect: float,
    alpha: float = 0.05,
    power: float = 0.80,
) -> int:
    """Calculate required sample size per group for a two-proportion test.
    
    Args:
        baseline_rate: Current conversion rate (e.g., 0.10 for 10%)
        minimum_detectable_effect: Minimum relative change to detect (e.g., 0.05 for 5% relative lift)
        alpha: Significance level
        power: Statistical power (1 - β)
    
    Returns:
        Required sample size per group
    """
    p1 = baseline_rate
    p2 = baseline_rate * (1 + minimum_detectable_effect)
    
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)
    
    p_avg = (p1 + p2) / 2
    
    n = ((z_alpha * np.sqrt(2 * p_avg * (1 - p_avg)) +
          z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) /
         (p2 - p1)) ** 2
    
    return int(np.ceil(n))


# Example: 10% baseline conversion, want to detect 5% relative lift
# required_sample_size(0.10, 0.05) → ~31,234 per group
```

### Power Analysis Decision Table

| Baseline Rate | MDE (relative) | α=0.05, Power=0.80 | Duration at 10K users/day |
|:---:|:---:|:---:|:---:|
| 5% | 10% | ~30,400/group | ~6 days |
| 5% | 5% | ~121,600/group | ~24 days |
| 10% | 10% | ~14,300/group | ~3 days |
| 10% | 5% | ~57,200/group | ~11 days |
| 20% | 10% | ~6,400/group | ~1.3 days |
| 20% | 5% | ~25,600/group | ~5 days |

## Confidence Intervals

### Correct Interpretation

> "If we repeated this experiment many times, 95% of the computed intervals would contain the true parameter."

**NOT**: "There is a 95% probability the true value is in this interval."

### Bootstrap Confidence Intervals

```python
def bootstrap_ci(
    data: np.ndarray,
    statistic: callable = np.mean,
    n_bootstrap: int = 10000,
    ci_level: float = 0.95,
    seed: int = 42
) -> tuple[float, float, float]:
    """Compute bootstrap confidence interval for any statistic.
    
    Returns:
        Tuple of (point_estimate, ci_lower, ci_upper)
    """
    rng = np.random.default_rng(seed)
    boot_stats = np.array([
        statistic(rng.choice(data, size=len(data), replace=True))
        for _ in range(n_bootstrap)
    ])
    
    alpha = 1 - ci_level
    ci_lower = np.percentile(boot_stats, 100 * alpha / 2)
    ci_upper = np.percentile(boot_stats, 100 * (1 - alpha / 2))
    
    return float(statistic(data)), float(ci_lower), float(ci_upper)
```

## Common Statistical Pitfalls

### 1. P-hacking / Multiple Testing

**Problem**: Testing many metrics or segments inflates false positive rate.

With 20 independent tests at α=0.05: P(at least one false positive) = 1 - (0.95)^20 = 64%

**Solutions**:
- Bonferroni correction: α_adj = α / number_of_tests
- Benjamini-Hochberg (FDR control): less conservative, controls false discovery rate
- Pre-register primary metric before running the test

### 2. Peeking at Results

**Problem**: Checking significance repeatedly as data comes in inflates false positive rate.

**Solutions**:
- Pre-commit to the sample size / duration
- Sequential testing methods (group sequential, always-valid p-values)

### 3. Simpson's Paradox

**Problem**: A trend that appears in several groups reverses when groups are combined.

**Example**: Treatment B has higher conversion in both desktop (30% vs 28%) and mobile (15% vs 14%), but treatment A has higher overall conversion because A has more mobile users (lower conversion segment).

**Solution**: Always segment by key confounders before drawing conclusions.

### 4. Survivorship Bias

**Problem**: Analyzing only users who made it to a certain point, ignoring those who dropped off.

**Example**: "Users who completed onboarding have 40% retention." Selection effect — they were already more engaged.

**Solution**: Analyze from the point of initial exposure, not from the point of completion.

### 5. Confusing Statistical and Practical Significance

**Problem**: With enough data, even tiny effects become statistically significant.

**Solution**: Always report effect size and confidence interval alongside p-value. Ask: "Is this effect large enough to matter for the business?"

## Distributions in Product Analytics

| Distribution | Common Use | Example |
|-------------|-----------|---------|
| Normal | Means of large samples (CLT) | Average session duration |
| Binomial | Success/failure outcomes | Conversion rate |
| Poisson | Count data with known average rate | Support tickets per day |
| Exponential | Time between events | Time between purchases |
| Log-normal | Right-skewed positive values | Revenue per user, session length |
| Power law | Heavy-tailed distributions | Content virality, page views |

## Best Practices

- **Always report confidence intervals**, not just p-values — CIs convey both significance and effect size
- **Use Welch's t-test** (unequal variance) as the default — it's more robust and reduces to the equal-variance t-test when variances are actually equal
- **Log-transform right-skewed metrics** (revenue, session time) before applying parametric tests
- **Pre-register your analysis plan**: primary metric, sample size, significance level, analysis method
- **Visualize the data** before running any test — boxplots, histograms, CDFs reveal more than a p-value ever will
- **When in doubt, bootstrap**: Non-parametric bootstrap works for any statistic without distributional assumptions

## Additional Resources

- "Statistics Done Wrong" by Alex Reinhart (free online)
- "Trustworthy Online Controlled Experiments" by Kohavi, Tang & Xu
- Khan Academy Statistics & Probability course
- Seeing Theory — Brown University visual statistics
- Supplementary: Statistical Testing & Inference (Data-Science-Analytical-Handbook)
