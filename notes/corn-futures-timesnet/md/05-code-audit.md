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

### 1.4 The value embedding wrapped the window around

The value embedding is a convolution with kernel size 3 over the 60 days of the window, and it used circular padding. The first embedded day was therefore computed from the newest day of the window together with days 1 and 2, and the last embedded day, the most recent one, from days 59 and 60 together with the oldest day. The two ends of the lookback were joined although they are 60 trading days apart. The paper's results were produced with circular padding (tag `paper-2024`).

**Fix.** The convolution is padded causally: two copies of the first day on the left and none on the right, so each embedded day depends only on itself and the two days before it (note 2, section 1). The parameters are unchanged. `token_padding: circular` restores the old embedding, and tests check that it reproduces the previous model output exactly and that with causal padding a change in the last day leaves every earlier embedded day unchanged.

### 1.5 The learning-rate schedule was applied every fifth epoch

`lradj = type1` sets the rate after epoch $e$ to $\eta_0\, 0.5^{\,e-1}$, and upstream TimesNet applies it after every epoch. The training loop here applied it only after epochs 5, 10, …, so the rate stayed at $\eta_0 = 10^{-2}$ for five epochs and then fell to $\eta_0 / 16$ in a single step (note 4, section 6). With early stopping at patience 3 and the best checkpoint at epoch 6, the paper run trained almost entirely at the initial rate. The paper's results used this schedule (tag `paper-2024`).

**Fix.** The schedule is applied after every epoch, as upstream: epochs 1 and 2 use $\eta_0$ and the rate halves after each later epoch (note 2, section 4). `lradj_every_epoch: false` restores the old schedule, and tests check that it reproduces the previous learning-rate sequence.

## 2. Look-ahead and leakage in the evaluation

### 2.1 Batch-level period selection (fixed)

Consecutive test windows shared a batch and its periods, so future inputs, including the value that defines a window's label, entered the period choice (note 4, section 2).

**Fix.** Periods and their aggregation weights are selected for each window from its own spectrum. For a single window the new code returns exactly the original output; for a batch of five random windows the outputs differed by up to 0.025, which is the cross-sample dependence that was removed. Tests check that predictions are invariant to the other windows in the batch and to their order.

### 2.2 Incomplete evaluation batches (fixed)

The validation and test loaders dropped the last incomplete batch, so 352 of 368 validation windows and 736 of 742 test windows were scored, and early stopping and the temperature were fitted on the reduced validation set. **Fix.** Validation and test now score every window; training still drops its last incomplete batch.

### 2.3 Split boundaries (no purge needed)

An earlier version of this note listed a missing purge between the splits, claiming that the last 10 windows of each split are labeled with returns from the next split. That is wrong. The label of a window ending at row $t$ is $\rho$ at row $t+h+\ell$, a backward-looking average of the returns of rows $t+\ell+1, \dots, t+h+\ell$, and each split stops its windows so that its last label row is its own last row. Training labels therefore never use returns from the validation period, and validation labels never use returns from the test period. Validation and test windows take the 60 rows before their split as input, which is past information, not leakage. No purge is needed (note 4, section 4).

## 3. Defects in the data

A second audit checked `inputv3.csv` and the first cleaned dataset against public sources: Yahoo `ZC=F` daily prices, the CBOT holiday calendar and contract rules, USDA WASDE release dates and the CFTC release schedules. The dataset is now rebuilt as `corn_clean_v2.csv` (note 1; column definitions and timing in `data/README.md`).

### 3.1 CFTC reports used before their release (fixed)

The positions were lagged by a fixed rule, "first trading day after the as-of date plus three days". During the 2013-10 and 2018-12 to 2019-03 government shutdowns and in some holiday weeks the CFTC published days to weeks late, so 70 rows used a report that was not yet public. The same rule kept 671 Friday rows one day staler than needed: the report comes out on Friday at 3:30 pm ET, after the close, so with execution at the next close it can already drive the Friday decision.

**Fix.** The release date of every report comes from a table shipped with the package (`cornfutures/data/resources/`): the CFTC release schedule pages for 2010-2021 (archived in the Wayback Machine), the CFTC announcements for the shutdowns and holiday weeks, and for 2006-2009 the third federal business day after the as-of date, a rule that reproduces 618 of the 624 archived dates and is never earlier than the published one. A report is used from the first trading row on or after its release. The CFTC corrected five weeks after their release (as-of 2010-08-17, 2012-11-27, 2017-03-28, 2019-03-26 and 2019-10-15); the public data holds only the corrected figures, so the previous report stays in effect until the correction was published (17 rows).

### 3.2 Wrong vendor prices in 2021 (fixed)

From 2021-05-17 to 2021-09-02 the vendor curve has no Jul 2021 contract and drops the Sep 2021 contract two weeks before it expires, so every price-derived column, the target included, is wrong on those 77 rows. **Fix.** The dataset ends on 2021-05-14.

### 3.3 The target followed the expiring contract into delivery (fixed)

The vendor's front contract stays the expiring one until its last trading day, two weeks into the delivery month. 22.5% of the rows lie within 15 calendar days of that day, where the front has on average 6.7% of the volume of the three nearest contracts: the target described a contract that a speculative position would not hold.

**Fix.** The target is the 10-day geometric-average daily return of the contract held under a stated rule: the front until 10 business days before its first notice day, then the next contract (note 1, sections 1 and 3). Its daily returns are reconstructed exactly from the vendor's target and `contango`, and they equal Yahoo's on all 3,666 rows where Yahoo has both days. On the rows within 15 days of the vendor front's expiry, the standard deviation of daily log returns is 0.0185 for the held contract against 0.0196 for the vendor front (0.0173 on the other rows); the largest daily move while the position holds the second contract falls from 0.104 to 0.085. A `roll_day` column marks the closes at which the position switches contract, so the backtest can charge the roll.

