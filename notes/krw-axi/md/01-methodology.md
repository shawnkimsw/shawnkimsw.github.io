---
title: Methodology
order: 1
---

# KRW AXI methodology

KRW AXI applies the design of the Across-the-Curve Credit Spread Index (AXI) to the Korean bond market. It measures the cost of borrowing above the risk-free rate across the full maturity curve, using actual transaction data. This page covers the background of AXI, why Korea needs a counterpart, and how KRW AXI is built.

## 1. Background: AXI

### 1.1 The LIBOR transition

LIBOR was a credit-sensitive rate: it reflected what banks paid to borrow unsecured, so loans that referenced it moved with banks' own funding costs. Its replacement in the US, SOFR, is a secured overnight rate backed by Treasuries. SOFR is close to risk-free, so it carries no credit component. When bank funding stress rises, SOFR does not move with it, and lenders whose costs do rise are left exposed.

### 1.2 Design

AXI was designed by Antje Berndt, Darrell Duffie and Yichao Zhu (working paper 2020; published 2023) as a credit spread to add on top of SOFR. Its key choices are:

- **Across the curve:** it uses unsecured bank debt with maturities from overnight to five years, rather than only the thin short-term funding markets that underpinned LIBOR.
- **Transaction-based:** spreads come from actual trades, not quotes or expert judgment, which makes the index harder to manipulate.
- **Volume-weighted:** trades are weighted by transaction and issuance volumes, so the index follows where banks actually fund themselves.

### 1.3 Adoption

| Year | Milestone |
|---|---|
| 2020 | Berndt, Duffie and Zhu publish the AXI working paper |
| 2021 | SOFR Academy announces it will publish AXI and its broader companion, FXI |
| 2022 | Invesco Indexing and SOFR Academy launch the Invesco USD AXI |
| 2022 | Tsinghua University publishes a white paper on AXI for China |
| 2023 | SOFR Academy proposes EURAXI, a euro version alongside €STR |
| 2026 | STOXX launches the iSTOXX USD AXI and FXI with SOFR Academy |

## 2. Why Korea

### 2.1 The missing credit benchmark

Korea faces the same gap. KOFR, the Korean counterpart of SOFR, is a risk-free overnight rate computed from repo transactions backed by Korea Treasury Bonds and Monetary Stabilization Bonds. Like SOFR, it carries no credit component.

### 2.2 The 2022 funding stress

The gap became visible in 2022. The Bank of Korea raised rates aggressively to fight post-COVID inflation, tightening funding conditions across the corporate sector. Then a major project finance deal, Legoland Korea, defaulted, and liquidity in the money market dried up. KOFR tracked the overnight risk-free rate, but there was no benchmark for stress in long-term borrowing. Existing measures, such as the KOFIA AA- 3-year corporate spread, are quote-based, lag the market and cover a single maturity.

### 2.3 KRW AXI and other AXI implementations

The Chinese AXI, built by Tsinghua University in 2022, showed that the design carries over to other markets. KRW AXI fills the Korean gap with a transaction-based credit spread across the curve, designed to sit on top of KOFR.

| | USD AXI | Chinese AXI | KRW AXI |
|---|---|---|---|
| Risk-free base | SOFR | DR (proposed) | KOFR |
| Risk-free curve | US Treasuries | Chinese government bonds | On-the-run KTBs and MSBs |
| Spread measure | Credit spread over the Treasury curve | Yield spread over the government curve | Z-spread over the spot curve |
| Aggregation | Volume-weighted average | Volume-weighted median by maturity bucket, weighted by issuance share | Volume-weighted median by maturity bucket |
| Frequency | Daily | Monthly | Daily |

## 3. Data: the Korean government bond market

The risk-free curve is built from two families of bonds: Korea Treasury Bonds (KTBs), issued by the Ministry of Economy and Finance to fund the government, and Monetary Stabilization Bonds (MSBs), issued by the Bank of Korea to absorb excess liquidity. Both are treated as government bonds, and both are eligible collateral for the repo transactions behind KOFR.

### 3.1 Korea Treasury Bonds

