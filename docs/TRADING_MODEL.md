# NORTHSTAR MOTION MODEL™ — EXACT ALGORITHMIC DEFINITIONS

Deliverable **(10) exact algorithmic definitions**. Status: PROPOSED.

Notation: bars are indexed `i`, current closed bar is `t`. `O,H,L,C,V` are that
bar's open/high/low/close/volume. `Δ` is a configurable parameter (see
[`PARAMETERS.md`](PARAMETERS.md)). All prices are in NQ index points; NQ tick =
0.25 points, `point_value` = $20 (NQ) / $2 (MNQ).

**Every definition below that required an invented threshold is cross-referenced
to an entry in [`AMBIGUITY_REGISTER.md`](AMBIGUITY_REGISTER.md) as `[A-nn]`.
Nothing was silently invented.**

Working assumption pending approval: **the execution timeframe is 1-minute**,
and displacement/FVG/CISD/structure are evaluated on 1m unless a parameter says
otherwise `[A-01]`.

---

## 1. Session and clock

```
tz                = America/New_York (DST-aware, never fixed UTC offset)
premarket_window  = [07:00, 08:00)
orb_window        = [08:00, 08:15)          # bars whose OPEN time ∈ window
orb_lock_time     = 08:15:00
execution_window  = [08:30, 11:00)
no_new_entries    = 11:00:00
flat_time         = 11:30:00
```

A bar belongs to a window by its **open timestamp**. `session_date` is the ET
calendar date of the RTH session. Storage is UTC; every displayed time is ET.

Globex/overnight definition (pending confirmation `[A-02]`):
```
prev_day_session  = [18:00 ET (D-2), 17:00 ET (D-1)]   # CME futures day
overnight_window  = [18:00 ET (D-1), 08:00 ET (D)]
midnight_open     = open of the 00:00 ET bar (D)
daily_open        = open of the 18:00 ET bar (D-1)     # CME futures day open
```

---

## 2. levelEngine — static levels

```
PDH = max(H) over prev_day_session
PDL = min(L) over prev_day_session
PDO = open of first bar of prev_day_session
PDC = close of last bar of prev_day_session
ONH = max(H) over overnight_window
ONL = min(L) over overnight_window
ON_RANGE = ONH − ONL
MIDNIGHT_OPEN = O of 00:00 ET bar
DAILY_OPEN    = O of 18:00 ET bar (D-1)
OPEN_0830     = O of 08:30 ET bar        (available only from 08:30)
OPEN_0930     = O of 09:30 ET bar        (available only from 09:30)
```

`OPEN_0830` and `OPEN_0930` are `Unavailable` before their time — the loader
enforces `available_at`, so pre-market code cannot reference them.

### 2.1 Swing pivots (no repaint)

```
pivot_high(i) ⇔ H[i] = max(H[i−L .. i+Rt])  and strictly ≥ neighbours
pivot_low(i)  ⇔ L[i] = min(L[i−L .. i+Rt])
L  = pivot_left  (default 3)   [A-09]
Rt = pivot_right (default 3)   [A-09]
```

A pivot at bar `i` is only **published** at bar `i+Rt`. Its event timestamp is
`time[i+Rt]`; its price is `H[i]`/`L[i]`. This introduces a deliberate `Rt`-bar
detection lag and guarantees no repainting.

### 2.2 Equal highs / equal lows

```
equal_highs(a,b) ⇔ |H[a] − H[b]| ≤ eq_tol_pts  ∧  |a − b| ≥ eq_min_sep_bars
eq_tol_pts       default 2.0 points   [A-11]
eq_min_sep_bars  default 3
```
A cluster of ≥ `eq_min_count` (default 2) qualifying pivots forms one pool whose
level is the **mean** of the members and whose strength = member count.

---

## 3. orbEngine

