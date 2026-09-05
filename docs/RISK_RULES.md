# RISK RULES

Status: PROPOSED. Contains **blocking question B2** — an internal contradiction
in the specification that must be resolved before any risk code is written.

---

## 1. Stop placement

```
LONG :  stop = swept_low  − stop_buffer_pts
SHORT:  stop = swept_high + stop_buffer_pts
stop_buffer_pts default 2.0
stop_distance   = |entry − stop|
```

Validity (all must hold):

| Rule | Value | Source |
|---|---|---|
| `stop_distance ≥ 10 pts` | `risk.stop_min_pts` | SPEC |
| `stop_distance ≤ 30 pts` | `risk.stop_max_pts` | SPEC |
| `stop_distance ≤ R_orb / 3` | `risk.stop_orb_fraction` | SPEC |

Failure of any rule → `STOP_INVALID` → **NO TRADE**. No widening, no
"just this once", no discretionary override path in the code.

### 1.1 ⚠ The contradiction (B2)

`R_orb` is LIVE from 10 to 100 points. The `R_orb/3` rule therefore caps the
stop between 3.33 and 33.3 points, while `stop_min` demands at least 10.

```
R_orb  = 10  →  max stop  3.33  <  min stop 10   → impossible
R_orb  = 20  →  max stop  6.67  <  min stop 10   → impossible
R_orb  = 29  →  max stop  9.67  <  min stop 10   → impossible
R_orb  = 30  →  max stop 10.00  =  min stop 10   → exactly one legal value
R_orb  = 60  →  max stop 20.00                    → workable
R_orb  = 90  →  max stop 30.00  =  cap            → workable
R_orb  = 100 →  max stop 33.3 → capped at 30      → workable
```

**As written, every session with an ORB below 30 points is untradeable**, even
though the spec calls 10–100 "LIVE". Options:

| Option | Effect | Cost |
|---|---|---|
| (a) Effective minimum ORB = 30 | Rules stay literal and consistent | Loses every 10–30 pt session |
| (b) `stop_min = min(10, R_orb/3)` | Small ranges tradeable with tight stops | Stops below ~7 pts sit inside 1m noise |
| (c) Drop `R_orb/3` when `R_orb < 30` | Small ranges tradeable with 10 pt stops | The stop can exceed a third of the range — the rule's intent is lost |
| (d) ATR-scaled minimum instead of a fixed 10 | Regime-aware | New parameter to validate |

`risk.conflict_policy` implements this as `reject` (a) / `relax_min` (b) /
`relax_fraction` (c). **Default is `reject`, because silently loosening a risk
rule is the worst possible default.** I need your decision; I will first produce
the historical distribution of `R_orb` so you can see exactly how many sessions
each option costs.

---

## 2. Position sizing

Fixed-dollar risk, always rounded **down**.

```
risk_per_contract = stop_distance × point_value
                  + commission_rt
                  + slippage_est × point_value
max_contracts     = floor(effective_risk_budget / risk_per_contract)
actual_risk       = max_contracts × risk_per_contract

effective_risk_budget = risk_per_trade_usd × size_multiplier
size_multiplier = 1.0 on the first trade of the day
                = 0.5 on the second trade (per the daily governor)

if max_contracts < 1 → NO TRADE, reason SIZE_ZERO
```

Worked example (NQ, 29.75 pt stop, $500 risk, $4.50 RT, 1 pt slippage):
```
risk_per_contract = 29.75×20 + 4.50 + 1×20 = 595.00 + 24.50 = 619.50
max_contracts     = floor(500 / 619.50) = 0        → NO TRADE (SIZE_ZERO)
```
This is not a bug — it is the system correctly refusing a trade whose minimum
size exceeds the risk budget. With MNQ (`point_value = 2`) the same setup gives
`risk_per_contract = 62.70`, `max_contracts = 7`, `actual_risk = $438.90`.

**Implication worth stating plainly:** at $500 risk per trade and 10–30 point
stops, NQ requires roughly $700–$1,300 of risk budget for a single contract.
MNQ is the appropriate instrument until the risk budget supports NQ. The system
will say so explicitly rather than silently returning zero.

