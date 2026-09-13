# Wyckoff + VSA + Pivot Futures Strategy (Pine Script v6)

TradingView strategy for Binance Futures USDT-M Perpetual backtests.

**File:** [`Wyckoff_VSA_Pivot_Futures_Strategy.pine`](Wyckoff_VSA_Pivot_Futures_Strategy.pine)

## How to use

1. Open TradingView → Pine Editor → paste the `.pine` file → Add to chart.
2. Chart timeframe: **5 minutes** (primary).
3. Input **Higher Timeframe** = `60` (1 hour).
   - If the chart timeframe is already at or above this, `request.security` would return the
     chart's own series and every HTF gate would pass trivially. **Min HTF multiple of chart TF**
     (default `4`) escalates the request instead. The dashboard's `HTF Trend (…)` row shows the
     effective timeframe in minutes, so check it there rather than assuming the input is in force.
     On a 1H chart this is the difference between profit factor 0.27 and 0.81.
4. Symbol: e.g. `BTCUSDT.P`, `ETHUSDT.P` on Binance Futures.
5. Strategy Tester: commission **0.04%** and slippage **2 ticks** (script defaults).
6. Keep **Confirmed bar signals** ON (default) — no repaint / no lookahead.

---

## What actually makes an entry good (measured, not asserted)

Measured on `BINANCE:BTCUSDT.P`, 1H, Jan 1 – Sep 13 2026. The **Setup Selection** input group
enables one setup class at a time, which is how these numbers were attributed.

| Run | Setups enabled | Trades | Net | Win rate | Profit factor | Max DD |
|---|---|---|---|---|---|---|
| A | all | 26 | −31.04 (−0.31%) | 42.31% | 0.942 | 2.33% |
| B | SPRING_TEST + SPRING + UPTHRUST only | **0** | — | — | — | — |
| C | the other five | 26 | −31.04 (−0.31%) | 42.31% | 0.942 | 2.33% |

Two conclusions follow, and they matter more than the headline number.

**1. Stop placement dominates entry selection.** The single change that moved results was fixing
`calculateStop`, which took `math.min` of three support levels for a long — the *furthest* one, and
therefore the widest possible stop. That inflates stop %, shrinks position size and degrades R:R,
so the `maxStopPercent` and `minRR` filters rejected setups that were otherwise valid. With the
stop at the *nearest* structure beyond entry, trades went 13 → 26 and profit factor 0.809 → 0.942.
The census of `NO BUY:` debug labels on the chart made this visible: "Stop too wide" and
"RR < minimum" were the most frequent rejections by a wide margin.

So a good entry is one where **the invalidation level is close to entry**. That is the real
content of the Wyckoff spring idea: price pierces support, recovers, and the stop goes just under
the spring low — tight, well-defined risk and therefore high R:R. Setups entered far from
structure are penalised automatically through position sizing, with no extra filter needed.

**2. The `SP` / `UT` markers are not all tradeable setups.** `detectSpring()` requires only a
pierce-and-recover of the N-bar low with a lower wick — it has **no range-context requirement**.
A lower wick in a downtrend therefore plotted the same `SP` marker as a genuine accumulation
spring. Judging setups by eye from those markers is a selection effect. The markers are now split:

- **`SP` / `UT`** (bright) — in accumulation / distribution context. Tradeable.
- **`sp` / `ut`** (dim grey) — no context. Not tradeable, and not counted as a setup.

Run B returning zero trades is consistent with this: over this window BTC trended on 1H, so
almost no spring occurred inside an established trading range. Three separate bugs had also been
blocking these setups outright, all now fixed — a `rVol < 1.0` versus `rVol >= 1.0` contradiction
that made `SPRING_TEST` unreachable, a duplicated trend score floor that overrode the
range-reversal floor, and trend-premise filters (ADX, BB width, sideway block) applied to setups
that occur by definition in ranges.

To test the spring premise properly you need a **ranging** period or symbol. The one knob that
widens recognition is **Trading-range max span** (`rangeSpanFrac`, default `0.475`); raising it
admits wider ranges as "trading ranges". It is deliberately defaulted to reproduce the measured
configuration, so change it as a measured experiment rather than an assumption.

---

## A. BUY (LONG) flow

1. HTF trend score bullish (closed 1H bar via `request.security` + `barmerge.lookahead_off`).
2. Regime not `STRONG_BEAR`; prefer `BULL` / `STRONG_BULL` (controlled `SIDEWAY` allowed with lower score).
3. Confluence score: structure + Wyckoff accumulation proxy + VSA/SOS + pivot breakout / retest / pullback / spring-test.
4. Hard filters: grade S+/S/A (B optional), min score, RR, max stop %, size > 0, cooldown, anti-whipsaw.
5. Signal on **bar close** → entry via `process_orders_on_close` (no future price).

## B. SELL (SHORT) flow

