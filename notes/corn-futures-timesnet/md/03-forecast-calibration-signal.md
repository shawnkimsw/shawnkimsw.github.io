---
title: Probabilistic forecast, calibration and the trading signal
order: 3
---

# Probabilistic forecast, calibration and the trading signal

## 1. Why not mean squared error

A regression network trained with MSE solves

$$\min_\theta \frac{1}{N}\sum_{n} \big(y_n - f_\theta(x_n)\big)^2 .$$

This is maximum likelihood under $y \mid x \sim \mathcal{N}(f_\theta(x), \sigma^2)$ with a constant $\sigma$, because

$$-\log p(y \mid x) = \frac{(y - f_\theta(x))^2}{2\sigma^2} + \frac{1}{2}\log(2\pi\sigma^2).$$

The model therefore assumes that conditional returns are Gaussian, have constant variance, and are fully described by their mean. Daily commodity returns violate all three: the tails are fat, volatility clusters, and the useful information may sit in the shape of the distribution (for example skew around harvest reports) rather than in a mean that is close to zero. A point forecast also gives no measure of confidence, which a position-sizing rule needs.

## 2. Cross-entropy over buckets

Discretize the target into $K$ buckets and model $P(y = k \mid x)$ directly with $q_k = \mathrm{softmax}(z)_k$. The average log loss

$$\mathcal{L} = -\frac{1}{N}\sum_n \log q_{y_n}(x_n)$$

is the negative log-likelihood of a categorical model. In expectation over the true conditional distribution $p(\cdot \mid x)$,

$$\mathbb{E}_{y \sim p}\left[-\log q_y\right] = H(p) + \mathrm{KL}(p \,\|\, q),$$

and $H(p)$ does not depend on the model, so minimizing cross-entropy minimizes the KL divergence from the true conditional distribution. The log score is also a strictly proper scoring rule: its expectation is minimized only by reporting $q = p$. The model is therefore rewarded for honest probabilities rather than for a single best guess, and it makes no parametric assumption about the shape of the distribution beyond the bucket grid.

The price is resolution. The distribution is known only up to the bucket edges, and the buckets are treated as unordered: confusing bucket 3 with bucket 4 costs as much as confusing bucket 0 with bucket 7. Note 4 returns to this.

## 3. Calibration by temperature scaling

Deep classifiers trained to low log loss are typically overconfident on new data. Temperature scaling (Guo et al., 2017) fixes this with a single scalar $T > 0$:

$$q_k(x; T) = \frac{\exp(z_k / T)}{\sum_j \exp(z_j / T)} .$$

$T$ is fitted **after training**, with the network frozen, by minimizing the log loss on the validation set:

$$\hat T = \arg\min_{T > 0} \; -\sum_{n \in \text{val}} \log q_{y_n}(x_n; T).$$

The implementation optimizes $\log T$ with L-BFGS so that $T$ stays positive. $T > 1$ flattens the distribution and $T < 1$ sharpens it. Dividing all logits by the same positive number does not change their order, so accuracy and every top-$N$ accuracy are unchanged; only the probabilities move.

Calibration is measured with the expected calibration error. Group the predictions into $M$ bins by their confidence $\hat c_n = \max_k q_k(x_n)$, and compare accuracy with mean confidence in each bin $B_m$:

$$\mathrm{ECE} = \sum_{m=1}^{M} \frac{|B_m|}{N} \left| \mathrm{acc}(B_m) - \mathrm{conf}(B_m) \right| .$$

The test step reports both the log loss and the ECE before and after calibration.

## 4. From a distribution to a signal

The bucket probabilities become a mean and a dispersion through the realized returns inside each bucket. From the **training** samples, estimate for every bucket $k$

$$m_k = \mathbb{E}[\rho \mid y = k], \qquad s_k = \mathbb{E}[\rho^2 \mid y = k], \qquad v_k = s_k - m_k^2 .$$

For a test window with calibrated probabilities $q$, the predictive distribution is the mixture $\sum_k q_k \, P(\rho \mid y = k)$, so

$$\mu = \sum_k q_k\, m_k, \qquad \sigma^2 = \sum_k q_k\, s_k - \mu^2 .$$

By the law of total variance this is

$$\sigma^2 = \underbrace{\sum_k q_k\, v_k}_{\text{within buckets}} + \underbrace{\sum_k q_k\,(m_k - \mu)^2}_{\text{between buckets}} .$$

The second term is what makes $\sigma$ reflect the model's uncertainty: a forecast spread evenly over many buckets has a large between-bucket term, while a forecast concentrated on one bucket has $\sigma$ close to that bucket's own width. (The original code used only the first term, with constants that did not match the data; note 5.)

The signal is the predictive Sharpe ratio of the next-10-day average daily return, with a zero risk-free rate:

$$S = \frac{\mu}{\sigma} .$$

## 5. Position rule

Following the mean-variance logic of Markowitz (1952), take risk only when the expected return is large relative to the uncertainty:

$$\text{position}_t = \begin{cases} +1 & S_t > \kappa \\ -1 & S_t < -\kappa \\ \ \ 0 & \text{otherwise} \end{cases}$$

with $\kappa = 0.001$ in the paper run (`signal_threshold` in the config). The output `signals.csv` holds $\mu$, $\sigma$, $S$ and the position for every test window, and `bucket_moments.csv` holds $m_k$ and $s_k$.

$\kappa$ is only meaningful relative to the scale of $S$. A typical value of $\rho$ has a standard deviation around $0.006$, and the bucket means range from about $\pm 0.0008$ to $\pm 0.01$, so $|S|$ is usually far above $0.001$. With $\kappa = 0.001$ the rule is close to $\operatorname{sign}(\mu)$; a meaningful flat zone needs $\kappa$ chosen on the validation set.

## 6. Evaluating the strategy

The forecast is about days $t+1, \dots, t+10$, but a new forecast arrives every day, so positions overlap. Two consistent ways to turn the positions into a profit-and-loss series are:

- **Daily rebalancing:** hold $\text{position}_t$ for one day, so $\text{PnL}_{t+1} = \text{position}_t \cdot R_{t+1}$.
- **Overlapping tranches:** each day open a tranche of size $1/10$ that is held for 10 days, so the book on any day is the average of the last 10 positions. This matches the forecast horizon.

Both need transaction and roll costs, and both should be compared with buy-and-hold and with a strategy that uses the sign of past returns, so that the forecast is credited only with what it adds. The backtest code of the original project was not kept, so the PnL figure in the paper cannot be reproduced from the repository.

## References

- Guo, C., Pleiss, G., Sun, Y. and Weinberger, K. Q. (2017). On Calibration of Modern Neural Networks. ICML.
- Gneiting, T. and Raftery, A. E. (2007). Strictly Proper Scoring Rules, Prediction, and Estimation. Journal of the American Statistical Association, 102(477), 359–378.
- Naeini, M. P., Cooper, G. and Hauskrecht, M. (2015). Obtaining Well Calibrated Probabilities Using Bayesian Binning. AAAI.
- Markowitz, H. (1952). Portfolio Selection. The Journal of Finance, 7(1), 77–91.
