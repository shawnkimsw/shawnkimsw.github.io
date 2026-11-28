---
title: Overview, data and target
order: 1
---

# Corn futures forecasting with a probabilistic TimesNet

This project forecasts CME corn futures with a modified TimesNet. The model does not regress a point value. It predicts a probability distribution over return buckets, and that distribution sizes a long or short position by the Kelly criterion. The work began as an internship project and was rewritten as the final project for Introduction to Deep Learning at Seoul National University (Fall 2024). These notes document the method, the mathematics behind each choice, the problems that appear when a model built for generic time series is applied to financial returns, and the corrections made after a code audit.

| Note | Content |
|---|---|
| 1 | Data, features, target and sample design (this page) |
| 2 | Model: TimesNet and the classification head |
| 3 | Probabilistic forecast, calibration and the trading signal |
| 4 | Problems of applying TimesNet to financial returns |
| 5 | Code audit: what was wrong and what changed |

## 1. Data

The model runs on `data/corn_clean_v2.csv`: one row per CBOT trading day from **2006-10-11 to 2021-05-14** (3,670 rows), built by `scripts/prepare_data.py` from the original dataset (`inputv3.csv`, 3,758 rows from 2006-09-27 to 2021-09-02) and public data. The raw inputs were the corn futures curve (prices and volumes of the three nearest contracts and the spot price), soybean and wheat futures, USDA WASDE balance-sheet data, CFTC Commitments of Traders positions and the effective federal funds rate (FRED `DFF`). The dataset is not published because part of it comes from licensed market data.

Rows are removed where the original data is wrong: everything from 2021-05-17 on (the vendor curve lacks the Jul 2021 contract and drops Sep 2021 early, so every price-derived column is wrong for 77 rows), 2007-01-02 (CBOT was closed for the national day of mourning for President Ford; its return is merged into the next row) and the first 10 rows, which have no full window of held returns. Six trading days are missing from the original data (the last trading days 2007-12-14, 2008-03-14, 2008-12-12, 2009-03-13 and 2010-05-14, and 2009-07-02); the next row then carries a two-day return.

**Timing.** A forecast made after the close of day $t$ is executed at the close of $t+1$. A value enters row $t$ only if it was public before the evening of $t$: prices at the close of $t$, the latest published federal funds rate (the rate of a Fed business day is published on the morning of the next one, so on Columbus Day and Veterans Day, when CBOT trades but the Fed is closed, the row keeps the rate published the business day before), the WASDE on its release day (8:30 am ET until 2012, noon ET from 2013), and a CFTC report from its release date on (normally Friday 3:30 pm ET, after the close, so the Friday row already uses it). The release dates come from the CFTC's published schedules and its announcements for the 2013 and 2018-19 shutdowns and holiday weeks, and the five weeks it corrected after release are used only from the correction; the earlier rule "as-of date plus three days" used unreleased reports on 70 rows and kept 671 Friday rows one day stale.

**Held contract.** Corn trades in Mar, May, Jul, Sep and Dec contracts. For a contract, the last trading day (LTD) is the business day before the 15th of the contract month and the first notice day (FND) is the last business day of the month before. The position holds the front contract until the close **10 business days before FND**, then switches to the next contract. This matches the roll windows of the S&P GSCI (5th to 9th business day of the month before delivery) and BCOM (6th to 10th) and the 10-day holding period, and it leaves the front about four weeks before expiry. The front's share of the volume of the three nearest contracts falls from 0.59 at 25 business days before FND to 0.48 at 10 days, 0.42 at 5 days and 0.33 on the day before FND, then collapses to 0.15 on FND and below 0.05 from four days after it. The vendor keeps the expiring contract as its front until LTD, so between the roll and LTD (40.8% of the rows) the position holds the vendor's second contract.

## 2. Features