```
orb_high = max(H) over orb_window
orb_low  = min(L) over orb_window
R        = orb_high − orb_low
equilibrium = (orb_high + orb_low) / 2

at t ≥ 08:15:00 →  LOCK.  orb_high, orb_low, R, equilibrium become immutable.

classification:
  R < orb_min (10)              → STAND_DOWN,  reason ORB_TOO_SMALL
  orb_min ≤ R ≤ orb_max (100)   → LIVE
  R > orb_max (100)             → STAND_DOWN,  reason ORB_TOO_LARGE
```

Note: the 10 and 100 bounds are absolute points, therefore **volatility-regime
dependent** — a 40-point range means something different at VIX 12 vs VIX 35.
Flagged as `[A-23]`; an ATR-normalised alternative is proposed there but the
literal spec values are the default.

---

## 4. liquidityEngine

A **pool** is `(level, kind, side, strength, created_at)` where `kind ∈ {PDH,
PDL, ONH, ONL, ORB_HIGH, ORB_LOW, SWING_HIGH, SWING_LOW, EQH, EQL}` and
`side ∈ {BUY_SIDE (above price), SELL_SIDE (below price)}`.

Pool lifecycle:

```
IDENTIFIED  → pool exists, untouched
SWEPT       → penetration condition met
RECLAIMED   → reclaim condition met within reclaim window   (the tradeable state)
FAILED      → penetration held; price accepted beyond the level
```

### 4.1 Sweep (penetration)

For a sell-side pool at level `P` (a low):
```
penetrated(t) ⇔ L[t] < P − pen_min_pts
pen_min_pts default 0.25 (1 tick)        [A-03]
```
Optional stricter mode `pen_mode = atr`: `L[t] < P − pen_atr_mult · ATR(14,1m)`.

### 4.2 Rejection + reclaim (what makes it a *sweep* rather than a break)

Within `reclaim_window_bars` (default 3 bars, inclusive of the penetrating bar):
```
reclaimed(t..t+k) ⇔ ∃ j ≤ t+k :  C[j] > P + reclaim_buffer_pts
reclaim_buffer_pts default 0.0           [A-03]
```
and, if `require_wick = true` (default true):
```
lower_wick(t) / range(t) ≥ wick_ratio_min   (default 0.50)    [A-04]
```
and, if `require_close_back = true` (default true):
```
C[t] ≥ P      # the penetrating bar closes back inside the prior range
```

`SWEPT ∧ reclaimed` → `RECLAIMED`. `SWEPT ∧ ¬reclaimed within window` → `FAILED`
(and, for ORB levels, this is the `ORB failure / one-sided expansion` case that
feeds the Continuation setup rather than the Reversal setup).

Symmetric definitions apply to buy-side pools with inequalities reversed.

### 4.3 Both-sides-swept detection

```
both_swept ⇔ orb_high pool ∈ {SWEPT, RECLAIMED, FAILED}
           ∧ orb_low  pool ∈ {SWEPT, RECLAIMED, FAILED}
           ∧ both occurred at t ≥ 08:30              [A-24]
→ default action: NO TRADE for the remainder of the session.
```

---

## 5. wickEngine — abnormal wick

```
range      = H − L
body       = |C − O|
upper_wick = H − max(O,C)
lower_wick = min(O,C) − L

wick_range_ratio_up   = upper_wick / range        (range > 0)
wick_range_ratio_down = lower_wick / range
wick_body_ratio_up    = upper_wick / max(body, tick)      # guarded
```

```
abnormal_wick(side) ⇔ wick_range_ratio(side) ≥ wick_ratio_min (0.50)   [A-04]
                    ∧ wick_points(side)      ≥ wick_min_pts   (5.0)
                    ∧ range                  ≥ wick_min_range_atr · ATR(14)  (0.8)
```

The two extra conditions exist because a 60% wick on a 1.5-point doji bar is
noise, not a liquidity event. All three are configurable; the ratio default of
0.50 is explicitly labelled a **research starting point, not a validated
optimum**.

---

## 6. displacementEngine

A **displacement leg** is a run of `n ∈ [disp_min_bars, disp_max_bars]`
consecutive closed bars (defaults 1–5) in one direction starting at the sweep
bar or the bar after it.

