# NORTHSTAR MOTION MODEL™ — TradingView indicator

`northstar_motion_model.pine` · Pine Script v6 · overlay indicator.

**Display only. This script cannot place an order and is not connected to a
broker.** It reports where the market is inside the Northstar progression and
what condition it is waiting for.

---

## Install

1. TradingView → **Pine Editor** → **Open** → **New indicator**.
2. Delete the template, paste the whole contents of
   `northstar_motion_model.pine`.
3. **Save**, then **Add to chart**.
4. Put the chart on **NQ1! (or MNQ1!) and the 1-minute timeframe**. The
   dashboard shows a red warning if it is on any other timeframe, because the
   ORB, sweep and displacement definitions assume 1m bars.

Chart timezone does not matter — the script converts internally. All session
inputs are interpreted in the **Timezone** input, which defaults to
`America/Los_Angeles`.

---

## Session defaults (Pacific, per `docs/DECISIONS.md`)

| Window | Pacific | Eastern |
|---|---|---|
| Trading day (previous-day high/low/open/close) | 00:00 – 23:59 | 03:00 – 02:59 |
| Overnight high / low | 15:00 – 05:00 | 18:00 – 08:00 |
| Opening range (locks at the end) | **05:00 – 05:15** | 08:00 – 08:15 |
| Execution window | 05:30 – 08:00 | 08:30 – 11:00 |
| Flat | 08:30 | 11:30 |

Pacific and Eastern shift together across DST, so these stay aligned all year.
Change any of them in the **01 · SESSION** input group.

---

## What is on the chart

| Element | Meaning |
|---|---|
| Gold lines + box | ORB high / low / equilibrium, locked at 05:15 PT and never modified |
| Grey lines | PDH, PDL, overnight high / low |
| Faint lines | Day open, midnight ET open, 08:30 ET open, 09:30 ET open |
| Shaded boxes | Fair value gaps (fading as they fill), order blocks, rejection blocks |
| Triangles | Liquidity sweep — a penetration that was *reclaimed*, not a break |
| Diamonds | Qualified displacement |
| Circles | Retest into the entry zone |
| Labels | Entry ready, with the full plan drawn as entry / stop / TP1 / TP2 lines |

## The dashboard

Reads top to bottom as the progression itself: session → ORB → bias → liquidity
→ sweep → displacement → FVG → blocks → structure → location → retest →
confirmation → confluence → quality → path → plan → state → **next condition**.

The last row is the one that matters most. The model is explicit about waiting:
`NEXT CONDITION` always names the single thing that has to happen next.

`DELTA` and `ABSORPTION` permanently read **N/A · NO ORDER FLOW**. TradingView
does not provide true bid/ask order-flow data, and bar volume is never
substituted for it. They are excluded from the confluence denominator rather
than quietly scored as zero.

Turn on **Show checklist panel** (group 14) for the Northstar checklist with
live ticks.

---

## Confluence: X / 9

Nine available factors, one point each:

liquidity sweep · displacement · FVG · order block · rejection block ·
daily bias alignment · premium/discount · key open · clean target path

CISD was removed by decision D-006. Delta divergence and absorption are
unavailable in TradingView. Those three are named on screen rather than hidden.

**The count is an inventory of present conditions, not a probability.** It is
not a win rate and must not be read as one until it has been measured against
outcomes on a real sample.

---

## Quality score

Nine weighted components (context, liquidity, displacement, structure, retest,
confirmation, confluence, risk geometry, target quality) producing 0–100, banded
INVALID / LOW / DEVELOPING / HIGH / A+.

**The weights are declared priors, not fitted values.** They exist to be
measured and replaced, and every one of them is an input.

---

## Risk figures

Entry, stop, TP1, TP2, R and an indicative contract count are **displayed for
reference only**. The 10-point minimum / 30-point maximum / ORB-third stop
filters are **off by default**, because as specified they are mutually
unsatisfiable for any ORB below 30 points — see `docs/RISK_RULES.md` §1.1.
Turning them on will make small-range days produce no setups. That is a real
decision, not a bug, and it is left to you.

---

## Alerts

Create one alert on the indicator using **"Any alert() function call"** to
receive every enabled event. Which events fire is controlled in group 16.

Format is switchable:

* **Text** — a readable Telegram/phone-friendly message.
* **JSON** — the documented Northstar event envelope (`docs/EVENT_SCHEMA.md`),
  so a webhook backend can be added later without touching this script.

Four classic `alertcondition` entries (sweep, displacement, retest, entry ready)
are also available in the standard alert dialog.

All alerts fire **on bar close only**.

---

## Non-repainting guarantees

1. State advances only on confirmed bars (`barstate.isconfirmed`).
2. Pivots use `ta.pivothigh` / `ta.pivotlow` and are published `pivot_right`
   bars after they form. That lag is deliberate — it is what makes them final.
3. There is **no `request.security()` call anywhere in the script**, so no
   higher-timeframe value can leak backwards into an earlier bar.
4. The opening range is stored once and frozen when its window closes.
5. Everything is computed from data available at that bar and nothing else.

The cost is honest: signals appear a few bars later than a repainting script
would show them in hindsight. That is the correct trade.

---

## Input groups

| Group | Controls |
|---|---|
| 01 · Session | timezone, ORB / execution / overnight windows, flat time |
| 02 · Opening range | min and max size in points |
| 03 · Liquidity | penetration, reclaim window, wick and close-back rules, which pools to track, equal-high tolerance, both-sides-swept behaviour |
| 04 · Abnormal wick | wick/range ratio, absolute size, ATR-relative bar size |
| 05 · Displacement | qualifying and strong scores, leg length, ATR target, all eight component weights |
| 06 · Imbalance / FVG | minimum size, displacement requirement, mitigation fraction, age |
| 07 · Structure | pivot left/right, break on close or wick, timeout |
| 08 · Blocks | order block zone construction, lookback, rejection wick minimum |
| 09 · Daily bias | threshold and six component weights |
| 10 · Location | equilibrium band, key open proximity |
| 11 · Retest | zone padding, >1R invalidation, timeout, failure buffer, preferred zone |
| 12 · Confluence & quality | minimum confluence, minimum quality, nine quality weights |
| 13 · Risk & targets | stop buffer, optional stop filters, TP1/TP2 sources, path severity, point value |
| 14 · Display | dashboard, checklist, levels, zones, markers, plan lines |
| 15 · Colours | institutional palette, all overridable |
| 16 · Alerts | format and which events fire |

Changing the **dashboard position** input takes effect after a chart refresh —
Pine tables cannot be repositioned in place.

---

## Known limitations, stated plainly

* Every threshold is a **starting point, not a validated optimum**. They are
  inputs precisely so they can be tested and changed. See
  `docs/AMBIGUITY_REGISTER.md` for what each one is likely to get wrong.
* Higher-timeframe structure is **not** part of the bias score. Including it
  would require `request.security()`, which is the most common source of
  lookahead bugs in Pine. Bias is computed from on-chart levels only.
* There is no news filter. The News Reversal classification is inferred from an
  abnormal wick within five bars of the sweep, not from an economic calendar.
* Hypothetical fill tracking (entry → TP1 → runner → exit) is for visual review.
  It assumes touch fills with no slippage and is not a backtest.
* Order flow is absent and is reported as absent.
