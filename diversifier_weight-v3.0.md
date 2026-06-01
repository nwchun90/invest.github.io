# VAA + Dynamic Internal Rebalancing — Methodology Record

**Author:** nwchun90 | **Series:** Systematic Investing, Value Investing | **Version:** 3 — May 2026

|Version|Date|Changes|
|-|-|-|
|1|May 2026|Initial release|
|2|May 2026|Rolling window shortened to 50 weeks; momentum gate added to rebalancing trigger (§4); §6.2 updated to reflect adopted mitigations|
|3|May 2026|Initialisation replaced: pre-deployment historical data used to seed w\_L / w\_H instead of 50/50 cold start; fallback rule added for short-history assets (§3); practical cap tightened from \[0.05, 0.70] to \[0.05, 0.50] (§3)|

\---

## 1\. THE STRATEGY IN ONE PARAGRAPH

Value-Averaged Accumulation (VAA) with Internal Rebalancing is a disciplined, buy-only systematic investing framework. A target portfolio value grows linearly each period. Cash is deployed **only when the portfolio falls short of the target** — never sold to cash, never capped. Two assets are held simultaneously. When one drifts away from its target allocation, the portfolio rebalances **internally** (selling the overweight asset to buy the underweight one), with no external cash exchanged. The combination of VAA's crash-buying mechanic and the internal rebalancing premium creates two distinct sources of mechanical return: **dip deployment** and **volatility harvesting**.

\---

## 2\. VAA RULE (BUY-ONLY)

```
target\\\\\\\_t  = V\\\\\\\_0 + step × t
shortfall = max(0, target\\\\\\\_t − portfolio\\\\\\\_value\\\\\\\_t)
invest    = shortfall   if portfolio < target
invest    = 0           if portfolio ≥ target   (sit on hands)
```

* V\_0 = initial investment (e.g. $10,000)
* step = fixed monthly / quarterly target growth (e.g. $500)
* t = period number
* **No selling for cash. No contribution cap.**
* When investing, split the shortfall between the two assets at the current target weights.

\---

## 3\. DIVERSIFIER WEIGHT FORMULA

```
w\\\\\\\_L = (E\\\\\\\_R\\\\\\\_L / E\\\\\\\_R\\\\\\\_H) × (σ\\\\\\\_H / (σ\\\\\\\_H + σ\\\\\\\_L)) × (1 − ρ)

w\\\\\\\_H = 1 − w\\\\\\\_L
```

|Symbol|Meaning|Computed from|
|-|-|-|
|w\_L|Target weight of the lower-return asset|Formula output|
|E\_R\_L|Expected return of the lower-return asset|Rolling 50-week mean weekly return|
|E\_R\_H|Expected return of the higher-return asset|Rolling 50-week mean weekly return|
|σ\_L|Volatility of the lower-return asset|Rolling 50-week sample std dev of weekly returns|
|σ\_H|Volatility of the higher-return asset|Rolling 50-week sample std dev of weekly returns|
|ρ|Correlation between the two assets|Rolling 50-week Pearson correlation|

**H / L assignment:** each period, the asset with the higher rolling mean return is H;
the other is L. **This assignment can flip over time.**

**Practical cap:** clip w\_L to **\[0.05, 0.50]** to prevent degenerate allocations. The
upper bound of 0.50 enforces a structural constraint: the lower-return asset is never
permitted to carry more weight than the higher-return asset (w\_L > 0.50 would imply
w\_H < 0.50, i.e. H is underweighted relative to L, which contradicts the return-based
ranking). The lower bound of 0.05 prevents the diversifier from being effectively
eliminated.

**Initialisation (v3):** Rather than using a 50/50 cold start, compute w\_L and w\_H
from historical price data available *before* the portfolio deployment date. The full
procedure is:

1. **Standard case — sufficient history for both assets:** if 50 or more weeks of
weekly closing prices exist for both assets prior to the start date, apply the
full 50-week rolling window to compute E\_R, σ, and ρ exactly as in the live
loop. The portfolio launches with analytically derived weights on day one.
2. **Short-history fallback — one asset has fewer than 50 weeks of pre-start data**
(e.g. a recently listed security): let *n* be the number of weeks available for
the shorter-history asset. Use *n* weeks of data for **both** assets — aligned to
the same trailing window — to compute E\_R, σ, and ρ. Do not mix window lengths
between the two assets. Apply the practical cap \[0.05, 0.50] as normal. Once the
portfolio is live, the window expands by one week per period until it reaches the
full 50 weeks, after which it rolls as a standard fixed-length window.
3. **Extreme edge case — fewer than 4 weeks of pre-start data for either asset:**
insufficient data to estimate correlation reliably. Use 50/50 weights for the
first period only, then switch to the expanding window as soon as a 4th weekly
return observation becomes available.

The 50/50 cold start is therefore only invoked as a genuine last resort, not as
the default. Pre-deployment calibration ensures the first cash deployment is directed
by signal rather than by an arbitrary equal split.

### Formula intuition (term-by-term)

|Term|Effect|
|-|-|
|E\_R\_L / E\_R\_H|Penalises the lower-return asset — gives it a smaller weight|
|σ\_H / (σ\_H + σ\_L)|Rewards higher main-asset volatility — more vol = more rebalancing premium|
|(1 − ρ)|Rewards low correlation — uncorrelated assets diverge more, creating more harvest opportunities|

**Note on asymmetry:** applying the same formula to each asset independently gives
w\_A + w\_B ≠ 1.0. This is by design — the formula is not a budget constraint. Always
compute w\_L from the formula and set w\_H = 1 − w\_L.

\---

## 4\. REBALANCING TRIGGER (10% BAND RULE + MOMENTUM GATE)

```
R\\\\\\\_target = w\\\\\\\_L / w\\\\\\\_H          ← target ratio of L-asset to H-asset value
R\\\\\\\_actual = val\\\\\\\_L / val\\\\\\\_H      ← actual ratio in current portfolio

Rebalance if: |R\\\\\\\_actual − R\\\\\\\_target| / R\\\\\\\_target > 0.10
```

**Momentum gate (v2 addition):** before executing a rebalance, apply the
following filter:

```
IF  R\\\\\\\_actual > R\\\\\\\_target          ← H is the overweight asset
AND price\\\\\\\_H > MA\\\\\\\_12(H)           ← H is above its 12-month simple moving average
THEN  skip rebalance this period  ← let the winner run
ELSE  rebalance normally
```

Where `MA\\\\\\\_12(H)` is the 12-month (≈ 52-week) simple moving average of H's price,
computed from weekly closing prices.

**All other cases rebalance normally:**

* L is overweight (R\_actual < R\_target): always rebalance — buying more of L when it is cheap aligns with both mean-reversion and the VAA crash-buying discipline.
* H is overweight but H's price is below MA\_12(H): rebalance — H is in a downtrend and trimming it is appropriate.

**Rationale:** the 10% band rule alone will sell the outperforming asset during a sustained bull run, mechanically capping gains. The momentum gate identifies the specific scenario where that trim is most costly — when H is both overweight *and* in a confirmed uptrend — and suspends the rebalance for that period only. This allows the portfolio to retain trend-driven gains without permanently abandoning the rebalancing discipline. The gate reactivates automatically once H's price crosses back below its 12-month moving average, restoring the full band rule.

Cash flow = $0 for all rebalances. Recheck every period after the VAA contribution is applied.

\---

## 5\. FULL PERIOD LOOP (ONE MONTH)

```
① Update rolling 50-week stats → compute w\\\\\\\_L, w\\\\\\\_H
② Check 10% band → rebalance internally if triggered
③ Compute portfolio value after rebalancing
④ Compute shortfall vs target\\\\\\\_t
⑤ If shortfall > 0: deploy cash split w\\\\\\\_H / w\\\\\\\_L
```