Per-leg raw measures:
```
leg_move        = |C[end] − O[start]|
leg_body_sum    = Σ |C[i] − O[i]|
leg_range       = max(H) − min(L)  over the leg
atr             = ATR(atr_len=14) on the execution TF, computed at start−1
body_ratio      = leg_body_sum / max(leg_range, tick)
atr_ratio       = leg_move / atr
avg_body_ratio  = mean_body(leg) / SMA(|C−O|, 20)[start−1]
consistency     = (# bars closing in leg direction) / n
close_location  = (C[end] − min(L)) / leg_range        for bullish
                  (max(H) − C[end]) / leg_range        for bearish
opp_wick_ratio  = 1 − (Σ opposing wick) / leg_range
```

**displacementScore (0–100)** — weighted, all weights configurable `[A-05]`:

| Component | Formula | Weight |
|---|---|---|
| ATR expansion | `clamp(atr_ratio / disp_atr_target, 0, 1)`, target 1.5 | 30 |
| Body dominance | `clamp((body_ratio − 0.4) / 0.5, 0, 1)` | 20 |
| Avg-body expansion | `clamp((avg_body_ratio − 1) / 2, 0, 1)` | 15 |
| Directional consistency | `consistency` | 10 |
| Close location | `clamp((close_location − 0.5) / 0.4, 0, 1)` | 10 |
| Opposing-wick cleanliness | `clamp((opp_wick_ratio − 0.5) / 0.4, 0, 1)` | 5 |
| Structure break in leg | `1 if BOS/MSS else 0` | 5 |
| FVG created in leg | `1 if ≥1 FVG ≥ min size else 0` | 5 |

```
displacementScore = Σ (component · weight)     → 0..100

classification:
  score <  30                 → NONE
  30 ≤ score < disp_min (55)  → WEAK
  disp_min ≤ score < 80       → QUALIFIED
  score ≥ 80                  → STRONG
```

Only `QUALIFIED` or `STRONG` advances the state machine. `disp_min` default 55
is a starting point `[A-05]`.

---

## 7. fvgEngine — imbalance / fair value gap

Three-bar formation ending at bar `t`:
```
bullish_fvg ⇔ L[t] > H[t−2]      gap = (low: H[t−2], high: L[t])
bearish_fvg ⇔ H[t] < L[t−2]      gap = (low: H[t],   high: L[t−2])
size        = gap.high − gap.low
midpoint    = (gap.high + gap.low) / 2
```

Qualification:
```
valid ⇔ size ≥ fvg_min_pts (default 3.0)          [A-06]
      ∧ (¬fvg_require_displacement ∨ leg is QUALIFIED+)
```

Mitigation (for a bullish FVG; mirror for bearish):
```
fill_pct(t) = clamp((gap.high − min(L[c..t])) / size, 0, 1)

CREATED             at formation bar
ACTIVE              fill_pct = 0
PARTIALLY_MITIGATED 0 < fill_pct < mitig_full (default 1.0)
FULLY_MITIGATED     fill_pct ≥ mitig_full
INVALIDATED         C[t] < gap.low        # traded fully through and closed beyond
```
`mitig_full = 1.0` means "wick through the far edge". A common alternative is
0.5 (midpoint) — configurable, flagged `[A-06]`.

---

## 8. structureEngine — BOS / MSS

Using published (non-repainting) pivots from §2.1:

```
BOS_bullish  ⇔ C[t] > last_confirmed_pivot_high  ∧ prevailing trend bullish
BOS_bearish  ⇔ C[t] < last_confirmed_pivot_low   ∧ prevailing trend bearish
MSS_bullish  ⇔ C[t] > last_confirmed_pivot_high  ∧ prevailing trend bearish
MSS_bearish  ⇔ C[t] < last_confirmed_pivot_low   ∧ prevailing trend bullish
```