KTBs are fixed-rate bonds with semiannual coupons. They are sold through competitive auctions to primary dealers every month, and the coupon rate is set in multiples of 0.125%.

To keep each benchmark large and liquid, KTBs use fungible issuance: a new issue is created only at fixed intervals, and every auction in between reopens the same bond with the same maturity and coupon. The current issue at each maturity is the on-the-run benchmark, and its yield is the market's reference rate for that maturity.

| Maturity | New issues per year | New issue date | Fungible period |
|---|---|---|---|
| 3-year | 2 | 10 June, 10 December | 6 months |
| 5-year | 2 | 10 March, 10 September | 6 months |
| 10-year | 2 | 10 June, 10 December | 6 months |
| 20-year | 1 | 10 September | 1 year |
| 30-year | 1 | 10 March | 1 year |
| 50-year | Every 2 years | 10 September | 2 years |

A 2-year KTB is also auctioned regularly. Each issue is named by its coupon and maturity: KTB 02625-2703, for example, pays a 2.625% coupon and matures in March 2027.

### 3.2 Monetary Stabilization Bonds

MSBs cover the short end of the curve, where KTBs are scarce.

| Maturity | Type | Auction frequency | Fungible period |
|---|---|---|---|
| 91-day | Discount | Weekly | None |
| 1-year | Coupon | Monthly | 3 months |
| 2-year | Coupon | Monthly | 3 months |
| 3-year | Coupon | Monthly | 6 months |

The 91-day MSB is a discount bond with a single cash flow at maturity, so it gives the cleanest observation of the short-end rate. A 182-day MSB was issued regularly until 2021, when it was replaced by the 3-year MSB. In August 2026 the Bank of Korea lengthened the fungible period of the 1-year MSB from two to three months, with new issues on the 1st of March, June, September and December.

### 3.3 Instruments used in the curve

| Maturity | Instrument |
|---|---|
| Overnight | KOFR |
| 3 months | On-the-run 91-day MSB |
| 1 year | On-the-run 1-year MSB |
| 2, 3, 5, 10, 20 and 30 years | On-the-run KTBs |

On-the-run issues carry most of the trading volume, so their prices are the most reliable measure of risk-free borrowing. Off-the-run bonds trade far less and carry a liquidity premium that would bias the risk-free curve upward.

The curve extends to 30 years, so the long-end boundary condition sits far out, where a settled forward curve is easiest to justify. The 50-year KTB is left out: it is issued only every two years in small size and trades thinly, because its main holders, insurers and pension funds, buy it to match long liabilities rather than to trade. Its prices would be too noisy to pin down the end of the curve.

### 3.4 Quoting and settlement

Korean bonds are quoted in yield, and trades settle at the dirty price, including accrued interest. Accrued interest uses an Actual/365 day count. The curve is therefore fitted to dirty prices computed from the quoted yields.

## 4. Risk-free term structure

### 4.1 Notation

All rates use continuous compounding. The curve is parametrized by the instantaneous forward rate $f(t)$.

| Quantity | Definition |
|---|---|
| Discount factor | $P(t) = \exp\left(-\int_0^t f(u)\,du\right)$ |
| Spot rate | $s(t) = \frac{1}{t}\int_0^t f(u)\,du$, so $P(t) = e^{-s(t)\,t}$ |
| Instantaneous forward | $f(t) = -\frac{d}{dt}\ln P(t) = s(t) + t\,s'(t)$ |
| Forward rate from $t_1$ to $t_2$ | $F(t_1, t_2) = \frac{s(t_2)\,t_2 - s(t_1)\,t_1}{t_2 - t_1}$ |
| Bond price | $\sum_j c\,P(t_j) + 100\,P(T)$ |

The spot rate is the average of the forward rates up to $t$, and the forward rate is the marginal rate at $t$. Each relation is derived in Appendix A.

### 4.2 Spline on the forward curve

The forward curve is represented as a cubic spline: a cubic polynomial on each segment between knots, joined so that the value, slope and curvature are continuous (C2). The spline is determined by the forward values at the knots together with two boundary conditions, and the bootstrap in Section 4.3 solves for those knot values.

