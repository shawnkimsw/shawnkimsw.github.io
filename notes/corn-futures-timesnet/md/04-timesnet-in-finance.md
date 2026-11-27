---
title: Problems of applying TimesNet to financial returns
order: 4
---

# Problems of applying TimesNet to financial returns

TimesNet was designed and benchmarked on series such as electricity load, traffic and weather, which have strong, stable periodicity and a high signal-to-noise ratio. Daily futures returns have neither. This note sets out, with the relevant mathematics, where the assumptions break and what each problem does to the results. Items 2 and 4 are look-ahead or leakage issues in the evaluation; the others reduce what the model can learn or what its scores mean.

## 1. Period detection on noise

TimesNet chooses its periods from the largest Fourier amplitudes of the input. For a stationary series with spectral density $g(\lambda)$, the periodogram at the Fourier frequencies $\lambda_f = 2\pi f / L$ satisfies

$$I(\lambda_f) = \frac{1}{2\pi L}\left|\sum_{t=1}^{L} x_t e^{-i \lambda_f t}\right|^2 \;\xrightarrow{d}\; g(\lambda_f)\, \frac{\chi^2_2}{2},$$

independently across $f$ (excluding $0$ and the Nyquist frequency). For returns, $g$ is close to flat, so the ordinates are approximately i.i.d. exponential with the same mean, and the $k$ largest are located at random. The selected periods then describe the noise of the current window, not a property of the market, and the fold-and-convolve step builds 2-D structure along arbitrary periods.

Two further details make the "periods" even less interpretable here:

- The FFT is applied to the *embedded* sequence $Z^{(0)} = W_{\text{pred}} E^\top$, not to prices. $E$ contains the sinusoidal positional encoding, which injects deterministic sinusoids, and $W_{\text{pred}}$ mixes all time steps with learned weights.
- The sequence includes the 10 horizon positions created by $W_{\text{pred}}$, which are linear extrapolations rather than observed data.

## 2. Batch-level period selection leaks information across samples

The amplitudes used for period selection are averaged over the batch:

$$A_f = \frac{1}{B\,d}\sum_{b=1}^{B}\sum_{c=1}^{d} \left|\mathrm{FFT}(Z_{b,\cdot,c})_f\right| .$$

Every sample in a batch is folded along the same periods, chosen with all the other samples. This has two consequences.

- **The prediction for a window depends on which other windows share its batch.** The model is not a function of its own input.
- **Look-ahead in validation and test.** Those loaders do not shuffle, so a batch of 32 holds 32 consecutive windows. The window that ends 10 days after window $i$ contains $\rho_{t+10}$ among its inputs, and $\rho_{t+10}$ is exactly the value whose bucket is window $i$'s label. That future value therefore enters window $i$'s period selection. The channel is narrow, because only a discrete choice of $k$ periods is affected, but the evaluation is not strictly out of sample.

The code now selects the periods and their weights for each sample from its own spectrum (note 5). For a batch of one window the output is identical to the original implementation, and tests confirm that a window's prediction no longer changes with the other windows in its batch or with their order.

## 3. Non-stationarity and normalization

Upstream TimesNet normalizes each window by its own mean and standard deviation and reverses the transformation at the output (the Non-stationary Transformer scheme), so that it forecasts shape rather than level. This project removes that step and standardizes every feature with training-period statistics. That keeps the level information the features are designed to carry (for example a steep contango or crowded positioning), but it assumes the training-period distribution still holds later. The labels of the paper's 70 / 10 / 20 split show it does not: the top bucket is 15.8% of training rows but 1.9% of validation rows, and the middle buckets rise from about 12% to about 20%. Early stopping and the temperature are both fitted on that validation period, so both inherit its regime. The default split is now 60 / 20 / 20, which doubles the validation period and starts it in July 2015 instead of December 2016 (note 1, section 5).

## 4. Overlapping labels and the effective sample size

Each label averages 10 daily returns, and consecutive labels share 9 of them. In the data the lag-1 autocorrelation of $\rho$ is $0.907$, the lag-5 autocorrelation is $0.511$ and the lag-10 autocorrelation is $0.028$. Roughly, only one window in ten carries new information, so

