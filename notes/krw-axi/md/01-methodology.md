---
title: Methodology
order: 1
---


# KRW AXI methodology

KRW AXI applies the design of the Across-the-Curve Credit Spread Index (AXI) to the Korean bond market. It measures the cost of borrowing above the risk-free rate across the full maturity curve, using actual transaction data. This page covers the background of AXI, why Korea needs a counterpart, and how KRW AXI is built.

## 1. Background: AXI

### The LIBOR transition

LIBOR was a credit-sensitive rate: it reflected what banks paid to borrow unsecured, so loans that referenced it moved with banks' own funding costs. Its replacement in the US, SOFR, is a secured overnight rate backed by Treasuries. SOFR is close to risk-free, so it carries no credit component. When bank funding stress rises, SOFR does not move with it, and lenders whose costs do rise are left exposed.

### Design

AXI was designed by Antje Berndt, Darrell Duffie and Yichao Zhu (working paper 2020; published 2023) as a credit spread to add on top of SOFR. Its key choices are:

- **Across the curve:** it uses unsecured bank debt with maturities from overnight to five years, rather than only the thin short-term funding markets that underpinned LIBOR.
- **Transaction-based:** spreads come from actual trades, not quotes or expert judgment, which makes the index harder to manipulate.
- **Volume-weighted:** trades are weighted by transaction and issuance volumes, so the index follows where banks actually fund themselves.

### Adoption

| Year | Milestone |
|---|---|
| 2020 | Berndt, Duffie and Zhu publish the AXI working paper |
| 2021 | SOFR Academy announces it will publish AXI and its broader companion, FXI |
| 2022 | Invesco Indexing and SOFR Academy launch the Invesco USD AXI |
| 2022 | Tsinghua University publishes a white paper on AXI for China |
| 2023 | SOFR Academy proposes EURAXI, a euro version alongside €STR |
| 2026 | STOXX launches the iSTOXX USD AXI and FXI with SOFR Academy |

## 2. Why Korea

Korea faces the same gap. KOFR, the Korean counterpart of SOFR, is a risk-free overnight rate computed from repo transactions backed by Korea Treasury Bonds and Monetary Stabilization Bonds. Like SOFR, it carries no credit component.

The gap became visible in 2022. The Bank of Korea raised rates aggressively to fight post-COVID inflation, tightening funding conditions across the corporate sector. Then a major project finance deal, Legoland Korea, defaulted, and liquidity in the money market dried up. KOFR tracked the overnight risk-free rate, but there was no benchmark for stress in long-term borrowing. Existing measures, such as the KOFIA AA- 3-year corporate spread, are quote-based, lag the market and cover a single maturity.

The Chinese AXI, built by Tsinghua University in 2022, showed that the design carries over to other markets. KRW AXI fills the Korean gap with a transaction-based credit spread across the curve, designed to sit on top of KOFR.

| | USD AXI | Chinese AXI | KRW AXI |
|---|---|---|---|
| Risk-free base | SOFR | DR (proposed) | KOFR |
| Risk-free curve | US Treasuries | Chinese government bonds | On-the-run KTBs and MSBs |
| Spread measure | Credit spread over the Treasury curve | Yield spread over the government curve | Z-spread over the spot curve |
| Aggregation | Volume-weighted average | Volume-weighted median by maturity bucket, weighted by issuance share | Volume-weighted median by maturity bucket |
| Frequency | Daily | Monthly | Daily |

## 3. Risk-free term structure

### Notation

All rates use continuous compounding. The curve is parametrized by the instantaneous forward rate $f(t)$.

| Quantity | Definition |
|---|---|
| Discount factor | $P(t) = \exp\left(-\int_0^t f(u)\,du\right)$ |
| Spot rate | $s(t) = \frac{1}{t}\int_0^t f(u)\,du$, so $P(t) = e^{-s(t)\,t}$ |
| Instantaneous forward | $f(t) = -\frac{d}{dt}\ln P(t) = s(t) + t\,s'(t)$ |
| Forward rate from $t_1$ to $t_2$ | $F(t_1, t_2) = \frac{s(t_2)\,t_2 - s(t_1)\,t_1}{t_2 - t_1}$ |
| Bond price | $\sum_j c\,P(t_j) + 100\,P(T)$ |