`prevailing trend` = sign of the last two confirmed pivot-pair displacements
(HH/HL = bullish, LH/LL = bearish, otherwise neutral). Break confirmation
requires a **close** beyond the pivot, not a wick (`structure_break_on = close`,
configurable to `wick`) `[A-09]`.

For the Northstar sequence, the relevant break after a sweep is normally an
**MSS** (trend reversal), which is why `structureEngine` reports both and the
state machine accepts either in the leg direction.

---

## 9. orderBlockEngine

```
bullish_OB = the LAST bearish candle (C < O) at or before the start of a
             QUALIFIED+ bullish displacement leg, searching back at most
             ob_lookback_bars (default 5)
bearish_OB = mirror
```

Zone boundaries (`ob_zone_mode`, default `body_to_wick`) `[A-07]`:
```
body_only   : [min(O,C), max(O,C)]
body_to_wick: bullish → [L, max(O,C)] ; bearish → [min(O,C), H]
full_range  : [L, H]
```

Quality flags stored per OB: `has_displacement`, `has_fvg`, `has_bos`,
`unmitigated`, `mitigation_count`. `OB_quality = count(flags)/4`.

```
mitigation: a bullish OB is mitigated when L[t] ≤ ob.high (first touch of zone)
status: ACTIVE → MITIGATED (count++) → INVALIDATED if C[t] < ob.low (bullish)
```

---

## 10. rejectionBlockEngine

Formed by a bar with an abnormal wick (§5) that penetrated a liquidity pool
(§4.1) and reclaimed it.

```
extreme        = L (bullish rejection) / H (bearish rejection)
body_boundary  = min(O,C) / max(O,C)
zone           = [extreme, body_boundary]         (bullish)
penetration    = P − L    (how far beyond the pool level it reached)
wick_pct       = lower_wick / range
reclaimed      = C ≥ P
direction      = BULLISH | BEARISH
```
Requires `wick_pct ≥ rb_wick_min` (default 0.50) and
`penetration ≥ rb_pen_min` (default 0.25) `[A-08]`.

---

## 11. cisdEngine — Change In State Of Delivery

**This is the definition most in need of your validation `[A-10]`.**

Proposed operational definition (bullish; mirror for bearish):

1. Identify the **delivery leg**: the most recent unbroken run of consecutive
   *bearish* closed candles (`C < O`) immediately preceding the sweep low, of
   length ≥ 1 and ≤ `cisd_leg_max_bars` (default 10).
2. The **CISD reference price** is the **open of the first candle of that
   bearish run** (`cisd_ref_mode = leg_first_open`, default). Alternatives,
   selectable by config: `leg_last_open` (open of the final bearish candle) and
   `swept_candle_open`.
3. **Bullish CISD confirmed** at bar `t` when:
```
C[t] > cisd_ref_price
∧ C[t] > O[t]                                   (the confirming bar is bullish)
∧ body(t) ≥ cisd_min_body_atr · ATR(14)         (default 0.5)  "strong candle"
∧ t occurs after the sweep bar
∧ t − sweep_bar ≤ cisd_max_bars_after_sweep     (default 15)
```

CISD is **confirmation only**. It can never, alone, produce `ENTRY_READY`.
The preferred canonical sequence is `SWEEP → CISD → RETEST`.

---

## 12. biasEngine — daily bias

`biasScore ∈ [−100, +100]`, computed pre-market from weighted components; all
weights configurable `[A-12]`:

| # | Component | Score contribution |
|---|---|---|
| 1 | Price vs `DAILY_OPEN` | `±sign · clamp(|C−DO| / (0.5·ADR), 0, 1) · 15` |
| 2 | Price vs `MIDNIGHT_OPEN` | same shape, weight 10 |
| 3 | Position in prev-day range | `((C − PDL)/(PDH − PDL) − 0.5) · 2 · 15` |
| 4 | Prev-day close location | `((PDC − PDL)/(PDH − PDL) − 0.5) · 2 · 10` |
| 5 | HTF structure (4H BOS/MSS state) | `±20` bullish/bearish, 0 neutral |
| 6 | Overnight positioning (ON range mid vs PD mid) | `±10` |
| 7 | Nearest untapped liquidity objective (asymmetry of distance to ONH/PDH vs ONL/PDL) | `±10` |
| 8 | Premium/discount in prev-day dealing range | `±10` |
| 9 | News context | `0` by default; only shifts bias if a directional prior is explicitly configured (normally it only gates trading, not direction) |

