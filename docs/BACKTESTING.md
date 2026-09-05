# BACKTESTING PLAN

Deliverable **(13)**. Status: PROPOSED.

---

## 1. Principle

The backtester runs **the same engine code** as live. There is no separate
"backtest logic". The only difference is the clock and the fill model:

```
live      : CausalView fed by a streaming bar source, wall clock
backtest  : CausalView fed by a stored tape, simulated clock
shadow    : live data, live clock, simulated fills
```

If those three ever diverge in behaviour, the parity/no-lookahead tests fail.

---

## 2. Data

| Item | Requirement |
|---|---|
| Instrument | NQ (and MNQ derived — same price series, different multiplier) |
| Timeframe | 1-minute OHLCV minimum; tick data if available for fill realism |
| Window | 07:00–12:00 ET is the decision window, but the loader needs 18:00 ET (D−1) onward for overnight levels and the prior CME day for PDH/PDL |
| History | ≥ 3 years for regime coverage (this spans very different volatility regimes — results must be reported per regime, never pooled into one headline number) |
| Quality checks | gap detection, duplicate bars, zero-volume bars, session boundary validation, roll-date verification, comparison of two vendors where possible |
| Provenance | vendor, download date, checksum recorded in `backtest_runs.data_source` |

**Blocking question B5/B6:** vendor and roll policy must be chosen before any
historical claim is meaningful.

---

## 3. Cost and fill model (never optimistic)

```
commission_rt      per instrument, real broker rate
slippage_entry     default 1.0 pt   (limit at zone; adverse)
slippage_stop      default 1.0 pt   (stops slip; often more on news)
slippage_target    default 0.0 pt   (limit fills at the level, no improvement)
entry_touch_ticks  1 tick through the zone required to assume a fill
allow_same_bar_fill = false         signal bar cannot also be the fill bar
```

Additional pessimistic conventions:

* If a bar's range contains **both** the stop and the target, assume the
  **stop** filled first, unless tick data proves otherwise.
* Entry limit orders are assumed filled only if price trades *through* the level
  by `entry_touch_ticks`, not merely to it.
* News-window trades use `slippage_news_mult` (default 2.0×) on stops.
* No partial-fill optimism: either the whole size fills or none.

These conventions cost real R in the results, which is the point. A backtest
that assumes perfect fills on a 08:30 NFP reaction is fiction.

---

## 4. Metrics

Per run, per dimension:

```
Total setups · Total trades · Signal-to-trade conversion
Win rate · Profit factor · Expectancy (R and $)
Average R · Median R · Standard deviation of R
Max drawdown (R and $) · Longest losing streak · Longest winning streak
Average setup quality · Average confluence
Average ORB size · Time-to-entry · Time-to-TP1 · Time-to-TP2
Invalidation rate · Retest failure rate
MAE / MFE distributions
Days traded / days stood down (and why)
```

Reported with **sample size and confidence intervals** on every number. A win
rate from 14 trades is not a win rate; the report labels any bucket with n < 30
as `INSUFFICIENT SAMPLE` and refuses to rank it.

---

## 5. Analysis dimensions

Every metric is sliced by:

| Dimension | Buckets |
|---|---|
| ORB size | 10–20, 20–30, …, 90–100 (plus the stand-down buckets for context) |
| Weekday | Mon–Fri |
| Entry time | 08:30–09:00, 09:00–09:30, 09:30–10:00, 10:00–10:30, 10:30–11:00 |
| Setup type | Continuation, News Reversal |
| Confluence | 3,4,5,6,7,8,9,10+ |
| Quality band | Low, Developing, High, A+ |
| Bias | Bullish, Bearish, Neutral |
| Sweep source | ORB, PDH, PDL, ONH, ONL, EQH, EQL |
| News day | catalyst present / absent |
| Volatility regime | ATR quintile |

Stored long-form in `backtest_metrics` so new dimensions need no migration.

**Multiple-comparison warning, stated in the report itself:** slicing 10
dimensions × ~8 buckets over one strategy will produce impressive-looking
subgroups by chance alone. Subgroup findings are hypotheses for out-of-sample
testing, not conclusions.

---

## 6. Validation protocol

1. **In-sample development** on the oldest 60% of history.
2. **Walk-forward**: rolling 6-month train / 3-month test windows; parameters
   fixed within each test window; results reported per fold, never pooled.
3. **Hold-out**: the most recent 20% is untouched until the model is frozen.
   It is looked at once. If it fails, the honest response is "the model does not
   generalise", not "re-tune and look again".
4. **Sensitivity analysis**: for each key parameter, sweep ±30% and plot the
   metric surface. **A parameter whose edge exists only at one value is
   overfitted** — prefer plateaus, not peaks.
5. **Ablation**: remove each confluence factor in turn; a factor whose removal
   does not degrade results is not carrying information and should be dropped or
   re-weighted.
6. **Correlation audit**: the 12-factor correlation matrix. Any pair with
   |ρ| > 0.8 is double-counting (sweep vs rejection block is the obvious
   candidate, `A-08`).
7. **Null benchmark**: compare against a deliberately naive alternative — e.g.
   "enter at 08:30 in the direction of the ORB break with the same risk rules".
   If the full model does not beat that, the complexity is not earning its keep.

---

## 7. Reproducibility

A backtest result is only valid when accompanied by:

```
run_id · config_hash · param_version · code git SHA
data source + checksum · date range · fill model + parameters
sessions/setups/trades counts · fold structure
```

`northstar backtest --rerun <run_id>` must reproduce identical numbers. If it
cannot, the result is discarded rather than explained away.

---

## 8. Reporting

Per run: a generated HTML/PDF report containing equity curve in R, drawdown
curve, R distribution histogram, per-dimension tables, the rejection-reason
ledger (why no-trade days were no-trade days), the correlation matrix, the
sensitivity surfaces, and an explicit "limitations" section listing every
`PROPOSED` parameter that was active.

The report never states a probability derived from the confluence count unless
that number came from measured outcomes with an adequate sample.
