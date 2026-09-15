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
5. **Turn on Extended Trading Hours.** Right-click the chart → **Settings** →
   **Symbol** tab → tick **Extended trading hours** (newer layouts also have an
   ETH/RTH toggle next to the symbol name).

### Extended hours is not optional

With extended hours **off**, TradingView shows only the regular session —
06:30–13:00 PT (09:30–16:00 ET) — and hides everything else. Nothing is deleted;
those bars are simply not being displayed, and an indicator can only read the
bars the chart gives it.

That breaks this model completely, because almost everything it needs happens
outside regular hours:

| Needs | Window | Inside regular hours? |
|---|---|---|
| Opening range | 05:00–05:15 PT | **No** — zero bars, no range can be built |
| Execution window | 05:30–08:00 PT | **No** — ends an hour before RTH opens |
| Overnight high / low | 15:00–05:00 PT | **No** |
| Previous-day high / low | 15:00–14:00 PT | Partly |

If the ORB window passes with no bars in it, the dashboard says so directly:
`ORB STATUS: NO BARS IN ORB WINDOW — TURN ON EXTENDED HOURS`.

Chart timezone does not matter — the script converts internally. All session
inputs are interpreted in the **Timezone** input, which defaults to
`America/Los_Angeles`.

---

## Session defaults (Pacific, per `docs/DECISIONS.md`)

| Window | Pacific | Eastern |
|---|---|---|
| Trading day (previous-day high/low/open/close) | **15:00 – 14:00** next day | 18:00 – 17:00 |
| Overnight high / low | 15:00 – 05:00 | 18:00 – 08:00 |
| Opening range (locks at the end) | **05:00 – 05:15** | 08:00 – 08:15 |
| Execution window | 05:30 – 08:00 | 08:30 – 11:00 |
| Flat | 08:30 | 11:30 |

The trading day is the CME futures session, so the 14:00–15:00 PT hour is the
maintenance halt and carries no bars. Sessions are labelled by the date they
close on, per CME convention. Boundaries are resolved from TradingView session
strings, so they stay correct across DST.

Change any of them in the **01 · SESSION** input group.

The model's daily reset — state machine, opening range, liquidity pools, key
opens — happens at **15:00 PT**, not at local midnight. From 15:00 PT the
dashboard reads `PRE-MARKET` for the next morning's opening range.

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

### Setting one up

1. With the indicator on the chart, click the **alarm clock** icon (or right-click
   the chart → **Add alert**).
2. **Condition** → select **Northstar Motion Model**.
3. In the dropdown below it, choose **Any alert() function call** for the whole
   progression, or pick a single named condition (e.g. *Northstar · Entry ready*)
   to receive only that one.
4. **Notifications** tab → tick **Push notification** for your phone (needs the
   TradingView app installed and signed in), and/or email or webhook.
5. Set expiry to **open-ended** if your plan allows it.

Everything fires on **bar close**, so on a 1-minute chart a signal reaches you
within a minute of the bar that produced it.

Active-alert limits are set by your TradingView plan — the free tier allows very
few, so if you want the full progression rather than a single event, use one
alert on *Any alert() function call* and control which events fire with the
group 16 toggles.

### Which alert to use for "tell me when it's ready"

| You want | Use |
|---|---|
| Only the finished plan | *Northstar · Entry ready*, or leave only **Alert · entry ready** on |
| An early heads-up while it develops | **Alert · confluence threshold reached** (default 6 of 9) |
| The whole story as it unfolds | *Any alert() function call* with several toggles on |

The confluence alert fires **once per setup**, the first time the count reaches
your threshold. It is deliberately not an entry — the plan is not final at that
point, and the setup can still be invalidated by the retest rules.

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
| 01 · Session | timezone, trading day, ORB / execution / overnight windows, flat time |
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
* The order block is captured on the bar displacement confirms, and the
  rejection block on the bar that made the swept extreme. Both are recorded
  where they form rather than reconstructed later, so their geometry does not
  depend on how long the setup takes to develop.
* Hypothetical fill tracking (entry → TP1 → runner → exit) is for visual review.
  It assumes touch fills with no slippage and is not a backtest.
* Order flow is absent and is reported as absent.
