# Binomial-Trinomial-Asian-Option-Pricing

> A computational implementation of binomial and trinomial tree models for pricing European and American options, computing Greeks, and simulating dynamic delta hedging of Asian options.

---

## Overview

This project implements tree-based methods for derivatives pricing, covering the full lifecycle of option valuation: from discrete-time binomial trees for European and American options, through QuantLib-powered trinomial trees across the moneyness spectrum, to dynamic delta hedging of path-dependent Asian options using Monte Carlo simulation with a geometric control variate. All model inputs spot price, historical volatility, and risk-free rate are sourced live from market data via yfinance, and the Asian option hedging simulation runs on a real historical price path rather than synthetic data.

| Module         | Method                                       | Options                        | Data Source                        |
| -------------- | -------------------------------------------- | ------------------------------ | ---------------------------------- |
| Binomial Tree  | Cox-Ross-Rubinstein (n=100)                  | European and American call/put | SPY spot + realised vol (yfinance) |
| Trinomial Tree | JarrowRudd via QuantLib (n=200)              | European and American call/put | SPY spot + realised vol (yfinance) |
| Delta Hedging  | Geometric Asian + Arithmetic Asian (MC + CV) | Asian put                      | SPY 6-month historical path        |

---

## Methodology

### Binomial Tree

**Model:** Cox-Ross-Rubinstein (CRR) binomial tree with n=100 steps.

**Parameters:** Spot price S0 and strike K (ATM proxy, nearest $5) sourced from SPY via yfinance. Volatility is calibrated as the annualised standard deviation of daily log-returns over the trailing 63 trading days. The risk-free rate is sourced from the 3-month US T-bill yield (^IRX) via yfinance. Expiry is set to T=3 months.

**Tree construction:**

Up and down factors: `u = exp(sigma * sqrt(dt))`, `d = 1/u`

Risk-neutral probability: `p = (exp(r * dt) - d) / (u - d)`

Backward induction for European options:

```
V(i,j) = exp(-r*dt) * [p * V(i+1,j+1) + (1-p) * V(i+1,j)]
```

For American options, at each node the continuation value is compared against immediate exercise:

```
V(i,j) = max(intrinsic_value, continuation_value)
```

**European Pricing:** Call and put prices are computed at n=100 with a convergence plot across step counts 5 to 200, confirming stability.

**Delta:** First-order finite difference at the root node:

```
Delta = (V_up - V_down) / (S_up - S_down)
```

Call delta is positive (0 to 1); put delta is negative (-1 to 0). Delta represents the instantaneous sensitivity of option value to a unit change in the underlying price and is used directly as the hedge ratio.

**Volatility Sensitivity (Vega):** Prices are recomputed at sigma=25% to measure the impact of a 200 basis point volatility increase. Both calls and puts increase in value with higher volatility; vega is always positive for long option positions.

**American Options:** The American put carries an early exercise premium over the European put due to the value of exercising before expiry when deeply in-the-money. The American call on a non-dividend-paying stock has no early exercise premium.

---

### Trinomial Tree

**Model:** JarrowRudd binomial engine via QuantLib, equivalent to trinomial tree pricing, with n=200 steps. Uses QuantLib's BlackScholesMertonProcess with flat yield curve and constant volatility surface. Parameters are sourced from SPY via yfinance, consistent with the binomial tree module.

**Strike prices tested:** Five strikes spanning a ±10% moneyness range around the live spot price S0:

| Label    | Strike    | Moneyness |
| -------- | --------- | --------- |
| Deep ITM | S0 x 0.90 | -10%      |
| ITM      | S0 x 0.95 | -5%       |
| ATM      | S0 x 1.00 | 0%        |
| OTM      | S0 x 1.05 | +5%       |
| Deep OTM | S0 x 1.10 | +10%      |

**European Calls:** Call prices decrease monotonically as strike increases. Deep ITM calls carry significant intrinsic value while deep OTM calls are dominated by time value.

**European Puts:** Put prices increase monotonically with strike. Put-call parity is verified across all moneyness levels.

**American Options:** American puts show an early exercise premium relative to European puts, increasing for deep ITM strikes where discounting costs exceed the optionality value of waiting. American calls on non-dividend-paying stocks are equal to European calls.

---

### Dynamic Delta Hedging

**Parameters:** S0 and sigma are drawn from a real 6-month SPY price path fetched via yfinance. N=25 evenly-spaced observations are sampled from the historical window. Strike K is set to the nearest $5 to S0 (ATM proxy).

**Asian Put Pricing:**

Arithmetic Asian options have no closed-form solution. Pricing uses Monte Carlo simulation with a geometric Asian control variate to reduce variance:

```
price_cv = E[payoff_arithmetic] - b* * (E[payoff_geometric] - price_geometric_exact)
```

where `b* = Cov(payoff_arithmetic, payoff_geometric) / Var(payoff_geometric)`

The geometric Asian option has a known closed-form solution with modified volatility and drift:

```
sigma_asian = sigma * sqrt((2N+1) / (6(N+1)))
r_asian = r - sigma^2/2 + sigma_asian^2/2
```

**Hedging Simulation:**

Dynamic delta hedging rebalances the hedge portfolio at each of the N=25 real price observations. At each point, the geometric Asian delta formula provides an approximation to the true arithmetic Asian delta:

```
delta_geometric = exp((r_asian - r)*T) * N(d1)   [for a call]
```

The hedging portfolio consists of a short option position offset by a dynamic stock holding of (-delta) shares. Cash flows from buying/selling shares are accumulated in a risk-free cash account earning r. At maturity, the stock position is closed, the option payoff is settled, and the residual cash balance represents the hedging error.

Two plots are produced: the delta path along the simulated stock price trajectory (showing how the hedge ratio evolves) and the cumulative portfolio value over time (showing hedging error accumulation). Discrete rebalancing introduces path-dependent hedging error; finer rebalancing reduces but does not eliminate this error.

---

## Key Findings

- CRR binomial tree converges to stable option prices by n=50, validating the choice of n=100.
- European and American calls on non-dividend-paying stocks price identically with no early exercise premium.
- The American put carries an early exercise premium that increases for deep ITM strikes.
- Trinomial tree prices confirm monotone call/put price profiles across the moneyness spectrum.
- Asian option delta hedging with geometric approximation produces small but non-zero hedging errors due to discrete rebalancing and the approximation in the delta formula.

---

## Tech Stack

```
Python 3.x
numpy
scipy
matplotlib
yfinance
QuantLib (trinomial tree via JarrowRudd engine)
```

---

## Installation

```bash
git clone https://github.com/QuantSingularity/Binomial-Trinomial-Asian-Option-Pricing.git
cd Binomial-Trinomial-Asian-Option-Pricing
pip install numpy scipy matplotlib yfinance QuantLib
jupyter notebook Binomial_Trinomial_Asian_Option_Pricing.ipynb
```

---

## References

- Cox, J.C., Ross, S.A., & Rubinstein, M. (1979). Option Pricing: A Simplified Approach. _Journal of Financial Economics_, 7(3), 229-263.
- Jarrow, R., & Rudd, A. (1983). _Option Pricing_. Dow Jones-Irwin.
- Hull, J.C. (2018). _Options, Futures, and Other Derivatives_ (10th ed.). Pearson.
- Kemna, A.G.Z., & Vorst, A.C.F. (1990). A Pricing Method for Options Based on Average Asset Values. _Journal of Banking and Finance_, 14(1), 113-129.
- QuantLib Documentation: https://www.quantlib.org
