# Levered FX Carry: Cross-Currency Bond Strategy

**Madhav Fadadu · February 2026**

---

There's a trade that's been quietly working since the 1980s. You borrow in low-rate currencies, lend in high-rate ones, and pocket the spread. Textbook economics says this shouldn't work. If markets are efficient and uncovered interest parity (UIP) holds, the high-yield currency depreciates by exactly the interest differential, leaving you with nothing. Except UIP doesn't hold. It barely even limps. High-yield currencies persistently underperform their implied depreciation path, and the gap between the rate differential and actual FX moves is what carry traders live on.

This project is a structured backtest of that trade across four EM currencies: Turkish Lira, Nigerian Naira, Brazilian Real, and South African Rand. It is framed not as a spot FX bet but as a synthetic cross-currency fixed-rate bond position. The distinction matters more than it sounds.

---

## Position Structure and Leverage Mechanics

The core framing: we enter each week (Wednesdays) as a fixed-rate bond investor in an EM currency, funded by borrowing in GBP at the BoE OIS rate.

**Lending Leg.** Convert \$10MM USD into local currency at spot. Invest in a synthetic 5-year local-rate bond issued at par, with coupon = entry 5Y par swap rate, quarterly payments. At exit (one week later), reprice that bond on a freshly-bootstrapped zero curve from current market yields. Convert the position back to USD at the new spot. The P&L here is the sum of carry accrual, mark-to-market rate moves, and FX translation.

**Funding Leg.** Borrow 80% of notional (\$8MM) in GBP at spot, translate to GBP. Accrue interest at OIS + 50bp over the week. Buy it back in USD at the new GBP/USD rate to close. The P&L here is the cross-currency funding cost including GBP/USD moves.

**Equity per leg: \$2MM. Leverage: 5x.**

**Entry filter:** Only trade if EM 5Y swap > OIS + 50bp. The 50bp minimum carry threshold is the cost-of-carry hurdle; below that, transaction costs alone would eat the edge.

One thing worth being explicit about: this is a constant-notional backtest. Each week is a fresh position at the same size. No compounding, no capital recycling. That's intentional: it makes P&L attribution cleaner and avoids the compounding-path dependency that can distort long-horizon backtest results when you're dealing with high-leverage strategies on volatile EM FX.

---

## Currency Universe and Selection Rationale

The universe isn't random. These are four countries where you can actually source 5Y local swap curve data from Bloomberg and where the carry differential vs. GBP is persistently wide enough to matter.

**TRY (Turkey):** The poster child of carry. Turkish rates have been structurally elevated for years driven by real inflation and periodic heterodox monetary policy. The 5Y swap has traded anywhere from 10% to over 50% during the backtest window. High carry, high vol, high reward.

**NGN (Nigeria):** A less-traded but genuinely interesting leg. Nigerian rates reflect structural dollar scarcity and a managed FX regime that has periodically blown up. Thin data (only 228 swap observations in the raw set vs. 1,763 for BRL), but the carry has been wide enough to show up meaningfully in results.

**BRL (Brazil):** A more institutionally developed carry leg. Brazilian rates track inflation and the Copom rate cycle closely. The 5Y swap has clear regime shifts (COVID collapse, tightening cycle, etc.) that the backtest picks up on through the active filter.

**ZAR (South Africa):** One structural quirk: the South African dataset had no 1Y yield, so the bootstrap uses a flat short-end assumption (s1 = s5). That's a simplification; the alternative would be fabricating a data point. ZAR is a commodity-linked EM currency with meaningful GBP correlation given historical trade ties.

---

## Data Pipeline and Preprocessing

This is where real implementation diverges from textbook framing.

**Swap curves** came from Bloomberg exports: unaligned CSVs where dates and values for 1Y, 5Y, and 10Y tenors sat in separate column pairs with no guarantee of row alignment. The `transform_swap_yield_data` function reconstructs a clean date-indexed panel by extracting valid date-value pairs per tenor independently and outer-joining them. Missing days within each curve are filled by linear time-interpolation before any trading logic sees the data.

**FX rates** are sourced from Nasdaq Data Link's EDI/CUR table (daily spot, CCY per USD). If that's rate-limited, there's a two-source fallback: FRED for GBP, BRL, ZAR, and ECB cross-rates via the EUR/X series for TRY and NGN. This ended up being necessary during development; Nasdaq's free tier is easy to exhaust on a 7-year, 5-currency pull.

**OIS rates** (BoE IUDSOIA, the Sterling Overnight Index Average) load from a local cache first, then Nasdaq BOE/IUDSOIA, then FRED as fallback. There's a scaling detection step built in: if the series max is below 0.15, it's assumed to be in decimal and gets multiplied by 100. The cached file showed a range of 4.51% to 520.01%, which is a tell that some rows had a data quality issue at the high end, worth flagging before using this in production.

---

## Zero-Curve Bootstrap and Bond Repricing

Most FX carry implementations just compute a forward rate and call it a day. Framing it as a bond position forces you to be explicit about things you usually hand-wave.

