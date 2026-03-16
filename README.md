# Binomial-Trinomial-Asian-Option-Pricing

> **Derivatives Pricing**
> A computational implementation of binomial and trinomial tree models for pricing European and American options, computing Greeks, and simulating dynamic delta hedging of Asian options.

---

## Overview

This project implements tree-based methods for derivatives pricing, covering the full lifecycle of option valuation: from discrete-time binomial trees for European and American options through QuantLib-powered trinomial trees across the moneyness spectrum, to dynamic delta hedging of path-dependent Asian options using Monte Carlo simulation with a geometric control variate.

| Step                   | Method                                                     | Options                        | Questions               |
| ---------------------- | ---------------------------------------------------------- | ------------------------------ | ----------------------- |
| Step 1 - Binomial      | Cox-Ross-Rubinstein (n=100)                                | European and American call/put | Q5, Q6, Q7, Q8, Q9, Q10 |
| Step 2 - Trinomial     | JarrowRudd via QuantLib (n=200)                            | European and American call/put | Q15, Q16, Q17, Q18      |
| Step 3 - Delta Hedging | Geometric Asian (closed-form) + Arithmetic Asian (MC + CV) | Asian put                      | Q25, Q26, Q27           |

---

## Methodology

### Step 1 - Binomial Tree (European and American)

**Model:** Cox-Ross-Rubinstein (CRR) binomial tree with n=100 steps.

**Parameters:** S0=100, K=100 (ATM), r=5%, sigma=20%, T=3 months.

**Tree construction:**

Up and down factors: u = exp(sigma \* sqrt(dt)), d = 1/u

Risk-neutral probability: p = (exp(r \* dt) - d) / (u - d)

Backward induction for European options:

V(i,j) = exp(-r*dt) * [p * V(i+1,j+1) + (1-p) * V(i+1,j)]

For American options, at each node the continuation value is compared against immediate exercise:

V(i,j) = max(intrinsic_value, continuation_value)

**Q5 - European Call/Put Pricing:** Prices computed at n=100 with convergence plot across step counts 5 to 200, confirming stability.

**Q6 - Delta:** First-order finite difference at the root node:

Delta = (V_up - V_down) / (S_up - S_down)

Call delta is positive (0 to 1); put delta is negative (-1 to 0). Delta represents the instantaneous sensitivity of option value to a unit change in the underlying price and is used directly as the hedge ratio.

**Q7 - Volatility Sensitivity (Vega):** Prices recomputed at sigma=25% to measure the impact of a 200 basis point volatility increase. Both calls and puts increase in value with higher volatility - vega is always positive for long option positions.

**Q8-Q10:** Steps Q5-Q7 repeated for American-style options. Key finding: American put carries early exercise premium over European put due to the value of exercising before expiry when deeply in-the-money. American call on a non-dividend-paying stock has no early exercise premium.

---

### Step 2 - Trinomial Tree (QuantLib)

**Model:** JarrowRudd binomial engine via QuantLib, equivalent to trinomial tree pricing, with n=200 steps. QuantLib's BlackScholesMertonProcess with flat yield curve and constant volatility surface.

**Strike prices tested:**

| Label    | Strike | Moneyness (K/S0) |
| -------- | ------ | ---------------- |
| Deep ITM | 90     | 0.90             |
| ITM      | 95     | 0.95             |
| ATM      | 100    | 1.00             |
| OTM      | 105    | 1.05             |
| Deep OTM | 110    | 1.10             |

**Q15 - European Calls:** Call prices decrease monotonically as strike increases - deep ITM calls carry significant intrinsic value while deep OTM calls are dominated by time value.

**Q16 - European Puts:** Mirror image - put prices increase monotonically with strike. Put-call parity is verified across all moneyness levels.

**Q17/Q18 - American Options:** American puts show early exercise premium relative to European puts, increasing for deep ITM strikes where discounting costs exceed the optionality value of waiting. American calls on non-dividend-paying stocks are equal to European calls.