\---

## 6\. CONCLUSIONS ON BEST DEPLOYMENT

Based on simulation work (ZGD.TO + VDY.TO, Nov 2012 – Apr 2026):

### 6.1 What works well

* **VAA's buy discipline** is the primary return driver. Cash deployed during crashes
(rare: 27 of 162 months for ZGD) compounds into the subsequent recovery.
* **Internal rebalancing** adds a genuine Fernholz bonus (\~1–3% per year theoretical)
when assets alternate leadership over short cycles.
* **Best suited for:** two assets with near-zero or negative correlation, both with
positive long-run expected returns, and volatilities that are meaningfully different.

### 6.2 The momentum lag penalty (key risk) — and practical fix

The rolling window is **reactive**, not predictive. After one asset crashes, the
formula reduces its weight — exactly when that asset is cheapest. This
**whipsaw effect** offsets or eliminates the rebalancing bonus in trending markets
where one asset has a structural multi-year advantage. The shorter the window,
the faster it reacts, but also the noisier the weight signal becomes in choppy
sideways markets.

**Adopted improvements (v2):**

1. **Shortened window to 50 weeks** (from 24 months). The tighter window captures
regime changes roughly twice as fast, reducing the lag between a trend reversal
and the formula re-weighting toward the recovering asset. Trade-off: weights
will be more volatile period-to-period in low-signal environments.
2. **Momentum gate on the rebalancing trigger** (see §4). When H is overweight
*and* above its 12-month moving average, the rebalance is skipped. This directly
addresses the most damaging scenario — mechanically selling a trending winner —
without altering the formula or the VAA contribution logic.

**Adopted improvement (v3):**
3. **Pre-deployment initialisation** (see §3). Computing weights from historical
data before the portfolio launch date eliminates the arbitrary 50/50 cold start.
The first cash deployment is now signal-driven. The tightened practical cap
\[0.05, 0.50] additionally enforces a logical floor on the H/L weight relationship,
preventing the formula from ever assigning the lower-return asset a majority
position.

**Other mitigations — documented for reference, not adopted:**

* *Contrarian weighting variant:* invert the formula so w\_H = formula output
(bet on the laggard recovering) — more aligned with mean-reversion theory but
structurally bets against momentum, which is a known persistent factor.
* *Dual moving-average momentum filter:* only rebalance when both assets are
above their own 12-month moving average — avoids buying into structural
declines on either side but may suppress rebalancing activity excessively in
bear markets.

### 6.3 Ideal asset pair profile

|Criterion|Target|
|-|-|
|Correlation ρ|< 0.20 (ideally near 0 or negative)|
|Both assets|Positive long-run expected return|
|Volatility of H|High (> 30% ann.) — amplifies harvesting|
|Volatility ratio σ\_H / σ\_L|> 1.5× for meaningful weight differentiation|
|H/L identity|Should be stable for 12+ months (not flip-flopping monthly)|
|Performance cycles|Assets should alternate leadership, not one dominating throughout|

### 6.4 Recommended use case

VAA + dynamic rebalancing is best deployed as a **long-horizon accumulation strategy**
(10+ years) for an investor who:

1. Has regular savings to invest (the step amount)
2. Can tolerate periods where the portfolio is below the target path
3. Will not override the mechanical rule during market stress
4. Pairs a high-conviction growth asset (H) with a structurally uncorrelated
crisis hedge or dividend income asset (L)

**The strategy is NOT recommended for:**

* Short horizons (< 5 years) — insufficient time to compound the rebalancing premium
* Pairs with correlation > 0.40 — insufficient divergence to harvest
* Situations where one asset may go to zero (the VAA keep-buying rule is dangerous
for structurally failing companies)

\---

*End of methodology record.*