The spot rate is the average of the forward rates up to $t$, and the forward rate is the marginal rate at $t$.

### Instruments and conventions

The curve is anchored at KOFR at $t = 0$ and fitted to on-the-run Monetary Stabilization Bonds at the short end and on-the-run KTBs out to 10 years. On-the-run issues carry most of the trading volume, so their prices are the most reliable measure of risk-free borrowing.

KTBs pay semiannual coupons and use an Actual/365 day count. Korean bonds are quoted in yield and settle at the dirty price, so the curve is fitted to dirty prices.

### Spline on the forward curve

The forward curve is represented as a cubic spline in a B-spline basis:

$$
f(t) = \sum_i c_i\,B_i(t)
$$

Each $B_i$ is a piecewise cubic that is nonzero over only a few adjacent knot intervals and is already C2 at the knots. Any weighted sum of them is therefore a C2 cubic spline, so continuity does not need to be imposed as separate equations.

The coefficients $c_i$ are chosen so that the model prices the benchmark bonds:

$$
\text{Price}_k = \sum_j \text{CF}_{k,j}\,\exp\left(-\int_0^{t_{k,j}} f(u)\,du\right)
$$

The integral of a B-spline has a closed form, so each model price is cheap to compute. Since prices are nonlinear in $c_i$, the system is solved numerically.

### Why the forward curve

Modeling the forward curve directly keeps it smooth. If the spline is placed on the spot curve instead, the implied forward $f = s + t\,s'$ is one degree less smooth: a quadratic spline on spot rates gives forwards that are continuous but kinked at every knot. Kinks at arbitrary knot dates have no economic meaning and distort anything priced off forward rates.

A spline is preferred over Nelson–Siegel here because the index needs the curve to match current market prices closely. Any fitting error in the risk-free curve would show up directly as a false credit spread. Nelson–Siegel's fixed functional form is interpretable, which suits policy analysis, but it cannot follow local features of the curve.

## 4. Credit spread

For each credit bond trade, the Z-spread is the constant spread $z$ over the spot curve that reprices the trade:

$$
\text{Dirty price} = \sum_j \text{CF}_j\,e^{-(s(t_j) + z)\,t_j}
$$

The Z-spread is used rather than a yield spread because a bond's yield depends on its cash-flow timing along the curve, not only on its credit risk. On an upward-sloping curve, a coupon bond yields less than a zero-coupon bond of the same maturity, so a yield spread would mix coupon effects into the credit measure.

## 5. Aggregation

Trade-level Z-spreads are cleaned and grouped by maturity bucket. Within each bucket, the spreads are aggregated as a volume-weighted median, which keeps large trades influential while limiting the effect of outliers. The index is computed daily.

## References

- Berndt, A., Duffie, D. and Zhu, Y. (2023). *Across-the-Curve Credit Spread Indices.* Financial Markets, Institutions & Instruments 32(3), 115–130.
- XIN Fintech Research Center, Tsinghua University PBC School of Finance (2022). *White Paper on Across-the-Curve Credit Spread Indices (AXI) for China.*
- Bank for International Settlements (2005). *Zero-coupon yield curves: technical documentation.* BIS Papers No 25.
- Fisher, M., Nychka, D. and Zervos, D. (1995). *Fitting the term structure of interest rates with smoothing splines.* Federal Reserve Board, FEDS Working Paper 95-1.
- Anderson, N. and Sleath, J. (2001). *New estimates of the UK real and nominal yield curves.* Bank of England Working Paper No 126.
- Hagan, P. and West, G. (2006). *Interpolation methods for curve construction.* Applied Mathematical Finance 13(2).