---

### Step 3 - Dynamic Delta Hedging (Questions 25, 26, 27)

**Parameters:** S0=180, K=182, r=2%, sigma=25%, T=6 months, N=25 averaging steps.

**Q25 - Asian Put Pricing:**

Arithmetic Asian options have no closed-form solution. Pricing uses Monte Carlo simulation with a geometric Asian control variate to reduce variance:

price_cv = E[payoff_arithmetic] - b\* \* (E[payoff_geometric] - price_geometric_exact)

where b\* = Cov(payoff_arithmetic, payoff_geometric) / Var(payoff_geometric)

The geometric Asian option has a known closed-form solution with modified volatility and drift:

sigma_asian = sigma \* sqrt((2N+1) / (6(N+1)))

r_asian = r - sigma^2/2 + sigma_asian^2/2

**Q26 - Delta Hedging Simulation:**

Dynamic delta hedging rebalances the hedge portfolio at each of the N=25 time steps. At each step, the geometric Asian delta formula provides an approximation to the true arithmetic Asian delta:

delta*geometric = exp((r_asian - r)\_T) * N(d1) [for a call]

The hedging portfolio consists of a short option position offset by a dynamic stock holding of (-delta) shares. Cash flows from buying/selling shares are accumulated in a risk-free cash account earning r.

At maturity: the stock position is closed, the option payoff is settled, and the residual cash balance is the hedging error.

**Q27 - Hedging P&L Path:**

Two plots are produced: the delta path along the simulated stock price trajectory (showing how the hedge ratio evolves) and the cumulative portfolio value over time (showing hedging error accumulation). Discrete rebalancing introduces path-dependent hedging error; finer rebalancing reduces but does not eliminate this error.

---

## Key Findings

- CRR binomial tree converges to stable option prices by n=50, validating the choice of n=100
- European and American calls on non-dividend-paying stocks price identically - no early exercise premium
- American put carries early exercise premium that increases for deep ITM strikes
- Trinomial tree prices confirm monotone call/put price profiles across moneyness
- Asian option delta hedging with geometric approximation produces small but non-zero hedging errors due to discrete rebalancing and the approximation in the delta formula

---

## Tech Stack

```
Python 3.x
numpy
scipy
matplotlib
QuantLib (trinomial tree via JarrowRudd engine)
```

---

## Installation

```bash
git clone https://github.com/QuantSingularity/MScFE620-Binomial-Trinomial-Asian-Option-Pricing.git
cd MScFE620-Binomial-Trinomial-Asian-Option-Pricing
pip install numpy scipy matplotlib QuantLib
jupyter notebook Binomial-Trinomial-Asian-Option-Pricing.ipynb
```

---

## Topics

`derivatives-pricing` `binomial-tree` `trinomial-tree` `options` `european-options` `american-options` `asian-options` `delta-hedging` `greeks` `quantlib` `monte-carlo` `control-variate` `black-scholes` `quantitative-finance` `python` `jupyter-notebook` `QuantSingularity`

---

## Organization

| Field        | Detail              |
| ------------ | ------------------- |
| Organization | QuantSingularity    |
| Domain       | Derivatives Pricing |

---

## References

- Cox, J.C., Ross, S.A., & Rubinstein, M. (1979). Option Pricing: A Simplified Approach. _Journal of Financial Economics_, 7(3), 229-263.
- Jarrow, R., & Rudd, A. (1983). _Option Pricing_. Dow Jones-Irwin.
- Hull, J.C. (2018). _Options, Futures, and Other Derivatives_ (10th ed.). Pearson.
- Kemna, A.G.Z., & Vorst, A.C.F. (1990). A Pricing Method for Options Based on Average Asset Values. _Journal of Banking and Finance_, 14(1), 113-129.
- QuantLib Documentation: https://www.quantlib.org