Let $F^{(1)}_t, F^{(2)}_t, F^{(3)}_t$ be the prices of the vendor's first, second and third nearest corn contracts, $V^{(i)}_t$ their volumes, $P^{soy}_t$ and $P^{wht}_t$ the soybean and wheat futures prices, and $T_t$ and $R_t$ the last trading day and the roll date of the contract held after the close of $t$. Each feature is a ratio, a log-ratio or a calendar count, chosen so that it is closer to stationary than the price levels.

| Feature | Definition | Economic meaning |
|---|---|---|
| `supplement` | $\log\left(F^{(1)}_t \big/ \sqrt{P^{soy}_t P^{wht}_t}\right)$ | Corn price relative to the geometric mean of two substitute crops |
| `contango` | $\log\left(F^{(1)}_t / F^{(2)}_t\right)$ | Slope of the front of the curve (contango vs. backwardation) |
| `time_to_expiry` | $(T_t - t)$ in calendar days $/\,365$ | Maturity of the held contract |
| `days_to_roll` | Business days from $t$ to $R_t$, $/\,21$ | Position in the roll cycle |
| `rate` | `DFF` last published by the morning of $t$ | Financing rate |
| `supply_and_demand` | World corn (domestic use $-$ exports $+$ imports) / production, projected marketing year (WASDE) | Tightness of the world balance sheet |
| `positions` | $\log(\text{long} / \text{short})$ of CFTC managed money, futures and options | Speculative positioning |
| `options_oi_share` | $1 -$ futures-only / combined open interest | Weight of options in open interest |
| `on_the_run` | $V^{(1)}_t / (V^{(1)}_t + V^{(2)}_t + V^{(3)}_t)$ | Share of volume in the front contract, which falls around the roll |
| `corn_future_10d_return` | $\rho_t$ below | Recent momentum |

`time_to_expiry`, `days_to_roll` and the column `roll_day` (true when the held contract switches at the close of $t$, used by the backtest to charge the roll) follow from the exchange calendar and are known in advance. The original `holding_cost` is not used: a cents/dollars mismatch makes 99.99% of its variance the term $\log(100) \cdot 360/\tau$, a function of days to expiry, and it subtracts the T-bill average of the current month, which is published only after the month ends.

## 3. Target

Let $r_t$ be the daily log return, from the close of the previous row to the close of $t$, of the contract held over that interval. The front's return is Yahoo `ZC=F`'s on the 3,666 rows where Yahoo has both days; on the 14 rows Yahoo lacks, it is recovered from the original 10-day average: with $L_t = 10 \log(1 + \rho^{\text{vendor}}_t)$, $L_t - L_{t-1} = r^{(1)}_t - r^{(1)}_{t-10}$. Each return uses data up to its own row only, and the series agrees with the vendor's 10-day average on every row except 2015-04-06 to 2015-04-16. The second contract's return follows from $\log F^{(2)} = \log F^{(1)} - c$ with $c$ = `contango`:

$$r^{(2)}_t = r^{(1)}_t - (c_t - c_{t-1}),$$

and on the day the vendor moves its front to the held contract, $r_t = r^{(1)}_t + c_{t-1}$. The target is the **geometric-average daily return of the held contract over the 10 days ending at $t$**:

$$\rho_t = \exp\left(\frac{1}{10} \sum_{i=0}^{9} r_{t-i}\right) - 1 .$$

A sample uses the 60 days of features ending at day $t$. With execution at the close of $t+1$ it is labeled with the bucket of $\rho_{t+11}$, which covers days $t+2, \dots, t+11$; the paper run, without the execution lag, used $\rho_{t+10}$. The input $\rho_t$ covers days $t-9, \dots, t$, so feature and label windows do not overlap.

The target is an average *daily* return over the next 10 days, not a 10-day cumulative return and not a price path.

## 4. Buckets

$\rho$ is discretized into $K = 8$ buckets with fixed edges

$$-0.00587,\ -0.00344,\ -0.00163,\ 0,\ 0.00163,\ 0.00344,\ 0.00587,$$

