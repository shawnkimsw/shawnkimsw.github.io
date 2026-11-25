---
title: Fama-MacBeth and the Shanken correction
order: 1
---

# Fama-MacBeth regression and the Shanken correction

A linear factor model explains the cross-section of expected returns with a small number of factors. The Fama-MacBeth procedure estimates how much each factor is paid, its premium, and gives standard errors for those premiums. This note sets up the model, derives the estimator, shows why its usual standard error is too small when the exposures are estimated, and derives the Shanken correction that fixes it.

## 1. Setup

There are $N$ assets, $T$ periods and $K$ factors. Let $R_t \in \mathbb{R}^N$ be the vector of excess returns in period $t$ and $f_t \in \mathbb{R}^K$ the factor values in period $t$. Each factor has **one value per period, shared by every asset**, for example the market's excess return or the return of a long-short portfolio.

Each asset follows the time-series model

$$
R_{it} = a_i + \beta_i^\top f_t + \varepsilon_{it}, \qquad t = 1, \dots, T,
$$

where $\beta_i \in \mathbb{R}^K$ is the asset's **factor exposure**. Stacking the assets, $R_t = a + B f_t + \varepsilon_t$ with $B \in \mathbb{R}^{N \times K}$ and $\operatorname{Var}(\varepsilon_t) = \Sigma$.

The pricing model says expected excess returns are linear in the exposures:

$$
\mathbb{E}[R_t] = B\lambda,
$$

where $\lambda \in \mathbb{R}^K$ is the vector of **factor premiums**. Three objects are easy to confuse, so we name them once:

| Object | Symbol | Varies over | Estimated in |
|---|---|---|---|
| Factor values | $f_t$ | time | observed |
| Factor exposures | $\beta_i$, $B$ | assets | step 1 |
| Factor premiums | $\lambda$ | neither | step 2 |

We write $\mu_f = \mathbb{E}[f_t]$ and $\Sigma_f = \operatorname{Var}(f_t)$. For a traded factor, which is itself an excess return, the model implies $\lambda = \mu_f$.

## 2. The two steps

### 2.1 Step 1: time-series regressions

For each asset $i$, regress its returns on all $K$ factors at once by OLS:

$$
\hat\beta_i = \left( \sum_{t=1}^T \tilde f_t \tilde f_t^\top \right)^{-1} \sum_{t=1}^T \tilde f_t R_{it}, \qquad \tilde f_t = f_t - \bar f .
$$

This is one regression per asset, $N$ regressions in total. It gives the estimated exposure matrix $\hat B$.

### 2.2 Step 2: cross-sectional regressions

For each period $t$, regress the cross-section of returns on the estimated exposures:

$$
\hat\lambda_t = (\hat B^\top \hat B)^{-1} \hat B^\top R_t, \qquad t = 1, \dots, T.
$$

This gives a **time series** of premium estimates $\hat\lambda_1, \dots, \hat\lambda_T$. The Fama-MacBeth estimate is their average,

$$
\hat\lambda = \frac{1}{T} \sum_{t=1}^T \hat\lambda_t .
$$

An intercept can be added by replacing $\hat B$ with $[\mathbf{1}_N, \hat B]$. We leave it out below to keep the algebra short; the results carry over.

### 2.3 The Fama-MacBeth standard error

The standard errors reported by each cross-sectional regression are ignored. Instead, the variance of $\hat\lambda$ is estimated from the **sample variance of the premium series**:

$$
\hat V_{\text{FM}} = \frac{1}{T^2} \sum_{t=1}^T (\hat\lambda_t - \hat\lambda)(\hat\lambda_t - \hat\lambda)^\top .
$$

The appeal is that cross-sectional correlation between the residuals of different assets, which would bias the per-period OLS standard errors, is absorbed into the period-to-period variation of $\hat\lambda_t$. The formula assumes the $\hat\lambda_t$ are uncorrelated over time.

## 3. Autocorrelated premiums: HAC standard errors

Premium estimates are often autocorrelated, for example when factors are persistent or when returns are measured over overlapping horizons. The variance of the mean is then estimated with the Newey-West estimator:

$$
\hat V_{\text{HAC}} = \frac{1}{T}\left[ \hat\Gamma_0 + \sum_{j=1}^{L} \left(1 - \frac{j}{L+1}\right)\left(\hat\Gamma_j + \hat\Gamma_j^\top\right) \right],
$$

where $L$ is the number of lags and

$$
\hat\Gamma_j = \frac{1}{T}\sum_{t=j+1}^{T} (\hat\lambda_t - \hat\lambda)(\hat\lambda_{t-j} - \hat\lambda)^\top .
$$

With $L = 0$ this reduces to $\hat V_{\text{FM}}$ up to the divisor. HAC standard errors change only the inference, never the estimate $\hat\lambda$.

## 4. The errors-in-variables problem

The second step regresses on $\hat B$, not on the true $B$. Write

$$
\hat B = B + U,
$$

where $U$ is the estimation error from step 1. In an ordinary regression the regressor may be random, but its value is observed, so inference conditional on it is valid. Here the true exposures are never observed. Two consequences follow:

1. **Bias.** Regressing on a noisy regressor attenuates the slope toward zero. Since $U \to 0$ as $T \to \infty$, $\hat\lambda$ is still **consistent**, but in short samples it is biased toward zero.
2. **Standard error.** The same $\hat B$ is used in every period, so its error $U$ is the same in every cross-sectional regression. It shifts every $\hat\lambda_t$ in the same direction and never shows up as variation over time. $\hat V_{\text{FM}}$ measures only variation over time, so it misses this component and is too small.

## 5. Derivation of the Shanken correction

We follow Shanken (1992). Assume $\varepsilon_t$ is i.i.d. over time with covariance $\Sigma$ and independent of the factors, and condition on the factor path $f_1, \dots, f_T$.

### 5.1 The estimation error in the exposures

Stack the demeaned factors as $\tilde F \in \mathbb{R}^{T \times K}$ and the residuals as $E \in \mathbb{R}^{T \times N}$. The OLS error in step 1 is

$$
U = E^\top \tilde F (\tilde F^\top \tilde F)^{-1}.
$$

For any fixed vector $v \in \mathbb{R}^K$, $Uv = \sum_t \varepsilon_t w_t$ with scalar weights $w_t = \tilde f_t^\top (\tilde F^\top \tilde F)^{-1} v$, so

$$
\operatorname{Var}(Uv) = \Sigma \sum_t w_t^2 = \Sigma \, v^\top (\tilde F^\top \tilde F)^{-1} v = \frac{1}{T}\, \Sigma \, v^\top \hat\Sigma_f^{-1} v .
$$

This is the matrix version of a simple fact: an exposure is hard to estimate when its factor barely moves, because $\hat\Sigma_f$ is small and its inverse is large.

### 5.2 The error in the premium

Since $\hat B$ is the same in every period, $\hat\lambda = (\hat B^\top \hat B)^{-1}\hat B^\top \bar R$ with $\bar R = \frac{1}{T}\sum_t R_t$. Averaging the time-series model, using $a = B(\lambda - \mu_f)$ from the pricing model, gives

$$
\bar R = B\lambda + B(\bar f - \mu_f) + \bar\varepsilon .
$$

Substituting $B = \hat B - U$,

$$
\bar R = \hat B \lambda - U\lambda + B(\bar f - \mu_f) + \bar\varepsilon ,
$$

so, to first order,

$$
\hat\lambda - \lambda \approx (B^\top B)^{-1} B^\top \left( \bar\varepsilon - U\lambda \right) + (\bar f - \mu_f).
$$

The term $-U\lambda$ is the new piece. The exposure error enters **multiplied by the premium**, so a larger premium amplifies it.

### 5.3 The variance

The three terms are uncorrelated: $\bar f$ is independent of the residuals, and $\operatorname{Cov}(\bar\varepsilon, U\lambda) \propto \sum_t \tilde f_t = 0$ because the factors are demeaned. Using $\operatorname{Var}(\bar\varepsilon) = \Sigma / T$ and Section 5.1 with $v = \lambda$,

$$
\operatorname{Var}(\hat\lambda) \approx \frac{1}{T}\left[ \left(1 + \lambda^\top \Sigma_f^{-1} \lambda\right) (B^\top B)^{-1} B^\top \Sigma B (B^\top B)^{-1} + \Sigma_f \right].
$$

