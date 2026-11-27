---
title: Overview, data and target
order: 1
---

# Corn futures forecasting with a probabilistic TimesNet

This project forecasts CME corn futures with a modified TimesNet. The model does not regress a point value. It predicts a probability distribution over return buckets, and that distribution is turned into a long, short or flat position. The work began as an internship project and was rewritten as the final project for Introduction to Deep Learning at Seoul National University (Fall 2024). These notes document the method, the mathematics behind each choice, the problems that appear when a model built for generic time series is applied to financial returns, and the corrections made after a code audit.

| Note | Content |
|---|---|
| 1 | Data, features, target and sample design (this page) |
| 2 | Model: TimesNet and the classification head |
| 3 | Probabilistic forecast, calibration and the trading signal |
| 4 | Problems of applying TimesNet to financial returns |
| 5 | Code audit: what was wrong and what changed |

## 1. Data

Daily observations from **2006-09-27 to 2021-09-02** (3,758 rows). The raw inputs were the corn futures curve (prices and volumes of the three nearest contracts and the spot price), soybean and wheat futures, USDA WASDE balance-sheet data, CFTC Commitments of Traders positions and the 3-month Treasury bill rate (FRED `TB3MS`). The dataset is not published because part of it comes from licensed market data.

## 2. Features

Let $F^{(1)}_t, F^{(2)}_t, F^{(3)}_t$ be the prices of the first, second and third nearest corn contracts, $V^{(i)}_t$ their volumes, $S_t$ the corn spot price, $P^{soy}_t$ and $P^{wht}_t$ the soybean and wheat futures prices, $\tau_t$ the days to expiry of the front contract and $r^f_t$ the T-bill rate. Each feature is a ratio or a log-ratio, chosen so that it is closer to stationary than the price levels.

| Feature | Definition | Economic meaning |
|---|---|---|
| `supplement` | $\log\left(F^{(1)}_t \big/ \sqrt{P^{soy}_t P^{wht}_t}\right)$ | Corn price relative to the geometric mean of two substitute crops |
| `contango` | $\log\left(F^{(1)}_t / F^{(2)}_t\right)$ | Slope of the front of the curve (contango vs. backwardation) |
| `holding_cost` | $\dfrac{\log(F^{(1)}_t / S_t)}{\tau_t/360} - r^f_t$ | Annualized basis net of the financing rate, a proxy for storage cost minus convenience yield |
| `supply_and_demand` | Domestic consumption / production (WASDE) | Tightness of the balance sheet |
| `positions` | $\log(\text{long} / \text{short})$ of CFTC managed money | Speculative positioning |
| `on_the_run` | $V^{(1)}_t / (V^{(1)}_t + V^{(2)}_t + V^{(3)}_t)$ | Share of volume in the front contract, which falls around the roll |
| `corn_future_10d_return` | $\rho_t$ below | Recent momentum |

## 3. Target

With $R_t = F^{(1)}_t / F^{(1)}_{t-1} - 1$ the daily return of the front contract, define the **geometric-average daily return over the 10 days ending at $t$**:

$$\rho_t = \left(\prod_{i=0}^{9} (1 + R_{t-i})\right)^{1/10} - 1 .$$

A sample uses the 60 days of features ending at day $t$ and is labeled with the bucket of $\rho_{t+10}$, which covers days $t+1, \dots, t+10$. The input $\rho_t$ covers days $t-9, \dots, t$, so feature and label windows do not overlap.

The target is an average *daily* return over the next 10 days, not a 10-day cumulative return and not a price path.

## 4. Buckets

$\rho$ is discretized into $K = 8$ buckets with fixed edges

$$-0.00587,\ -0.00344,\ -0.00163,\ 0,\ 0.00163,\ 0.00344,\ 0.00587,$$

so bucket $k$ is $\{\rho : e_{k} < \rho \le e_{k+1}\}$ with $e_0 = -\infty$ and $e_8 = +\infty$. The edges are symmetric around zero. The share of rows in each bucket differs across the three periods, which matters for evaluation (note 4):

| Split | Bucket 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| Train | 0.144 | 0.100 | 0.102 | 0.120 | 0.122 | 0.129 | 0.124 | 0.158 |
| Validation | 0.069 | 0.088 | 0.130 | 0.202 | 0.188 | 0.188 | 0.117 | 0.019 |
| Test | 0.073 | 0.100 | 0.136 | 0.157 | 0.148 | 0.116 | 0.119 | 0.152 |

## 5. Splits

The series is split chronologically, 70 / 10 / 20 by rows. Every split also carries the 60 rows before its first label as input history, and the last 10 rows of each split have no label inside the split. That gives 2,561 training, 368 validation and 742 test samples. Features are standardized with the mean and standard deviation of the training rows only.

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