```
BULLISH if biasScore ≥ bias_thresh (+25)
BEARISH if biasScore ≤ −bias_thresh
NEUTRAL otherwise
```

The dashboard must render bias as `BULLISH +68` — a **working assumption with a
magnitude**, never as a probability or a prediction.

---

## 13. premiumDiscountEngine

```
equilibrium = (orb_high + orb_low) / 2          # per spec, ORB-based
location = DISCOUNT if price < equilibrium
           PREMIUM  if price > equilibrium
           EQUILIBRIUM if |price − eq| ≤ eq_band (default 2.0 pts)
```
Confluence credit: `LONG ∧ DISCOUNT` or `SHORT ∧ PREMIUM` → +1.
An optional second reference (the swept dealing range: sweep extreme →
displacement extreme) is available as `pd_reference = dealing_range` `[A-14]`.

---

## 14. keyOpenEngine

```
key_opens = {MIDNIGHT_OPEN, DAILY_OPEN, OPEN_0830, OPEN_0930}   (available ones only)
aligned ⇔ ∃ k ∈ key_opens : |entry_price − k| ≤ key_open_prox (default 5.0 pts)
→ +1 confluence, and the matched open is named in the alert.
```

---

## 15. retestEngine

Valid retest zones, ranked by `retest_zone_priority` (default order):
```
1. FVG midpoint  ± zone_pad
2. FVG proximal boundary (near edge)
3. Order Block zone
4. Rejection Block zone
5. Broken ORB level ± zone_pad
```

```
retest_detected ⇔ price trades into the zone:
                  bullish: L[t] ≤ zone.high  ∧ L[t] ≥ zone.low − tolerance
                  (i.e. any touch of the zone band)
zone_pad default 1.0 pt, tolerance default 0.5 pt

R_ref        = |proposed_entry − proposed_stop| evaluated at displacement end
MFE_R        = (max(H) since leg end − leg_end_price) / R_ref     (bullish)
invalidate ⇔ MFE_R > mfe_max_R (default 1.0)              → MOVED_1R_BEFORE_RETEST
timeout    ⇔ bars since leg end > retest_timeout_bars (default 20)  → NO_RETEST
zone_failed⇔ C[t] < zone.low − fail_buffer (bullish)                → RETEST_FAILED

states: WAITING → RETEST_DETECTED → RETEST_ACTIVE → ENTRY_ZONE_TOUCHED
                → ENTRY_QUALIFIED | RETEST_FAILED
```

**"Never chase displacement"** is enforced structurally: there is no path from
`DISPLACEMENT_DETECTED` to `ENTRY_READY` that does not pass through a retest.

---

## 16. confirmationEngine

Independent booleans; **no aggregation happens here**:

```
conf_structure   : BOS/MSS in trade direction after the retest touch
conf_cisd        : §11 confirmed
conf_rejection   : rejection candle inside the zone (wick ≥ rb_wick_min, close back out)
conf_order_block : entry zone is an unmitigated OB
conf_fvg         : entry zone is an unmitigated/partially mitigated FVG
conf_delta_div   : §17 — Measured only, else UNAVAILABLE
conf_absorption  : §18 — Measured only, else UNAVAILABLE
conf_volume      : V[t] ≥ vol_mult · SMA(V,20)   (default 1.5) — weak, informational [A-17]
```

`min_confirmations` (default 1) must fire before `CONFLUENCE_CHECK`.

---

## 17. Delta divergence (gated)

