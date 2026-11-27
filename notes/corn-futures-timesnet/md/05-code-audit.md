---
title: Code audit and changes
order: 5
---

# Code audit and changes

After the course project, the code was moved into a package and audited against the paper, the data and the upstream TimesNet implementation. The code that produced the paper's results is preserved as the tag `paper-2024`. The paper's figures have not been regenerated with the corrected code, so they describe the tagged version.

## 1. Defects that changed the method

### 1.1 Temperature scaling was never applied

The temperature was a separate `nn.Parameter` that was not passed to the optimizer, so it stayed at $T = 1$ throughout training. Test predictions did not use it at all. The calibration described in the paper therefore did not take place.

**Fix.** $T$ is now fitted after training on the validation logits by minimizing the log loss (note 3, section 3), stored with the checkpoint and applied at test time. The test step reports log loss and ECE before and after calibration. On a synthetic check (an overfitted model on random data), the fitted temperature was $T = 6.96$ and the ECE fell from $0.262$ to $0.010$, which is the behavior temperature scaling should show.

### 1.2 The signal used constants that do not match the data

The bucket means and variances behind $\mu$, $\sigma$ and the Sharpe ratio were hard-coded. Compared with the empirical values of $\rho$ in each bucket:

| Bucket | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| Hard-coded mean | −0.00105 | −0.00057 | −0.00031 | −0.00010 | 0.00010 | 0.00031 | 0.00057 | 0.00105 |
| Empirical mean | −0.01072 | −0.00452 | −0.00249 | −0.00083 | 0.00085 | 0.00248 | 0.00456 | 0.00965 |
| Hard-coded "variance" | 0.0053 | 0.0179 | 0.0306 | 0.0379 | 0.0379 | 0.0306 | 0.0179 | 0.0053 |
| Empirical variance | 2.7e−5 | 4.6e−7 | 2.9e−7 | 2.2e−7 | 2.1e−7 | 2.8e−7 | 4.8e−7 | 1.4e−5 |

The means were about 10 times too small, the variances 3 to 5 orders of magnitude too large, and the variance pattern was inverted (the outer, unbounded buckets must have the largest within-bucket variance). The resulting $\sigma$ was between 0.07 and 0.19 for a quantity whose standard deviation is about 0.006, so the Sharpe ratios were tiny and the threshold $\kappa = 0.001$ acted on that distorted scale. The empirical row uses the full sample for illustration; the code now estimates the moments from the training period only.

**Fix.** $m_k$ and $s_k$ are estimated from the training labels and their realized returns, through a method that returns each window's label together with its $\rho_{t+10}$.

### 1.3 The predictive variance dropped the between-bucket term

The code computed $\sigma^2 = \sum_k q_k v_k$, the first term of the law of total variance, and omitted $\sum_k q_k (m_k - \mu)^2$. That term is the part of $\sigma$ that reflects how uncertain the model is about the bucket.

**Fix.** $\sigma^2 = \sum_k q_k s_k - \mu^2$ (note 3, section 4), checked in the tests against the explicit two-term decomposition.

## 2. Look-ahead and leakage in the evaluation

### 2.1 Batch-level period selection (fixed)

Consecutive test windows shared a batch and its periods, so future inputs, including the value that defines a window's label, entered the period choice (note 4, section 2).

**Fix.** Periods and their aggregation weights are selected for each window from its own spectrum. For a single window the new code returns exactly the original output; for a batch of five random windows the outputs differed by up to 0.025, which is the cross-sample dependence that was removed. Tests check that predictions are invariant to the other windows in the batch and to their order.

### 2.2 Incomplete evaluation batches (fixed)

The validation and test loaders dropped the last incomplete batch, so 352 of 368 validation windows and 736 of 742 test windows were scored, and early stopping and the temperature were fitted on the reduced validation set. **Fix.** Validation and test now score every window; training still drops its last incomplete batch.

### 2.3 No purge between splits (open)

The last 10 windows of each split are labeled with returns from the next split (note 4, section 4).

## 3. Where the paper and the code differ

| Paper | Code and data |
|---|---|
| "takes the last 50 days … as input" | The input is 60 days. The paper's own sample counts confirm this: 60 days give 2,561 / 368 / 742 samples, while 50 days would give 2,571 training samples. |
| "predicts the price for the next 10 days" | The model predicts one bucket of the geometric-average daily return over the next 10 days (note 1, section 3). |
| "positional embeddings were set to be learned" | `learned` makes the calendar embedding trainable; the positional encoding stays sinusoidal (note 2, section 1). |
| 742 test samples | The paper run scored 736 test and 352 validation windows (section 2.2). |
| Not mentioned | The upstream per-window normalization was removed (note 2, section 5). |
| Not mentioned | The learning rate is cut to 1/16 after epoch 5 (note 4, section 6). |
| Not mentioned | An auxiliary up/down loss was computed in the training loop but never added to the objective; it has been removed. |

## 4. Verified as correct

- **No feature/label overlap:** $\rho_t$ uses days $t-9, \dots, t$ and the label uses days $t+1, \dots, t+10$.
- **Train-only scaling:** feature scaling is fitted on the training rows only.
- **Unchanged TimesBlock:** the FFT period selection, Inception blocks and aggregation match the upstream code.
- **Labels match their definition:** every `group` label equals the bucket of $\rho$ under the stated edges.

## 5. Repository changes

- The code was reorganized from the upstream script layout into the `cornfutures` package, with settings in `configs/default.yaml`, a test suite and continuous integration.
- Unused upstream modules (other models, M4 and UEA loaders) and an unused heavy dependency were removed.
- A NumPy 2 incompatibility in early stopping (`np.Inf`) was fixed.
- Code that never affected the forward pass or the objective was removed: the decoder input, the mixed-precision and attention-output branches, and the unused auxiliary loss.
- Settings that were taken silently from script defaults (`freq`, `top_k`, `num_kernels`, `lradj`) are now listed in `configs/default.yaml` with the values in effect.
- Datasets, checkpoints and internship materials are excluded from version control.

## 6. Next steps

1. Purge 10 days between the splits.
2. Retrain, and report log loss, RPS and ECE against the unconditional and majority-bucket baselines, with uncertainty that accounts for $n_{\text{eff}}$.
3. Compare with an MSE-trained TimesNet and with simple models on the same features (multinomial logistic regression, gradient boosting).
4. Backtest with a defined holding rule, costs and a validated threshold.