so bucket $k$ is $\{\rho : e_{k} < \rho \le e_{k+1}\}$ with $e_0 = -\infty$ and $e_8 = +\infty$. The edges are symmetric around zero. The share of rows in each bucket differs across the three periods of the paper data (70 / 10 / 20 split), which matters for evaluation (note 4):

| Split | Bucket 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| Train | 0.144 | 0.100 | 0.102 | 0.120 | 0.122 | 0.129 | 0.124 | 0.158 |
| Validation | 0.069 | 0.088 | 0.130 | 0.202 | 0.188 | 0.188 | 0.117 | 0.019 |
| Test | 0.073 | 0.100 | 0.136 | 0.157 | 0.148 | 0.116 | 0.119 | 0.152 |

## 5. Splits

The series is split chronologically by rows, 60 / 20 / 20 by default (`train_ratio` and `test_ratio` in `configs/default.yaml`). Every split also carries the 60 rows before its first label as input history, and the last 10 rows of each split have no label inside the split. On `corn_clean_v2.csv` with the one-day execution lag that gives 2,132 training, 724 validation and 724 test samples: training 2006-10-11 to 2015-07-17, validation 2015-07-20 to 2018-06-15 and test 2018-06-18 to 2021-05-14. Features are standardized with the mean and standard deviation of the training rows only.

The 2024 paper split 70 / 10 / 20, which gives 2,561 training, 368 validation and 742 test samples on the paper data, and 2,499, 357 and 724 on `corn_clean_v2.csv`. Its validation set, 2016-12-30 to 2018-06-15, was small and unusually calm (section 4). The default split moves a tenth of the rows from training to validation, so model selection and the temperature rest on a validation set twice as large and less unusual, while the test rows stay the same.

The hyperparameter search selects on that one validation period, and three years can favor a configuration by regime alone. Before the final test, the five finalists of the search are therefore scored again on three earlier folds with an expanding training window (`scripts/walkforward.py`, dates in `configs/walkforward.yaml`):

| Fold | Training | Validation | Validation samples |
|---|---|---|---|
| F0 | 2006-10-11 to 2011-03-04 | 2011-03-07 to 2012-08-16, the 2011–2012 drought rally | 357 |
| F1 | 2006-10-11 to 2012-08-16 | 2012-08-17 to 2014-01-31 | 357 |
| F2 | 2006-10-11 to 2014-01-31 | 2014-02-03 to 2015-07-17 | 357 |
| F3 | 2006-10-11 to 2015-07-17 | 2015-07-20 to 2018-06-15, the default split | 724 |

Each fold retrains the configuration with two seeds and refits the bucket edges and the feature scaling on its own training rows, so the skill against the unconditional baseline uses that fold's edges and training frequencies; F3 reuses the five seeds of the search. The choice is the highest mean skill over the four folds, each weighted equally as one regime, followed by the one-standard-error rule on the standard error across folds: among the configurations within one standard error of the best, the one with the fewest parameters. The test period, 2018-06-18 to 2021-05-14, is not used.

## 6. Repository layout

```
cornfutures/
├── data/        Dataset_Custom (labels, train-only scaling, label/return alignment), loaders
├── models/      TimesNet with the classification head; embedding and Inception layers
├── training/    training loop, temperature calibration, signal construction
└── utils/       early stopping, learning-rate schedule, metrics
configs/default.yaml   settings of the paper run
scripts/train.py       entry point
tests/                 unit tests (calibration, signal algebra, label alignment, model shapes)
website/notes/         these notes
```

The code is adapted from the TimesNet implementation in `thuml/Time-Series-Library` (MIT License).

## References

- Wu, H., Hu, T., Liu, Y., Zhou, H., Wang, J. and Long, M. (2023). TimesNet: Temporal 2D-Variation Modeling for General Time Series Analysis. ICLR.
- Guo, C., Pleiss, G., Sun, Y. and Weinberger, K. Q. (2017). On Calibration of Modern Neural Networks. ICML.
- Markowitz, H. (1952). Portfolio Selection. The Journal of Finance, 7(1), 77–91.