### 5.4 What Fama-MacBeth estimates instead

The deviations used by $\hat V_{\text{FM}}$ are

$$
\hat\lambda_t - \hat\lambda \approx (B^\top B)^{-1} B^\top (\varepsilon_t - \bar\varepsilon) + (f_t - \bar f).
$$

The $U\lambda$ term is constant over $t$, so it cancels in the deviations. Hence

$$
\hat V_{\text{FM}} \xrightarrow{p} \frac{1}{T}\left[ (B^\top B)^{-1} B^\top \Sigma B (B^\top B)^{-1} + \Sigma_f \right],
$$

which is the correct variance with $c = 0$.

### 5.5 The correction

Let $c = \lambda^\top \Sigma_f^{-1} \lambda$. For a single factor, $c = (\lambda / \sigma_f)^2$, the factor's squared Sharpe ratio when the factor is traded. Comparing 5.3 and 5.4, the Shanken-corrected variance is

$$
\hat V_{\text{Shanken}} = (1 + \hat c)\left( \hat V_{\text{FM}} - \frac{\hat\Sigma_f}{T} \right) + \frac{\hat\Sigma_f}{T}.
$$

A common shorthand multiplies all of $\hat V_{\text{FM}}$ by $1 + \hat c$. This slightly overstates the correction, so it is conservative.

**Intuition.** The extra variance grows with the premium squared, because the exposure error enters multiplied by the premium. It shrinks with the factor's variance, because a volatile factor lets step 1 pin down the exposures precisely. If the factor's Sharpe ratio is small, $c \approx 0$ and the correction barely matters.

**Example.** For the CAPM with annual data, a market premium of 6% and volatility of 15% give $1 + c = 1 + (0.06/0.15)^2 = 1.16$. The variance rises by 16%, so the standard error rises by about 8%.

## 6. Testing a factor premium

To test whether a premium is positive, set $H_0: \lambda_k \le 0$ against $H_1: \lambda_k > 0$ and use the one-sided statistic

$$
t_k = \frac{\hat\lambda_k}{\sqrt{\hat V_{kk}}} .
$$

In practice:

1. Use $\hat V_{\text{HAC}}$ in place of $\hat V_{\text{FM}}$ when the premium series is autocorrelated.
2. Apply the Shanken correction on top when the exposures are estimated betas. Shanken derived it under i.i.d. residuals, so combining it with HAC is a common approximation rather than an exact result.

The correction does not apply when the exposures are **observed characteristics**, such as momentum or size, used directly as $B$, as in Barra-style risk models. There is no step 1, so there is no estimation error in the regressors.

## 7. Choosing the estimator in each step

| Step | Default | Reason |
|---|---|---|
| Time-series regression | OLS | HAC standard errors handle autocorrelated residuals; they change only inference |
| Cross-sectional regression | WLS | Residual variance differs across assets; weights are often inverse residual variance or a liquidity measure |
| Correlated residuals | GLS or FGLS in principle | With many assets the residual covariance matrix is too noisy to estimate well, so WLS is usually more reliable |
| Influential observations | Robust regression (Huber loss) | The loss is quadratic for small residuals and linear for large ones, so a few extreme returns get less weight |

Winsorizing returns before the cross-sectional regression is a simpler alternative to robust regression.

## References

- Fama, E. F. and MacBeth, J. D. (1973). *Risk, Return, and Equilibrium: Empirical Tests.* Journal of Political Economy 81(3), 607–636.
- Shanken, J. (1992). *On the Estimation of Beta-Pricing Models.* Review of Financial Studies 5(1), 1–33.
- Newey, W. K. and West, K. D. (1987). *A Simple, Positive Semi-Definite, Heteroskedasticity and Autocorrelation Consistent Covariance Matrix.* Econometrica 55(3), 703–708.
- Jagannathan, R. and Wang, Z. (1998). *An Asymptotic Theory for Estimating Beta-Pricing Models Using Cross-Sectional Regression.* Journal of Finance 53(4), 1285–1309.
- Cochrane, J. H. (2005). *Asset Pricing*, revised edition, Chapter 12. Princeton University Press.