**Zero-curve bootstrap.** At each entry point, we bootstrap a continuously-compounded zero curve from the 1Y and 5Y par swap rates. The 1Y zero comes directly from `z1 = ln(1 + s1)`. The 5Y zero is solved numerically (Brent's method, 1e-12 tolerance) to ensure the 5Y par swap prices at par given linear zero-curve interpolation between 1Y and 5Y. Annual payments assumed for the bootstrap; quarterly for the bond itself.

**Dirty price repricing.** At exit, we don't just mark the FX. We reprice the bond on the new zero curve constructed from exit-date swap yields, with `time_elapsed = actual calendar days / 365.25`. The dirty price includes the partial coupon accrual. This means rate moves (duration exposure) are properly captured in the lending leg P&L alongside the FX translation.

This distinction between lending-leg P&L and funding-leg P&L matters for decomposition. The lending leg is where you earn carry, take duration risk, and take FX risk. The funding leg is a low-vol drag: you're paying OIS + 50bp on borrowed GBP, and GBP/USD moves hit you there. In the results, the lending leg dominates total P&L across all active currencies, as you'd expect.

---

## Backtest Results and Performance Attribution

**Portfolio total P&L over the backtest period: ~\$53.6MM** across \$8MM equity (4 legs x \$2MM).

| Currency | Total P&L | Active Weeks | Notes                                       |
|----------|-----------|--------------|---------------------------------------------|
| TRY      | \$19.9MM  | 98 / 316     | Rate volatility = wide swings, net positive |
| NGN      | \$19.5MM  | 91 / 324     | Thin data, concentrated carry episodes      |
| BRL      | \$5.4MM   | 61 / 301     | More selective entry, steadier equity curve |
| ZAR      | \$8.7MM   | 89 / 363     | Consistent contributor                      |

A few things stand out on honest inspection:

**The carry filter is mostly non-binding across the four currencies.** TRY's 5Y rate was above OIS + 50bp almost universally; the filter is only actively doing work in the tails of the rate cycle. The strategy's returns are from persistent carry, not market timing.

**Cross-currency correlations are genuinely low,** most pairings below 0.2. That's real diversification in normal markets. The caveat is the right one: EM carry correlations converge toward 1.0 in crisis events (2018 EM tantrum, COVID, risk-off episodes). A volatility-targeted or drawdown-limited overlay would help.

**Sharpe ratios are inflated and should be read as such.** Zero transaction costs, clean weekly rebalancing at last Wednesday price, and interpolated curves all bias Sharpe upward. A realistic estimate would add bid/ask (50-100bp round-trip in most of these markets), some slippage on the FX leg, and realistic curve data availability constraints. Sharpe > 5 is a signal that the simulation is clean, not that the strategy is a perpetual motion machine.

**The "max drawdown > 100%"** mentioned in the notebook is a methodology note, not a real loss. Returns-on-equity for a constant-notional levered strategy are unbounded: the equity base is fixed at entry, not dynamically adjusted. Drawdown on the cumulative dollar P&L path is the right way to read it.

---

## Production Gaps and Execution Constraints

In roughly priority order:

1. **Transaction costs.** BRL and ZAR have tighter swap markets; TRY and NGN have significantly wider bid/ask on the FX side. A realistic friction model would compress returns and change the active-week distribution: the marginal carry weeks near the 50bp threshold probably flip from slightly positive to slightly negative.

2. **Volatility targeting.** Position size should scale inversely with recent realized vol. Right now \$10MM is constant regardless of whether TRY vol is 15% or 60%. A simple inverse-vol sizing would cut drawdown materially.

3. **Funding haircuts.** OIS flat is a theoretical benchmark. Real repo/FX swap funding in these markets includes a basis: the GBP/USD cross-currency basis swap spread can be 20-40bp on its own. That compresses carry before you even add bid/ask.

4. **Liquidity and notional limits.** NGN in particular has thin liquidity. \$10MM notional in the NGN swap market would move the market. A real implementation would either reduce size or accept worse execution.

5. **Stress scenarios.** The carry trade's known failure mode is a sudden EM risk-off event: yields spike, FX gaps down, funding costs spike, all at once. A stress test against 2018 EM tantrum, COVID March 2020, and a hypothetical NGN devaluation event would give a much more honest picture of tail risk.

---

## Dependencies

```
Python 3.11+
pandas · numpy · scipy (brentq) · matplotlib · seaborn
nasdaqdatalink (Nasdaq Data Link Python SDK)
python-dotenv
Jupyter / IPython
```

**Data sources:**

- Bloomberg Terminal (swap yield curve exports, CSV)
- Nasdaq Data Link EDI/CUR (FX spot rates)
- Bank of England IUDSOIA via Nasdaq / FRED (OIS benchmark)
- FRED + ECB Statistical Data Warehouse (fallback FX)

---

## Repository Structure

```
.
├── HW6_FX_Carry_Strategy.ipynb  # Main notebook, end-to-end strategy
├── .env                          # API keys (not committed to git)
└── data/
    ├── EmergingMkt_YC_Download_BBERG_Sheets_1.csv   # TRY swap curve
    ├── EmergingMkt_YC_Download_BBERG_Sheets_2.csv   # NGN swap curve
    ├── EmergingMkt_YC_Download_BBERG_Sheets_4.csv   # ZAR swap curve
    ├── EmergingMkt_YC_Download_BBERG_Sheets_5.csv   # BRL swap curve
    └── IUDSOIA_cached.csv                            # BoE OIS (cached)
```

---

The carry filter does most of its work quietly. Across four currencies and several years of data, the filter is mostly non-binding: TRY, NGN, BRL, and ZAR cleared the 50bp threshold the majority of eligible weeks. The returns are from persistent structural carry, not timing. What the filter does catch are the episodic dislocations: rate spikes, managed FX regimes, and crisis periods where the quoted yield and the tradeable yield diverge. That's exactly what it's supposed to do.