Symmetric: HTF bearish, distribution / upthrust / SOW, pivot breakdown or retest or failed breakout, sell score + hard filters.

## C. Wyckoff → quantitative proxy

Not full discretionary Wyckoff. Scores a **proxy**:

- Trading range width vs ATR, volatility contraction, volume dry-up
- Spring / test / SOS (long); Buying climax / upthrust / No Demand / SOW (short)
- `wyckoffAccumScore` / `wyckoffDistScore` vs input thresholds

## D. VSA

Per bar: spread, body, wicks, close position, relative volume.

Classifies strong bull/bear, No Supply/Demand, SOS/SOW, climax vs absorption. Score is **location-weighted** (breakout/support vs mid-range).

## E. Pivot

```text
pivotHigh = ta.highest(high[1], lookback)
pivotLow  = ta.lowest(low[1], lookback)
```

Current bar excluded → no pivot lookahead.

## F. Breakout confirmation

Close beyond pivot + body/close-position/spread rules + `volume >= avg × multiplier` → strong breakout. Without volume → `WEAK_BREAKOUT` (no immediate entry).

## G. Retest

Within `retestWindow` bars after breakout: touch pivot ± ATR tolerance, close back on side of breakout, pullback volume < breakout volume, then VSA/candle confirmation.

## H. Stop loss

Modes: STRUCTURE / ATR / PERCENT / STRUCTURE + ATR BUFFER. Long under swing/spring/pivot; short over swing/upthrust/pivot. Reject if stop % > max.

## I. Position size

```text
riskAmount   = equity × riskPercent / 100
stopDistance = |entry − stop|
qty          = riskAmount / stopDistance
```

Clamp notional to `maxPositionPercent`. **Leverage input is informational only** — it does not inflate risk size.

## J. TP / trailing

- TP1 = +`tp1R` R (partial %), TP2 = +`tp2R` R (partial %)
- Move SL to break-even (+ fee buffer) at +2R
- Trail remainder after TP1/+2R: Swing / EMA21 / ATR / Chandelier
- Early exit on structure failure (SOW/SOS, failed breakout, EMA flip, HTF regime flip)

## K. Wyckoff proxies (limits)

Pine cannot “understand” full phases (SC→AR→ST→Spring→SOS) with human context. Springs, upthrusts, ranges, and volume dry-up are **rule-based approximations**. Always discretionary-review live.

## L. Parameters to optimize (Strategy Tester)

Prioritize: Pivot Lookback, Volume Multiplier, Minimum BUY/SELL Score, ATR Stop Mult, Max Stop %, Risk %, Retest Window, HTF bull/bear score thresholds, sideway ATR%/BB width, Cooldown bars.  
Change **one group at a time**; walk-forward across symbols/periods.

## M. Avoid overfitting

- Scoring confluence, not 10 mandatory indicators
- RSI/MACD/CMF/MFI/OBV are **soft bonuses** only
- Same defaults across BTC/ETH/SOL/BNB before symbol-specific tweaks
- Prefer fewer high-quality trades over max trade count
- Always include commission + slippage

---

## Modular functions (in script)

`getTrendScore`, `getMarketRegime`, `getVsaScore`, `detectSOS` / `detectSOW`, `detectSpring` / `detectTest` / `detectUpthrust`, `detectWyckoffAccumulation` / `detectWyckoffDistribution`, `detectBreakout` / `detectBreakdown`, `detectRetestLong` / `detectRetestShort`, `detectPullbackLong` / `detectPullbackShort`, `calculateBuyScore` / `calculateSellScore`, `calculateStop`, `calculatePositionSize`, `calculateRR`, `manageLong` / `manageShort`

## Checklist — BTC / ETH / SOL / BNB USDT-M Perp (5m / HTF 1H)

- [ ] Paste script, chart TF = 5m, HTF input = 60
- [ ] Commission 0.04% (or your fee tier), slippage ≥ 1–2 ticks
- [ ] Date range: ≥ 6–12 months including chop and trend
- [ ] Debug Mode ON → sample “NO BUY” reasons make sense
- [ ] No signal changes on historical closed bars when refreshing (anti-repaint smoke test)
- [ ] Flat in clear SIDEWAY / tiny BB width / low ADX
- [ ] Longs rare or blocked when HTF = STRONG_BEAR
- [ ] Stop never on wrong side; TP beyond entry in trade direction
- [ ] Position size scales with stop distance; notional ≤ max %
- [ ] Compare metrics across BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT — not only one coin
- [ ] Inspect entry comments: `PIVOT_BREAKOUT`, `RETEST`, `PULLBACK`, `POCKET_PIVOT`, `SPRING`, `UPTHRUST`, etc.
- [ ] After SL, no instant re-entry (cooldown)

## Priority order (code)

1. Risk management → 2. HTF trend → 3. Structure → 4. Wyckoff → 5. VSA → 6. Pivot → 7. Volume → 8. Retest/pullback → 9. Momentum bonus → 10. Execution