Modeling the forward curve directly keeps it smooth. If the spline is placed on the spot curve instead, the implied forward $f = s + t\,s'$ is one degree less smooth: a quadratic spline on spot rates gives forwards that are continuous but kinked at every knot. Kinks at arbitrary knot dates have no economic meaning and distort anything priced off forward rates.

### 4.3 Fitting procedure

The curve is built with QuantLib's piecewise yield curve, which bootstraps a spline so that every benchmark reprices exactly.

1. **Instruments.** Each on-the-run MSB and KTB enters as a fixed-rate bond helper with its full terms: issue and maturity dates, coupon rate, semiannual coupon schedule on the South Korea business-day calendar, Actual/365 day count and settlement lag. Its quoted yield is converted to a dirty price using KTB conventions, and the helper's price type is set to dirty. KOFR enters as an overnight deposit, which anchors the short end of the forward curve.
2. **Curve type.** The curve interpolates instantaneous forward rates, using a cubic spline with the boundary conditions in Section 4.4. The knots are the maturities of the instruments, so each segment runs from one benchmark maturity to the next.
3. **Bootstrap.** Starting from the shortest instrument, QuantLib finds the forward value at each knot that makes that instrument reprice exactly, using Brent's method, a one-dimensional root-finder. Each cash flow is discounted with $P(t) = \exp\left(-\int_0^t f(u)\,du\right)$, and coupons that fall in earlier segments use the part of the curve already built.
4. **Iteration.** A cubic spline is global: setting a later knot slightly moves the earlier segments. QuantLib therefore repeats the full pass, using the latest knot values, until no knot value changes. At convergence, every instrument reprices exactly.
5. **Daily refit.** The curve is refitted every business day on that day's on-the-run prices, matching the daily frequency of the index.
6. **Extrapolation.** Beyond 30 years, the forward curve is held flat at its 30-year value.

### 4.4 Boundary conditions

Exact repricing of the benchmarks leaves two degrees of freedom in a cubic spline, so two additional assumptions are needed, one at each end of the curve:

| End | Condition | Rationale |
|---|---|---|
| Short end | $f''(0) = 0$ | Yield and forward curves usually have a clear slope at the short end, which reflects the expected direction of the next policy moves. Zero curvature is the weakest condition that leaves this slope free. Forcing the slope itself to zero would contradict the shape of observed curves. |
| Long end | $f'(30) = 0$ | The forward curve has zero slope at the last knot, so it joins the flat-forward extension beyond 30 years with no jump in level or slope. This makes extrapolation to longer maturities well defined. |

This follows the design of McCulloch and Kochin (2000), who fit an exact spline to a small number of well-spaced Treasury maturities. Their spline is placed on the log discount function rather than on the forward curve, so their conditions do not carry over exactly, but the reasoning is the same: a flat forward at the long end so that the curve can be extrapolated, and a short end that is left free to slope, because forcing a zero slope at maturity zero is contrary to the upward slope the term structure ordinarily shows there.

**A modeling choice, not a unique answer.** Other conditions are possible. For example, both $f'(30) = 0$ and $f''(30) = 0$ could be imposed at the long end, since the forward from 30 to 31 years should barely differ from the forward from 31 to 32 years. That choice was rejected for two reasons:

- **Stability.** With both conditions at one end, the spline is determined by marching from that end, and an error in one price grows by a factor of about $2 + \sqrt{3} \approx 3.7$ at each knot. A small error in a thinly traded 30-year price would swing the short end, where most credit trades sit. With one condition at each end, the curve is held at both sides, like a string held at both ends, and errors instead shrink by about the same factor away from their source.
- **Implementation.** Standard spline solvers, including QuantLib and SciPy, take one boundary condition per end.

**How central banks handle the same issue.** Central banks that use splines do not impose boundary conditions by hand. They solve an optimization rather than a system of equations: they minimize pricing errors plus a roughness penalty $\lambda \int f''(t)^2\,dt$ on the forward curve, and the penalty itself pushes the curvature to zero at the ends. The Bank of England also increases the penalty with maturity, so its long-end forwards come out nearly flat, which is the same goal as $f'(30) = 0$ here.

