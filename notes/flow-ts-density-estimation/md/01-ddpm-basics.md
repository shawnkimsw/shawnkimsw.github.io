---
title: DDPM basics
order: 1
---

# DDPM basics

Notes on Denoising Diffusion Probabilistic Models (Ho, Jain and Abbeel, 2020), the root of this project. Later notes build on it toward flow-based models for estimating the full conditional density of a time series.

## 1. The idea in one paragraph

A diffusion model learns a distribution by learning to undo noise. We fix a **forward process** that slowly destroys a data point $x_0$ by adding Gaussian noise over $T$ steps, until $x_T$ is indistinguishable from $\mathcal{N}(0, I)$. We then train a network to run the **reverse process**: starting from pure noise, remove a little noise at a time until a realistic sample appears. Sampling many times gives many draws from the learned distribution, and that set of draws is the density forecast.

Intuition: going from noise to data in one jump is hard, but each small denoising step is close to Gaussian, so it is easy to model. The model is a long chain of easy steps.

## 2. Notation

| Symbol | Meaning |
|---|---|
| $x_0$ | a clean data sample (later: the future window of a time series) |
| $x_t$ | the sample after $t$ noising steps, $t = 1, \dots, T$ |
| $\beta_t \in (0, 1)$ | variance of the noise added at step $t$ (the noise schedule) |
| $\alpha_t = 1 - \beta_t$ | fraction of signal variance kept at step $t$ |
| $\bar\alpha_t = \prod_{s=1}^{t} \alpha_s$ | fraction of signal variance left after $t$ steps |
| $\epsilon \sim \mathcal{N}(0, I)$ | standard Gaussian noise |
| $\epsilon_\theta(x_t, t)$ | the neural network, which predicts the noise in $x_t$ |

## 3. Forward process

Each step shrinks the signal slightly and adds a little noise:

$$
q(x_t \mid x_{t-1}) = \mathcal{N}\!\left(\sqrt{\alpha_t}\, x_{t-1},\ (1 - \alpha_t) I\right),
\qquad
x_t = \sqrt{\alpha_t}\, x_{t-1} + \sqrt{1 - \alpha_t}\, \epsilon_t .
$$

The shrink factor $\sqrt{\alpha_t}$ is what keeps the variance bounded: if $\operatorname{Var}(x_{t-1}) = 1$ then $\operatorname{Var}(x_t) = \alpha_t + (1 - \alpha_t) = 1$. This is why it is called **variance preserving**.

### Closed form: jumping straight to step $t$

Unroll one step:

$$
x_t = \sqrt{\alpha_t \alpha_{t-1}}\, x_{t-2} + \sqrt{\alpha_t (1 - \alpha_{t-1})}\, \epsilon_{t-1} + \sqrt{1 - \alpha_t}\, \epsilon_t .
$$

The two noise terms are independent Gaussians, so their sum is Gaussian with variance

$$
\alpha_t (1 - \alpha_{t-1}) + (1 - \alpha_t) = 1 - \alpha_t \alpha_{t-1}.
$$

By induction,

$$
\boxed{\,x_t = \sqrt{\bar\alpha_t}\, x_0 + \sqrt{1 - \bar\alpha_t}\, \epsilon,
\qquad
q(x_t \mid x_0) = \mathcal{N}\!\left(\sqrt{\bar\alpha_t}\, x_0,\ (1 - \bar\alpha_t) I\right).\,}
$$

Sanity check: the squared coefficients sum to one, $\bar\alpha_t + (1 - \bar\alpha_t) = 1$.

Why this matters: training never has to simulate the chain. Pick a random $t$, draw one $\epsilon$, and form $x_t$ directly.

### Signal-to-noise ratio

$$
\mathrm{SNR}(t) = \frac{\bar\alpha_t}{1 - \bar\alpha_t},
$$

which decreases from very large at $t = 1$ to nearly zero at $t = T$. The noise schedule is really a choice of how fast the SNR falls.

## 4. Reverse process

The model is a Markov chain run backwards from noise:

$$
p(x_T) = \mathcal{N}(0, I),
\qquad
p_\theta(x_{t-1} \mid x_t) = \mathcal{N}\!\left(\mu_\theta(x_t, t),\ \sigma_t^2 I\right).
$$

Why a Gaussian is a reasonable guess for each reverse step: when $\beta_t$ is small, the true reverse conditional $q(x_{t-1} \mid x_t)$ is approximately Gaussian (Feller, 1949). This holds only because the steps are small, which is why $T$ is large.

### The tractable target: the posterior given $x_0$