```
bullish_div ⇔ price makes LL vs prior swing low ∧ CVD makes HL
bearish_div ⇔ price makes HH vs prior swing high ∧ CVD makes LH
```
Requires a real order-flow provider. With `NullOrderFlowProvider` the result is
`Unavailable("no order flow feed")`, the dashboard shows `DELTA: N/A`, and the
confluence factor scores 0. **Delta is never derived from bar volume.**

---

## 18. Absorption (gated)

```
buying_absorption  ⇔ aggressive buy volume ≥ absorb_vol_pctile (default 90th)
                   ∧ price progression over the window ≤ absorb_max_move (default 0.25·ATR)
                   ∧ passive sell size at the level ≥ absorb_passive_min
selling_absorption ⇔ mirror
```
Requires bid/ask-classified trade data (footprint/DOM). Absent that:
**`ORDER FLOW DATA UNAVAILABLE`**. Bar volume with a small candle body is *not*
absorption and must never be reported as such.

---

## 19. confluenceEngine

Twelve factors, +1 each, evaluated at the moment of entry qualification:

| # | Factor | Condition |
|---|---|---|
| 1 | Liquidity sweep | a pool reached `RECLAIMED` in this setup |
| 2 | Displacement | `displacementScore ≥ disp_min` |
| 3 | FVG | valid, unmitigated or ≤50% mitigated, in the trade direction |
| 4 | Order block | entry zone is an unmitigated OB |
| 5 | Rejection block | rejection block present at the swept extreme |
| 6 | CISD | confirmed in the trade direction |
| 7 | Daily bias | `sign(biasScore)` matches direction ∧ `|biasScore| ≥ bias_thresh` |
| 8 | Premium/discount | long∧discount or short∧premium |
| 9 | Key open | within `key_open_prox` of a key open |
| 10 | Delta divergence | Measured ∧ aligned (else 0, marked N/A) |
| 11 | Absorption | Measured ∧ aligned (else 0, marked N/A) |
| 12 | Clean target path | `targetPathEngine` returns `PATH_CLEAR` |

```
confluence = Σ factors ∈ [0,12]
gate: confluence ≥ min_confluence (default 3)
display: "X / 12" plus the named list of which factors are present and absent.
```

**The count is an inventory, not a probability.** The dashboard is forbidden
from rendering it as a percentage or win rate until `analyticsEngine` has
published measured outcomes with an adequate sample.

---

## 20. targetPathEngine

Scan the price interval `(entry → TP2)` for obstructions in the trade direction:

```
obstruction types:
  - opposing FVG   with size ≥ path_fvg_min (default 3.0) and unmitigated
  - opposing OB    unmitigated, quality ≥ 0.5
  - major liquidity pool of the OPPOSITE side (e.g. a long targeting through PDL)
  - confirmed swing high/low with ≥ path_pivot_touches (default 2) touches

severity(obstruction) = size_factor · proximity_factor · strength_factor
PATH_CLEAR       if Σ severity ≤ path_clear_max (default 1.0)
TARGET_OBSTRUCTED otherwise → default action INVALIDATE            [A-13]
```
Obstructions between entry and **TP1** are weighted 2×: an obstruction before
the first scale-out matters far more than one before the runner target.

---

## 21. riskEngine

```
bullish: stop = swept_low  − stop_buffer      (default buffer 2.0 pts)
bearish: stop = swept_high + stop_buffer
stop_distance = |entry − stop|

valid ⇔ stop_min (10) ≤ stop_distance ≤ stop_max (30)
      ∧ stop_distance ≤ R_orb / 3
otherwise → NO TRADE, reason STOP_INVALID
```
`entry` is the retest fill reference (zone touch price or zone midpoint,
`entry_ref` configurable, default `zone_touch`).

**Note the structural tension, flagged `[A-15]`:** with `R_orb = 10..100` the
`R/3` rule caps the stop at 3.3–33.3 pts while `stop_min` is 10 pts. For any ORB
below 30 points the `R/3` rule is *stricter* than the 10-point minimum, so
`R_orb < 30` makes a compliant trade impossible. Either the effective minimum
ORB is 30 points, or one of the two rules needs adjustment. This needs your
decision before implementation.