**Robustness check.** Because the boundary choice can move the forwards near the ends of the curve, the index should be recomputed with the natural spline, $f''(0) = 0$ and $f''(30) = 0$, to confirm that the spreads do not depend on it.

### 4.5 Comparison of curve-fitting methods

| Method | Used by | How the curve is fitted | Strengths | Weaknesses |
|---|---|---|---|---|
| Nelson–Siegel / Svensson | Federal Reserve, Deutsche Bundesbank, Swiss National Bank, Norges Bank, and most central banks in the BIS survey | Spot curve set by a few parameters for level, slope and one or two humps, chosen to minimize price or yield errors | Smooth and parsimonious; interpretable factors; stable long end | Cannot follow local features of the curve; Svensson parameters can be unstable from day to day |
| Smoothing spline | Bank of England (variable roughness penalty), Bank of Japan, Sveriges Riksbank | Cubic spline on the forward curve; minimizes pricing errors plus a roughness penalty, which the Bank of England increases with maturity | Flexible where the data are rich and smooth where they are sparse | Penalty must be calibrated; bonds are not priced exactly |
| Monotone convex | US Treasury (official yield curve); dealer desks | Bootstraps discrete forwards from benchmark securities and interpolates them with a shape-preserving rule | Exact fit; local; keeps forwards positive | Forward curve is continuous but not smooth |
| Exact-fit cubic spline | This project | Cubic spline on the forward curve, repricing each benchmark exactly | No fitting error enters the spread; smooth forwards | Needs explicit boundary conditions; sensitive to bad prices because it does not smooth |

**Why this project uses an exact-fit spline.** Most central banks use Nelson–Siegel or Svensson because they want a smooth, interpretable curve for policy analysis. KRW AXI has a different goal: any fitting error in the risk-free curve would show up directly as a false credit spread. With a small set of liquid on-the-run benchmarks, an exact fit removes that error, and placing the spline on the forward curve keeps the forwards smooth.

## 5. Credit spread

### 5.1 Z-spread

For each credit bond trade, the Z-spread is the constant spread $z$ over the spot curve that reprices the trade:

$$
\text{Dirty price} = \sum_j \text{CF}_j\,e^{-(s(t_j) + z)\,t_j}
$$

### 5.2 Why the Z-spread

The Z-spread is used rather than a yield spread because a bond's yield depends on its cash-flow timing along the curve, not only on its credit risk. On an upward-sloping curve, a coupon bond yields less than a zero-coupon bond of the same maturity, so a yield spread would mix coupon effects into the credit measure.

## 6. Aggregation

### 6.1 Maturity buckets

Trade-level Z-spreads are cleaned and grouped by maturity bucket.

### 6.2 Volume-weighted median

Within each bucket, the spreads are aggregated as a volume-weighted median, which keeps large trades influential while limiting the effect of outliers.

### 6.3 Frequency

The index is computed daily.

## Appendix A. Derivations of the term structure relations

### A.1 Continuous compounding

With a rate $r$ compounded $n$ times a year, one unit invested today grows to $(1 + r/n)^{nt}$ by time $t$. Since $(1 + r/n)^n \to e^r$ as $n \to \infty$,

$$
\lim_{n \to \infty} \left(1 + \frac{r}{n}\right)^{nt} = e^{rt}
$$

So under continuous compounding, one unit paid at $t$ is worth $e^{-rt}$ today. With the spot rate $s(t)$ for maturity $t$, the discount factor is

$$
P(t) = e^{-s(t)\,t}
$$

### A.2 Forward rate between two dates

Borrowing from today to $t_2$ must cost the same as borrowing to $t_1$ and then rolling over from $t_1$ to $t_2$ at the forward rate $F(t_1, t_2)$ agreed today. Otherwise one could borrow on one route, lend on the other and lock in a riskless profit. In discount factors,

$$
P(t_2) = P(t_1)\, e^{-F(t_1, t_2)\,(t_2 - t_1)}
$$