$q(x_{t-1} \mid x_t)$ itself is intractable because it depends on the unknown data distribution. Conditioning on $x_0$ makes it tractable. By Bayes' rule, with every factor Gaussian,

$$
q(x_{t-1} \mid x_t, x_0) = \mathcal{N}\!\left(\tilde\mu_t(x_t, x_0),\ \tilde\beta_t I\right),
$$

$$
\tilde\mu_t(x_t, x_0) = \frac{\sqrt{\bar\alpha_{t-1}}\,\beta_t}{1 - \bar\alpha_t}\, x_0 + \frac{\sqrt{\alpha_t}\,(1 - \bar\alpha_{t-1})}{1 - \bar\alpha_t}\, x_t,
\qquad
\tilde\beta_t = \frac{1 - \bar\alpha_{t-1}}{1 - \bar\alpha_t}\, \beta_t .
$$

Intuition: the posterior mean is a weighted average of "where we are" ($x_t$) and "where we started" ($x_0$). Early in the reverse process (large $t$) it leans on $x_t$; near the end it leans on $x_0$.

### Rewriting the mean in terms of the noise

Substitute $x_0 = \left(x_t - \sqrt{1 - \bar\alpha_t}\, \epsilon\right) / \sqrt{\bar\alpha_t}$ from the closed form:

$$
\tilde\mu_t = \frac{1}{\sqrt{\alpha_t}} \left( x_t - \frac{\beta_t}{\sqrt{1 - \bar\alpha_t}}\, \epsilon \right).
$$

So if a network can predict the noise $\epsilon$ inside $x_t$, it knows the posterior mean. That is the **$\epsilon$-parameterization**:

$$
\mu_\theta(x_t, t) = \frac{1}{\sqrt{\alpha_t}} \left( x_t - \frac{\beta_t}{\sqrt{1 - \bar\alpha_t}}\, \epsilon_\theta(x_t, t) \right).
$$

## 5. Training objective

### The variational bound

Like a VAE, DDPM maximizes a lower bound on the log-likelihood. The negative bound splits into one term per step:

$$
-\log p_\theta(x_0) \le
\underbrace{D_{\mathrm{KL}}\!\left(q(x_T \mid x_0) \,\Vert\, p(x_T)\right)}_{L_T}
+ \sum_{t=2}^{T} \underbrace{\mathbb{E}_q\, D_{\mathrm{KL}}\!\left(q(x_{t-1} \mid x_t, x_0) \,\Vert\, p_\theta(x_{t-1} \mid x_t)\right)}_{L_{t-1}}
\underbrace{-\, \mathbb{E}_q \log p_\theta(x_0 \mid x_1)}_{L_0}.
$$

- $L_T$ has no parameters, and it is near zero when $\bar\alpha_T \approx 0$.
- Each $L_{t-1}$ compares two Gaussians, so it is available in closed form.
- $L_0$ is a reconstruction term.

### From KL to a regression

For two Gaussians with the same variance $\sigma_t^2 I$,

$$
L_{t-1} = \frac{1}{2\sigma_t^2}\, \mathbb{E}\, \lVert \tilde\mu_t - \mu_\theta \rVert^2 + C
= \frac{\beta_t^2}{2 \sigma_t^2\, \alpha_t\, (1 - \bar\alpha_t)}\, \mathbb{E}\, \lVert \epsilon - \epsilon_\theta(x_t, t) \rVert^2 + C .
$$

### The simple loss

Ho et al. drop the weight in front and sample $t$ uniformly:

$$
\boxed{\,L_{\text{simple}}(\theta) = \mathbb{E}_{x_0,\ t \sim \mathcal{U}\{1, \dots, T\},\ \epsilon} \left\lVert \epsilon - \epsilon_\theta\!\left(\sqrt{\bar\alpha_t}\, x_0 + \sqrt{1 - \bar\alpha_t}\, \epsilon,\ t\right) \right\rVert^2 .\,}
$$

Intuition: show the network a noisy sample and ask "what noise was added?" That is ordinary mean-squared-error regression. Dropping the weight down-weights the easy small-$t$ steps, which improves sample quality but means the loss is no longer exactly the likelihood bound. That matters for us, because density forecasting cares about calibration and likelihood, not only good-looking samples.

## 6. Algorithms

**Training**

1. Draw $x_0$ from the data, $t \sim \mathcal{U}\{1, \dots, T\}$, and $\epsilon \sim \mathcal{N}(0, I)$.
2. Take a gradient step on $\lVert \epsilon - \epsilon_\theta(\sqrt{\bar\alpha_t}\, x_0 + \sqrt{1 - \bar\alpha_t}\, \epsilon,\ t) \rVert^2$.
3. Repeat until converged.