$$n_{\text{eff}} \approx \frac{n}{h}: \qquad 2{,}561 \text{ training windows} \to \approx 256, \qquad 742 \text{ test windows} \to \approx 74 .$$

The standard error of an accuracy near the base rate $p = 1/8$ is then

$$\mathrm{SE} \approx \sqrt{\frac{p(1-p)}{n_{\text{eff}}}} \approx \sqrt{\frac{0.125 \times 0.875}{74}} \approx 0.038$$

on the test set. Top-1 accuracies a few points above $1/8$ are within noise.

Overlap does not leak across the split boundaries here. The label of a window ending at $t$ is $\rho$ at row $t+h+\ell$ (with execution lag $\ell$), a backward-looking average of returns up to that row, and each split ends its windows so that its last label row is its own last row. The last training labels therefore use only training-period returns, and the first validation label starts after the training period ends, so no purge between the splits is needed. A purge (López de Prado, 2018) is required when labels look forward from the last training row, which is not the case for this loader.

## 5. Capacity against information

The model has 18.8 million parameters, 99.8% of them in the Inception kernels, against roughly 256 independent training observations. The information available is small: $\rho_{t+10}$ falls in the same bucket as $\rho_t$ 15.6% of the time, against 12.6% for two independent draws from the bucket frequencies, and the lag-10 correlation is $0.028$. Validation accuracy peaking at epoch 6 and falling afterwards is what this ratio predicts. The trainable calendar tables add risk of their own: a day-of-month embedding with 32 entries can fit seasonality that does not exist.

## 6. Learning-rate schedule

The run uses Adam with an initial learning rate of $10^{-2}$; the upstream default, which the TimesNet reference scripts keep, is $10^{-4}$. The schedule `type1` sets $\eta_e = \eta_0\, 0.5^{\,e-1}$ but the paper run called it only every 5 epochs, so the rate stayed at $\eta_0$ for five epochs and then dropped to $\eta_0/16$ at once. Combined with early stopping at patience 3, most of the useful training happened at the large initial rate. The code now applies the schedule after every epoch, as upstream (note 5, section 1.5).

## 7. Evaluation against the wrong baseline

The paper compares top-$N$ accuracy with a uniform random guess, $N/8$. That is the right baseline only if the buckets are equally likely. In the test period the largest bucket holds about 15.7% of rows, so always predicting it beats $1/8$ with no model at all. Fair baselines are:

- the **unconditional forecast** $q_k = \hat P_{\text{train}}(y = k)$ for any probability score;
- the **majority bucket** for top-1 accuracy, and the frequency-ordered buckets for top-$N$.

The buckets are also ordered, and cross-entropy and accuracy ignore that. The ranked probability score compares cumulative distributions and so penalizes a forecast more the further its mass is from the outcome:

$$\mathrm{RPS} = \frac{1}{K-1}\sum_{k=1}^{K-1} \left(Q_k - O_k\right)^2, \qquad Q_k = \sum_{j \le k} q_j, \quad O_k = \mathbf{1}\{y \le k\}.$$

The RPS is strictly proper and suits ordinal targets such as return buckets.

## 8. From forecast to strategy

The signal forecasts an average return over 10 days but is computed every day, so the implied positions overlap, and the threshold $\kappa = 0.001$ on the predictive Sharpe ratio was set when $\sigma$ was mis-scaled (note 5). A credible strategy evaluation needs a defined holding rule (note 3, section 6), transaction and roll costs, a threshold chosen on validation data, and comparison with simple rules such as momentum on $\rho_t$, since $\rho_t$ is itself one of the inputs.

## References

- Brockwell, P. J. and Davis, R. A. (1991). Time Series: Theory and Methods, 2nd ed., chapter 10 (asymptotic distribution of the periodogram). Springer.
- Liu, Y., Wu, H., Wang, J. and Long, M. (2022). Non-stationary Transformers. NeurIPS.
- López de Prado, M. (2018). Advances in Financial Machine Learning, chapter 7 (purging and embargo). Wiley.
- Epstein, E. S. (1969). A Scoring System for Probability Forecasts of Ranked Categories. Journal of Applied Meteorology, 8(6), 985–987.
- Gneiting, T. and Raftery, A. E. (2007). Strictly Proper Scoring Rules, Prediction, and Estimation. JASA, 102(477), 359–378.