---

## 3. Targets

```
R   = stop_distance
TP1 = entry ± 2R          → close 50%, move stop to breakeven
TP2 = first satisfied option in the configured hierarchy:
        1. opposite ORB extreme, if it is ≥ tp2_min_R (3R) away
        2. fixed 4R
        3. measured move (displacement leg length projected from the retest)
```

The chosen source is recorded in `risk_evaluations.tp2_source` so per-target
analytics can compare them.

### Runner

After TP1: trail to the last **confirmed, non-repainting** 1-minute pivot low
(long) / pivot high (short). The trail never loosens. Hard time stop at 11:30 ET
regardless of position state.

---

## 4. Daily governor

```
max_trades_per_day        = 2
trade 1 WINS              → DAY DONE
trade 1 LOSES             → trade 2 permitted at 50% size
trade 2 LOSES             → DAY DONE
daily loss limit reached  → DAY DONE (checked before every entry)
t ≥ 11:00                 → NO NEW ENTRIES
t ≥ 11:30                 → FLAT, unconditionally
```

Open question **B11**: after an *invalidation with no fill*, does the attempt
count against `max_trades`? Proposed default: **no** — only filled trades count,
because the rule's purpose is limiting risk taken, not opportunities observed.
`daily.second_trade_after = loss_only` encodes this.

Breakeven and scratch outcomes: proposed to **not** consume the "first trade
wins → day done" rule but **do** consume a trade slot. So BE on trade 1 permits
trade 2 at full size (it was not a loss). Flag this if you disagree.

---

## 5. Hard no-trade filters

Evaluated in order; the first failure stops evaluation and is recorded:

```
1  ORB < 10 pts                           ORB_TOO_SMALL
2  ORB > 100 pts                          ORB_TOO_LARGE
3  Both ORB extremes swept                BOTH_SIDES_SWEPT
4  No qualified sweep                     NO_SWEEP
5  No qualified displacement              NO_DISPLACEMENT
6  No FVG when required                   NO_FVG
7  No structure confirmation              NO_STRUCTURE
8  No retest                              NO_RETEST
9  MFE > 1R before retest                 MOVED_1R_BEFORE_RETEST
10 Confluence < threshold                 CONFLUENCE_BELOW_THRESHOLD
11 Quality < threshold                    QUALITY_BELOW_THRESHOLD
12 Stop invalid                           STOP_INVALID
13 Contracts < 1                          SIZE_ZERO
14 Target obstructed                      TARGET_OBSTRUCTED
15 Daily loss limit reached               DAILY_LOSS_LIMIT
16 Two trades used / first trade won      MAX_TRADES
17 t ≥ 11:00                              SESSION_CLOSED
18 Restricted news window                 NEWS_FILTER
19 Holiday / half day / thin (if enabled) HOLIDAY_SESSION
20 Data gap or feed loss                  DATA_INTEGRITY
```

Every filter writes to the `rejections` ledger even when a later filter would
also have failed, so the daily review shows the *complete* picture of what was
missing, not just the first blocker.

---

## 6. Risk invariants (asserted at runtime)

1. No `ENTRY_READY` without a complete, persisted `TradePlan`.
2. `actual_risk ≤ risk_per_trade_usd × size_multiplier`, always.
3. `Σ open risk ≤ daily_loss_limit_usd − realised_loss`, always.
4. Only one position at a time. Ever.
5. No position may exist after 11:30 ET.
6. `contracts` is an integer ≥ 1 and derived only from the formula above.
7. The daily governor is checked **immediately before** entry, not only at
   qualification — conditions can change between the two.
8. In shadow mode, no `ExecutionProvider` method that mutates state may be
   called; `NullExecutionProvider` raises if one is.

---

## 7. What this engine deliberately does not do

* It does not average down, add to losers, or scale in.
* It does not move a stop away from price, at any time, for any reason.
* It does not re-enter an invalidated setup under a new ID to "get back in".
* It does not size up after a loss.
* It has no manual override path. If you want to trade something the system
  rejected, you do that yourself, outside the system, and the archive records
  that the system said no.