**Sampling**

1. Draw $x_T \sim \mathcal{N}(0, I)$.
2. For $t = T, \dots, 1$: draw $z \sim \mathcal{N}(0, I)$ if $t > 1$, else set $z = 0$, and update
$$
x_{t-1} = \frac{1}{\sqrt{\alpha_t}} \left( x_t - \frac{\beta_t}{\sqrt{1 - \bar\alpha_t}}\, \epsilon_\theta(x_t, t) \right) + \sigma_t z .
$$
3. Return $x_0$.

Ho et al. found $\sigma_t^2 = \beta_t$ and $\sigma_t^2 = \tilde\beta_t$ give similar results. They are the two extremes: optimal when $x_0$ is pure noise, and optimal when $x_0$ is a single fixed point.

## 7. Noise schedules

- **Linear** (Ho et al.): $\beta_t$ rises linearly from $10^{-4}$ to $0.02$ with $T = 1000$.
- **Cosine** (Nichol and Dhariwal, 2021): define $\bar\alpha_t$ directly,
$$
\bar\alpha_t = \frac{f(t)}{f(0)}, \qquad f(t) = \cos^2\!\left( \frac{t/T + s}{1 + s} \cdot \frac{\pi}{2} \right), \qquad s = 0.008 .
$$
It destroys information more gradually, so fewer steps are wasted on near-pure noise.

## 8. Connection to score matching

From the closed form, the score of the noised marginal given $x_0$ is

$$
\nabla_{x_t} \log q(x_t \mid x_0) = -\frac{x_t - \sqrt{\bar\alpha_t}\, x_0}{1 - \bar\alpha_t} = -\frac{\epsilon}{\sqrt{1 - \bar\alpha_t}} .
$$

So predicting the noise is the same as estimating the score, up to scale:

$$
\epsilon_\theta(x_t, t) \approx -\sqrt{1 - \bar\alpha_t}\; \nabla_{x_t} \log p_t(x_t).
$$

This links DDPM to score-based models and to the continuous-time SDE view (Song et al., 2021), which matters later because the SDE view gives an **exact log-likelihood** through the probability-flow ODE.

## 9. Why this fits time-series density estimation

Let $c$ be the observed history (a window of past values and covariates) and $y$ the future window. We want the conditional distribution $p(y \mid c)$, not just a point forecast.

- **Conditioning:** use $x_0 = y$ and give the network the history, $\epsilon_\theta(x_t, t, c)$. Nothing else in the derivation changes.
- **No parametric form:** the forecast distribution is whatever the samples look like. Fat tails, skew, multimodality and cross-asset dependence come for free if the model learns them.
- **From samples to a density:** draw $y^{(1)}, \dots, y^{(N)} \sim p_\theta(\cdot \mid c)$, then read off quantiles, Value-at-Risk and expected shortfall, or smooth with a kernel density estimate.
- **Scoring:** proper scoring rules evaluate the whole distribution, such as CRPS for one variable and the energy score for several. On synthetic data with a known density (for example GARCH or BEKK), the error against the true density can be measured exactly.

## 10. Open questions to work through

- Does dropping the weight in $L_{\text{simple}}$ hurt calibration of the forecast density, and should the full bound be used instead?
- How many samples $N$ are needed for stable tail quantiles, and how does the error scale with dimension?
- KDE on samples versus the exact probability-flow likelihood: which is the better density estimate here?
- How should non-stationarity be handled: normalize each window, or condition on volatility features?
- Should diffusion run on raw returns or in a learned latent space?

## References

- Ho, J., Jain, A., and Abbeel, P. (2020). Denoising Diffusion Probabilistic Models. NeurIPS.
- Sohl-Dickstein, J., Weiss, E., Maheswaranathan, N., and Ganguli, S. (2015). Deep Unsupervised Learning using Nonequilibrium Thermodynamics. ICML.
- Nichol, A., and Dhariwal, P. (2021). Improved Denoising Diffusion Probabilistic Models. ICML.
- Song, Y., Sohl-Dickstein, J., Kingma, D. P., Kumar, A., Ermon, S., and Poole, B. (2021). Score-Based Generative Modeling through Stochastic Differential Equations. ICLR.
- Rasul, K., Seward, C., Schuster, I., and Vollgraf, R. (2021). Autoregressive Denoising Diffusion Models for Multivariate Probabilistic Time Series Forecasting. ICML.
- Tashiro, Y., Song, J., Song, Y., and Ermon, S. (2021). CSDI: Conditional Score-based Diffusion Models for Probabilistic Time Series Imputation. NeurIPS.