---

## 22. positionSizingEngine

```
risk_per_contract = stop_distance · point_value + commission_rt + slippage_est·point_value
max_contracts     = floor(risk_per_trade / risk_per_contract)
actual_risk       = max_contracts · risk_per_contract
if max_contracts < 1 → NO TRADE, reason SIZE_ZERO
if second attempt of the day → risk_per_trade ·= 0.5   (per daily governor)
```
`point_value`: NQ 20, MNQ 2. `tick_size` 0.25. `commission_rt` and
`slippage_est` are per-instrument config, never zero in backtests.

---

## 23. Targets

```
R = stop_distance
TP1 = entry ± 2R      → close tp1_fraction (0.50), move stop to breakeven
TP2 = per target_hierarchy, first satisfying option wins:
      1. opposite ORB extreme, if distance ≥ tp2_min_R (default 3R)
      2. 4R fixed
      3. measured move = displacement leg length projected from the retest low/high
```
Runner (after TP1): trail stop to the last **confirmed** 1-minute pivot low
(long) / pivot high (short), using the non-repainting pivots of §2.1, never
loosening. Hard time stop at 11:30 ET.

---

## 24. Quality score (0–100)

Nine components, weights configurable `[A-25]`:

| Component | Measure | Weight |
|---|---|---|
| Context | bias alignment · `|biasScore|/100` | 12 |
| Liquidity | pool strength + reclaim cleanliness | 12 |
| Displacement | `displacementScore/100` | 15 |
| Structure | BOS/MSS confirmed, cleanliness | 10 |
| Retest | depth into zone, time-to-retest, MFE headroom | 12 |
| Confirmation | count/strength of triggers fired | 10 |
| Confluence | `confluence/12` | 12 |
| Risk | how comfortably the stop sits inside its bounds | 9 |
| Target quality | path clearness + R available to TP2 | 8 |

```
0–39   INVALID
40–59  LOW QUALITY
60–74  DEVELOPING
75–89  HIGH QUALITY
90–100 A+ SETUP
gate: quality ≥ min_quality (default 60)
```

These weights are **declared priors, not fitted values**. Their purpose is to be
measured against outcomes in the archive and then replaced.

---

## 25. Setup type classification

```
NEWS_REVERSAL   ⇔ a high-impact release ∈ [08:30 − 1m, 08:30 + news_window (5m)]
                 ∧ sweep occurred within news_window of the release
                 ∧ displacement direction is counter to the sweep
                 ∧ abnormal wick present
CONTINUATION    ⇔ any other qualifying sequence (sweep → displacement → FVG →
                 structure → retest) with displacement in the direction of bias
UNCLASSIFIED    ⇔ neither; permitted to trade only if allow_unclassified = false → NO TRADE
```
`[A-18]` covers the overlap case (both patterns arguably present).

---

## 26. Rejection reason codes (canonical)

```
ORB_TOO_SMALL · ORB_TOO_LARGE · NO_LIQUIDITY · NO_SWEEP · BOTH_SIDES_SWEPT
NO_RECLAIM · NO_DISPLACEMENT · NO_FVG · NO_STRUCTURE · NO_RETEST
MOVED_1R_BEFORE_RETEST · RETEST_FAILED · NO_CONFIRMATION
CONFLUENCE_BELOW_THRESHOLD · QUALITY_BELOW_THRESHOLD · STOP_INVALID
SIZE_ZERO · TARGET_OBSTRUCTED · DAILY_LOSS_LIMIT · MAX_TRADES · SESSION_CLOSED
NEWS_FILTER · HOLIDAY_SESSION · DATA_INTEGRITY · ENTRY_EXPIRED · STOP_FIRST
```
Every code carries a human sentence for the dashboard and Telegram, e.g.
`STOP_INVALID → "Stop 34.5 pts exceeds the 30-point maximum."`