The reconstruction also showed that the vendor's target from 2015-04-06 to 2015-04-16 does not match the vendor's own prices (by up to 0.047 in the 10-day log return); the rebuilt target uses Yahoo's returns there. An earlier version chose between the vendor and Yahoo by a majority vote over 50 rows on either side, so up to 2015-06-12 a row's return depended on later data; the returns are now built causally and match the earlier ones to within $10^{-15}$.

### 3.4 Days to expiry from a broken input (fixed)

`time_to_expiry` was inferred from `holding_cost`, whose cents/dollars mismatch makes 99.99% of its variance the term $\log(100)\cdot 360/\tau$ and which subtracts the unpublished T-bill average of the current month. **Fix.** `time_to_expiry` counts calendar days to the last trading day of the held contract from the CBOT calendar, and a new `days_to_roll` counts business days to the next roll. Both are known in advance.

### 3.5 Calendar (fixed and reported)

The row of 2007-01-02 is a day CBOT was closed (national day of mourning for President Ford); it is dropped and its return merged into the next row. Six trading days are missing from the original data (2007-12-14, 2008-03-14, 2008-12-12, 2009-03-13, 2010-05-14 and 2009-07-02); they are kept as gaps, and the reconstruction treats the next row's return as the two-day return the vendor's target already used.

### 3.6 WASDE balance sheet late or stale (fixed)

`supply_and_demand` is the world corn ratio (total domestic use $-$ exports $+$ imports) / production for the projected marketing year, from the WASDE "World Corn Supply and Use" table. The vendor's values match this definition, within the rounding of the published figures, in all 172 reports checked as first published, and none is used before its release. But the vendor never took up three reports (August 2009, February 2017 and May 2017: 61 rows kept the previous month's value) and took up five one or two days after their release (2006-10-12, 2006-11-09, 2010-02-09, 2016-04-12 and 2016-05-10: 6 rows).

**Fix.** The column is rebuilt from the 180 archived WASDE issues of 2006-09 to 2021-08 (`cornfutures/data/resources/wasde_world_corn.csv`, rebuilt from USDA ESMIS by `scripts/fetch_wasde.py`). A report counts from its release row if it came out before the CBOT close (8:30 am ET until 2012, noon ET from 2013), otherwise from the next trading row. The reissues of 2008-12-12 and 2018-12-14 count from their own release, and the previous report stays in effect through the cancelled October 2013 and January 2019 reports. On the other rows the rebuilt values differ from the vendor's by at most $1.7 \times 10^{-5}$, within the rounding of the published figures.

### 3.7 Verified

- **Rate:** `rate` equals `DFF` of the previous calendar day except on Columbus Day and Veterans Day, when CBOT trades but the Fed is closed: that rate is published only on the next Fed business day, so 27 rows used it a day early (8 with a different value, up to 61 bp on 2008-10-13). `rate` is now the latest rate published by the morning of $t$, on Fed business days that match the New York Fed's EFFR dates from 2006 to 2021.
- **Vendor switch days:** all 72 switches of the vendor's front in the kept period fall on the first trading day after the contract's last trading day.

## 4. Where the paper and the code differ

| Paper | Code and data |
|---|---|
| "takes the last 50 days … as input" | The input is 60 days. The paper's own sample counts confirm this: 60 days give 2,561 / 368 / 742 samples, while 50 days would give 2,571 training samples. |
| "predicts the price for the next 10 days" | The model predicts one bucket of the geometric-average daily return over the next 10 days (note 1, section 3). |
| "positional embeddings were set to be learned" | `learned` makes the calendar embedding trainable; the positional encoding stays sinusoidal (note 2, section 1). |
| 742 test samples | The paper run scored 736 test and 352 validation windows (section 2.2). |
| Not mentioned | The upstream per-window normalization was removed (note 2, section 5). |
| Not mentioned | The learning rate was cut to 1/16 after epoch 5 (section 1.5). |
| Not mentioned | The value embedding was padded circularly (section 1.4). |
| Not mentioned | An auxiliary up/down loss was computed in the training loop but never added to the objective; it has been removed. |

## 5. Verified as correct

- **No feature/label overlap:** $\rho_t$ uses days $t-9, \dots, t$ and the label uses days $t+1, \dots, t+10$.
- **Train-only scaling:** feature scaling is fitted on the training rows only.
- **Unchanged TimesBlock:** the FFT period selection, Inception blocks and aggregation match the upstream code.
- **Labels match their definition:** every `group` label equals the bucket of $\rho$ under the stated edges.

## 6. Repository changes

- The code was reorganized from the upstream script layout into the `cornfutures` package, with settings in `configs/default.yaml`, a test suite and continuous integration.
- Unused upstream modules (other models, M4 and UEA loaders) and an unused heavy dependency were removed.
- A NumPy 2 incompatibility in early stopping (`np.Inf`) was fixed.
- Code that never affected the forward pass or the objective was removed: the decoder input, the mixed-precision and attention-output branches, and the unused auxiliary loss.
- Settings that were taken silently from script defaults (`freq`, `top_k`, `num_kernels`, `lradj`) are now listed in `configs/default.yaml` with the values in effect.
- Datasets, checkpoints and internship materials are excluded from version control.

## 7. Next steps

1. Retrain on `corn_clean_v2.csv`, and report log loss, RPS and ECE against the unconditional and majority-bucket baselines, with uncertainty that accounts for $n_{\text{eff}}$.
2. Compare with an MSE-trained TimesNet and with simple models on the same features (multinomial logistic regression, gradient boosting).
3. Backtest with a defined holding rule, costs including the roll cost on `roll_day`, and a validated threshold.