Taking logs and solving for $F$:

$$
F(t_1, t_2) = \frac{\ln P(t_1) - \ln P(t_2)}{t_2 - t_1}
$$

Substituting $\ln P(t) = -s(t)\,t$ from A.1:

$$
F(t_1, t_2) = \frac{s(t_2)\,t_2 - s(t_1)\,t_1}{t_2 - t_1}
$$

### A.3 Instantaneous forward rate

The instantaneous forward rate $f(t)$ is the forward rate for an infinitesimal period starting at $t$. Letting $t_1 = t$ and $t_2 = t + \Delta$ in A.2,

$$
f(t) = \lim_{\Delta \to 0} \frac{\ln P(t) - \ln P(t + \Delta)}{\Delta} = -\frac{d}{dt} \ln P(t)
$$

Since $\ln P(t) = -s(t)\,t$, the product rule gives

$$
f(t) = \frac{d}{dt}\big[s(t)\,t\big] = s(t) + t\,s'(t)
$$

### A.4 Discount factor from the forward curve

Integrate $f(u) = -\frac{d}{du} \ln P(u)$ from $0$ to $t$, using $P(0) = 1$:

$$
\int_0^t f(u)\,du = -\big[\ln P(t) - \ln P(0)\big] = -\ln P(t)
$$

Therefore

$$
P(t) = \exp\left(-\int_0^t f(u)\,du\right)
$$

Discounting to $t$ is the same as discounting through every instant on the way, each at its own forward rate.

### A.5 Spot rate as the average forward rate

Equating the two expressions for the discount factor, $e^{-s(t)\,t}$ from A.1 and $\exp\left(-\int_0^t f(u)\,du\right)$ from A.4:

$$
s(t)\,t = \int_0^t f(u)\,du \quad\Longrightarrow\quad s(t) = \frac{1}{t}\int_0^t f(u)\,du
$$

The spot rate is the average of the forward rates up to $t$. A.3 is the matching marginal relation: $f(t) = s(t) + t\,s'(t)$, so where the spot curve slopes upward ($s' > 0$), the forward rate lies above the spot rate, just as a marginal cost above the average cost pulls the average up.

### A.6 Bond price

A coupon bond is a portfolio of zero-coupon bonds, one for each cash flow. A cash flow $\text{CF}_j$ paid at $t_j$ is worth $\text{CF}_j\,P(t_j)$ today, so by no-arbitrage the bond is worth the sum of its parts. For a bond with coupon $c$ per period, paid at $t_1, \dots, t_n = T$, and face value 100,

$$
\text{Price} = \sum_{j=1}^{n} c\,P(t_j) + 100\,P(T)
$$

For a KTB with an annual coupon rate $C$ paid semiannually, $c = 100\,C/2$.

## References

- Berndt, A., Duffie, D. and Zhu, Y. (2023). *Across-the-Curve Credit Spread Indices.* Financial Markets, Institutions & Instruments 32(3), 115–130.
- XIN Fintech Research Center, Tsinghua University PBC School of Finance (2022). *White Paper on Across-the-Curve Credit Spread Indices (AXI) for China.*
- Bank for International Settlements (2005). *Zero-coupon yield curves: technical documentation.* BIS Papers No 25.
- Fisher, M., Nychka, D. and Zervos, D. (1995). *Fitting the term structure of interest rates with smoothing splines.* Federal Reserve Board, FEDS Working Paper 95-1.
- Anderson, N. and Sleath, J. (2001). *New estimates of the UK real and nominal yield curves.* Bank of England Working Paper No 126.
- Gürkaynak, R., Sack, B. and Wright, J. (2007). *The U.S. Treasury yield curve: 1961 to the present.* Journal of Monetary Economics 54(8), 2291–2304.
- McCulloch, J. H. and Kochin, L. A. (2000). *The inflation premium implicit in the US real and nominal term structures of interest rates.* Ohio State University Economics Department Working Paper No 98-12.
- Hagan, P. and West, G. (2006). *Interpolation methods for curve construction.* Applied Mathematical Finance 13(2).
