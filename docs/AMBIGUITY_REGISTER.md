# AMBIGUITY REGISTER — DEFINITIONS REQUIRING VALIDATION

Deliverable **(11)**. Every entry follows the required template:
**Concept · Definition · Proposed mathematical implementation · Configurable
parameters · Expected false positives · Expected false negatives · Validation
method.**

Nothing here was decided unilaterally. Where I had to choose a default to make
the architecture concrete, the default is stated and marked as a *starting
point*, not a validated value.

---

## BLOCKING QUESTIONS — Phase 2 cannot start without these

| # | Question | Why it blocks |
|---|---|---|
| B1 | **Execution timeframe?** (`A-01`) 1m assumed. | Changes every engine's inputs. |
| B2 | **Stop rule conflict** (`A-15`): `stop ≥ 10 pts` and `stop ≤ R_orb/3` are mutually unsatisfiable for `R_orb < 30`. Which wins? | Makes ORB 10–30 untradeable as written. |
| B3 | **CISD reference leg** (`A-10`): first open of the bearish run, last open, or swept candle open? | Three different signals, three different backtests. |
| B4 | **Session boundaries** (`A-02`): is "previous day" the CME futures day (18:00–17:00 ET) or the RTH day (09:30–16:00 ET)? Is "daily open" 18:00 ET or 09:30 ET? | PDH/PDL/ONH/ONL/daily-open all shift. |
| B5 | **Data source and vendor** for research bars + live bars. | Determines roll handling, gaps, backfill, cost. |
| B6 | **Contract roll policy** (`A-21`): continuous back-adjusted, unadjusted stitched, or front-month only? | Historical level accuracy across quarterly rolls. |
| B7 | **Screenshot mechanism** (`A-19`): server-rendered charts vs headless TradingView capture. | Different build, different ToS exposure. |
| B8 | **News data source** and the exact "restricted news" list (`A-16`). | The 08:30 catalyst is central to the News Reversal setup. |
| B9 | **Both-sides-swept**: does a *pre-08:30* ORB sweep count toward "both swept"? (`A-24`) | Materially changes daily trade availability. |
| B10 | **Order-flow feed**: will one ever be attached, and which? | Decides whether factors 10 and 11 are permanently 0. |
| B11 | **Second-trade rule**: is trade 2 allowed after an *invalidation* (no fill), or only after a *loss*? | Governor semantics. |
| B12 | **ORB size bounds** (`A-23`): keep absolute 10/100 points, or ATR-normalise? | 10/100 are regime-dependent constants. |

---

## A-01 · Execution timeframe

* **Concept.** The specification never states the chart timeframe on which
  displacement, FVG, CISD, structure and retest are evaluated. Only the runner
  explicitly mentions 1-minute structure.
* **Definition.** A single primary execution timeframe `tf_exec` with optional
  higher-timeframe context `tf_context`.
* **Proposed implementation.** `tf_exec = 1m` for all L2/L3 engines (consistent
  with the 1-minute runner rule and a 15-minute ORB); `tf_context = 15m` for
  structure sanity; `tf_bias = 4H/1D` for `biasEngine` only.
* **Parameters.** `tf_exec`, `tf_context`, `tf_bias`, and per-engine overrides
  `fvg_tf`, `disp_tf`, `cisd_tf`, `structure_tf`.
* **False positives.** 1m produces many small FVGs and micro-displacements →
  more setups, more noise, more invalidations.
* **False negatives.** 5m would miss the fast 08:30 reaction entirely on some
  days; the whole reaction can be over inside two 5m bars.
* **Validation.** Run the identical model at 1m / 2m / 5m over the same sample;
  compare setup count, invalidation rate, expectancy, and time-to-entry. Choose
  on expectancy stability, not on raw win rate.

## A-02 · Session boundaries, prior day, overnight, "daily open"

* **Concept.** "Previous Day High/Low", "Overnight High/Low", "Daily Open" have
  no unique meaning in futures.
* **Definition.** Explicit windows, DST-aware.
* **Proposed implementation.** Prev day = CME futures day `[18:00 ET D−2,
  17:00 ET D−1]`; overnight = `[18:00 ET D−1, 08:00 ET D]`; daily open = open of
  the 18:00 ET bar; midnight open = open of the 00:00 ET bar.
* **Parameters.** `prev_day_mode ∈ {cme_day, rth}`, `overnight_start`,
  `overnight_end`, `daily_open_ref ∈ {1800, 0930, 0000}`.
* **False positives.** Using the CME day makes PDH/PDL wider → sweeps of them
  are rarer but more significant; a wider level may look "swept" by ordinary
  overnight drift.
* **False negatives.** Using RTH-only PDH/PDL ignores overnight extremes that
  actually hold the resting orders.
* **Validation.** For a sample of sessions, measure the reaction magnitude
  (MFE within 30m) after taps of each definition. The definition with the larger
  and more consistent reaction is the real liquidity level.

## A-03 · Sweep penetration and reclaim

* **Concept.** How far past a level is a "sweep", and how fast must the reclaim be?
* **Definition.** Penetration threshold + bounded reclaim window + close-back rule.
* **Proposed implementation.** `pen ≥ 1 tick (0.25)`; reclaim = a close back
  beyond the level within 3 bars; penetrating bar must close back inside the
  range; wick ratio ≥ 0.50 required.
* **Parameters.** `pen_min_pts`, `pen_mode {ticks, atr}`, `pen_atr_mult`,
  `reclaim_window_bars`, `reclaim_buffer_pts`, `require_close_back`,
  `require_wick`.
* **False positives.** 1 tick is very permissive: ordinary noise around a level
  registers as a sweep, especially on thin overnight levels.
* **False negatives.** A large, violent sweep that takes 5 bars to reclaim is
  rejected by a 3-bar window even though it was a textbook liquidity grab.
* **Validation.** Grid over `pen_min_pts × reclaim_window_bars`; measure the
  conditional distribution of the next 20 minutes' move. Pick the region where
  the reversal edge is stable, not maximal.

## A-04 · Abnormal wick threshold

* **Concept.** "wick/range ≥ 50%" is given as a research default and explicitly
  not asserted as optimal.
* **Definition.** Ratio + absolute size + relative-range triple test.
* **Proposed implementation.** `ratio ≥ 0.50 ∧ wick_pts ≥ 5.0 ∧ range ≥ 0.8·ATR14`.
* **Parameters.** `wick_ratio_min`, `wick_min_pts`, `wick_min_range_atr`,
  `wick_body_ratio_min` (optional secondary test).
* **False positives.** Small-range bars trivially clear 50%; without the
  absolute filters, quiet chop generates constant "abnormal wicks".
* **False negatives.** A genuine 40-point sweep bar that also travelled far in
  the body direction can score below 0.50 and be missed.
* **Validation.** Sweep `wick_ratio_min ∈ [0.35, 0.75]` and
  `wick_min_pts ∈ [0, 15]`; score by precision of "wick preceded a ≥1R reversal".

## A-05 · Displacement score composition and threshold

* **Concept.** "Fast, strong, one-directional" must become a number.
* **Definition.** Weighted 8-component score, 0–100, threshold-classified.
* **Proposed implementation.** See `TRADING_MODEL.md` §6. `QUALIFIED ≥ 55`.
* **Parameters.** All eight weights, `disp_atr_target`, `disp_min`,
  `disp_min_bars`, `disp_max_bars`, `atr_len`.
* **False positives.** ATR is depressed pre-08:30, so the first post-news bar
  scores near-maximum almost automatically — the score may say STRONG on every
  news day regardless of quality.
* **False negatives.** A steady three-bar grind that breaks structure and leaves
  an FVG can score below 55 because no single bar is dramatic.
* **Validation.** Two-stage: (a) label ~100 legs by hand as displacement /
  not, measure ROC-AUC of the score; (b) refit weights by logistic regression on
  "leg produced a valid retest entry that reached 2R". Report both.

## A-06 · FVG minimum size and mitigation

* **Concept.** What counts as a *clean* imbalance, and when is it used up?
* **Definition.** Minimum absolute size + mitigation fraction + invalidation.
* **Proposed implementation.** `size ≥ 3.0 pts`; `FULLY_MITIGATED` at 100% fill;
  `INVALIDATED` on a close beyond the far edge.
* **Parameters.** `fvg_min_pts`, `fvg_min_atr_mult` (alternative sizing mode),
  `mitig_full`, `fvg_require_displacement`, `fvg_max_age_bars`.
* **False positives.** 3 points on 1m NQ is common; many FVGs will be
  structurally meaningless.
* **False negatives.** Requiring 50% mitigation to disqualify a zone will
  reject good zones that got a deep-but-valid tap.
* **Validation.** Measure reaction size on first touch, bucketed by FVG size and
  by prior fill %. Set the minimum where the reaction distribution separates.

## A-07 · Order block zone boundaries

* **Concept.** Body only, body-to-wick, or full range?
* **Definition.** Selectable zone construction.
* **Proposed implementation.** `body_to_wick` default (bullish OB = `[L, max(O,C)]`).
* **Parameters.** `ob_zone_mode`, `ob_lookback_bars`, `ob_require_bos`,
  `ob_require_fvg`, `ob_max_mitigations`.
* **False positives.** Full-range zones are wide → almost any pullback "touches
  the OB" → the factor stops discriminating.
* **False negatives.** Body-only zones are narrow → price reverses one tick
  above the zone and no entry is ever registered.
* **Validation.** Compare fill rate and post-touch MFE across the three modes on
  the same setups; prefer the mode with the best fill-rate × edge product.

## A-08 · Rejection block construction

* **Concept.** A "zone created by a significant wick" has no standard geometry.
* **Definition.** `[extreme, body_boundary]` of the rejecting candle, gated by
  wick % and penetration.
* **Parameters.** `rb_wick_min`, `rb_pen_min`, `rb_require_reclaim`,
  `rb_zone_mode {extreme_to_body, extreme_to_mid}`.
* **False positives.** Every sweep bar with a long wick produces a rejection
  block, so the factor correlates ~1.0 with the sweep factor — double counting
  inside the confluence score.
* **False negatives.** Multi-bar rejections (two bars share the work) produce no
  single qualifying candle.
* **Validation.** Compute the correlation matrix of all 12 confluence factors on
  archived setups. Any pair with |ρ| > 0.8 must be merged or re-weighted — this
  is the main defence against a confluence score that is really counting one
  thing six times.

## A-09 · Structure: pivots, BOS vs MSS, break confirmation

* **Concept.** "Structure" and "market structure break" are undefined in the spec.
* **Definition.** Fractal pivots with `left/right` confirmation, close-based
  breaks, BOS (with trend) vs MSS (against trend) distinguished.
* **Parameters.** `pivot_left`, `pivot_right`, `structure_break_on {close, wick}`,
  `structure_timeout_bars`, `trend_pivot_count`.
* **False positives.** `right = 1` gives fast but noisy pivots; on 1m NQ nearly
  every bar becomes a pivot in chop.
* **False negatives.** `right = 5` delays confirmation by 5 minutes — after a
  news displacement the entry is already gone.
* **Validation.** No-repaint test (mandatory pass), then sweep `right ∈ [2,5]`
  measuring detection lag vs false-break rate.

## A-10 · CISD reference leg  ⚠ highest ambiguity

* **Concept.** "the opening price of the relevant bearish reference candle or
  delivery leg" — *relevant* is undefined and this is the single most
  interpretation-dependent rule in the model.
* **Definition.** Three candidate reference prices, one selectable.
* **Proposed implementation.** Default `leg_first_open` = open of the first
  candle of the last unbroken bearish run before the sweep low.
* **Parameters.** `cisd_ref_mode {leg_first_open, leg_last_open,
  swept_candle_open}`, `cisd_leg_max_bars`, `cisd_min_body_atr`,
  `cisd_max_bars_after_sweep`, `cisd_require_close`.
* **False positives.** `leg_last_open` triggers on almost any green bar after a
  sweep — CISD becomes meaningless and fires on every pullback.
* **False negatives.** `leg_first_open` on a long 10-bar decline sits far above
  price; CISD may never confirm inside the 15-bar window even on a genuine
  reversal, killing otherwise valid setups.
* **Validation.** Implement all three, run in parallel over the same archive,
  and record for each: trigger frequency, median bars-to-trigger, and forward
  MFE/MAE. Also collect your `HUMAN VALIDATION` tags on 50 sweeps to see which
  one matches your discretionary read. **Please answer B3 directly if you have a
  preference — this is where an unvalidated guess would do the most damage.**

## A-11 · Equal highs / equal lows tolerance

* **Concept.** "Equal" is never exactly equal.
* **Proposed implementation.** `|Δ| ≤ 2.0 pts`, min separation 3 bars, min 2
  members, level = mean of members.
* **Parameters.** `eq_tol_pts`, `eq_tol_atr_mult`, `eq_min_sep_bars`,
  `eq_min_count`, `eq_lookback_bars`.
* **False positives.** Wide tolerance turns any consolidation into "equal highs".
* **False negatives.** Tight tolerance misses the 3-point-apart double top that
  every human would call equal highs.
* **Validation.** Compare sweep-reaction magnitude for pools with 2 vs 3+
  members and across tolerance settings; strength should be monotone in member
  count if the concept is real.

## A-12 · Daily bias weights

* **Concept.** Nine heterogeneous inputs must produce one number in [−100, +100].
* **Proposed implementation.** Weighted sum, `TRADING_MODEL.md` §12.
* **Parameters.** All nine weights, `bias_thresh`, `adr_len`, `htf_tf`.
* **False positives.** Components are correlated (price vs daily open, price vs
  midnight open, and premium/discount all measure roughly "where is price"), so
  the score saturates at ±80 on trending days and overstates conviction.
* **False negatives.** Genuinely bullish days that open at the top of the prior
  range score neutral because component 3 penalises them.
* **Validation.** Regress next-session close-vs-open direction on `biasScore`
  over ≥300 sessions. If the relationship is not monotone, the weights are
  wrong. Publish the measured hit rate per bias decile on the dashboard instead
  of the raw score alone.

## A-13 · "Major" obstruction on the target path

* **Concept.** "INVALIDATE if major obstruction exists" — *major* undefined.
* **Proposed implementation.** Severity = size × proximity × strength summed
  over obstructions, threshold 1.0; obstructions before TP1 weighted 2×.
* **Parameters.** `path_fvg_min`, `path_ob_min_quality`, `path_pivot_touches`,
  `path_clear_max`, `path_tp1_weight`, `path_scan_mode {tp1_only, tp2}`.
* **False positives.** On NQ there is nearly always *something* between entry
  and 4R; an aggressive threshold rejects almost every trade.
* **False negatives.** A permissive threshold ignores the untouched PDL sitting
  0.5R before TP1, which is exactly the obstruction that matters.
* **Validation.** Measure TP1 and TP2 hit rates conditioned on severity buckets.
  Set the threshold where the TP2 hit rate degrades materially.

## A-14 · Premium/discount reference range

* **Concept.** Spec says use the ORB; standard practice uses the current dealing
  range (sweep extreme → displacement extreme).
* **Proposed implementation.** ORB equilibrium is the default (per spec);
  `pd_reference = dealing_range` available as an alternative.
* **Parameters.** `pd_reference`, `eq_band`.
* **False positives.** After a strong trend day the ORB is far away and
  everything reads "premium", making the factor a constant.
* **False negatives.** A perfect discount entry inside the dealing range scores
  0 because it is above the ORB midpoint.
* **Validation.** Compare factor-9 predictive value under both references.

## A-15 · Stop rule conflict  ⚠ blocking

* **Concept.** `stop_min = 10`, `stop_max = 30`, `stop ≤ R_orb/3` are jointly
  unsatisfiable when `R_orb < 30`, yet `R_orb ≥ 10` is declared LIVE.
* **Options.** (a) raise the effective minimum ORB to 30; (b) drop `stop_min` to
  `min(10, R_orb/3)`; (c) drop the `R/3` rule for small ranges; (d) scale
  `stop_min` with ATR.
* **Parameters.** `stop_min`, `stop_max`, `stop_orb_fraction`,
  `stop_conflict_policy ∈ {reject, relax_min, relax_fraction}`.
* **False positives.** Relaxing the minimum permits 4-point stops that sit
  inside normal 1m noise and get stopped constantly.
* **False negatives.** Rejecting means every ORB in 10–30 points is a no-trade
  day, discarding a large share of sessions.
* **Validation.** Distribution of `R_orb` over 12 months tells you exactly how
  many sessions each option costs. I will produce that histogram first, before
  you decide.

## A-16 · News filter and catalyst window

* **Concept.** "8:30 catalyst", "restricted news" undefined.
* **Proposed implementation.** A calendar of high-impact US releases; catalyst
  window `[08:29, 08:35]`; a configurable blacklist that forces NO TRADE.
* **Parameters.** `news_source`, `news_window_min`, `news_impact_min`,
  `news_blacklist[]`, `news_blackout_before_min`, `news_blackout_after_min`.
* **False positives.** Treating every 08:30 print as a catalyst mislabels
  ordinary days as News Reversal setups.
* **False negatives.** A single-source calendar misses revisions, unscheduled
  Fed speakers, and geopolitical headlines.
* **Validation.** Compare realised 08:30–08:45 range on flagged vs unflagged
  days; a real catalyst filter should show a clearly higher range distribution.

## A-17 · Volume confirmation

* **Concept.** Listed as a confirmation trigger, but no definition, and it is
  *not* order flow.
* **Proposed implementation.** `V ≥ 1.5 · SMA(V,20)`, marked explicitly as
  "activity, not order flow", weight held low.
* **Parameters.** `vol_mult`, `vol_len`, `vol_enabled` (default true but
  informational only).
* **False positives.** Volume always spikes at 08:30 and 09:30 regardless of
  direction — the flag will be on for nearly every setup in the window.
* **False negatives.** Genuine absorption shows *high volume with no progress*,
  which this test cannot distinguish from momentum.
* **Validation.** Measure its standalone information value; if ~0 (likely),
  demote it to display-only and remove it from confirmation triggers.

## A-18 · Setup-type classification overlap

* **Concept.** A session can satisfy Continuation and News Reversal at once.
* **Proposed implementation.** News Reversal wins when a catalyst is present and
  displacement is counter to the sweep; otherwise Continuation; otherwise
  `UNCLASSIFIED` (no trade by default).
* **Parameters.** `setup_priority`, `allow_unclassified`.
* **False positives.** Mislabelling changes which analytics bucket a trade lands
  in, silently corrupting per-setup statistics.
* **False negatives.** `UNCLASSIFIED → no trade` may reject good sequences that
  simply do not fit either template.
* **Validation.** Human tagging of setup type on the archive; measure
  agreement rate with the classifier.

## A-19 · Screenshot capture mechanism  ⚠ blocking

* **Concept.** "Automatic screenshot" of a TradingView chart has no supported
  API, and automating a logged-in TradingView session may conflict with their
  terms of service. I am not going to build something that quietly puts your
  account at risk.
* **Options.** (a) **Server-rendered charts** from our own bar data with our own
  annotation layer — fully deterministic, reproducible, no ToS exposure, but not
  pixel-identical to your TradingView view. (b) **Headless browser capture** of a
  TradingView chart — visually identical, but automation-dependent, fragile, and
  subject to TradingView's terms and your plan's limits. (c) Both: (a) as the
  archival record, (b) as an optional convenience layer.
* **Proposed implementation.** (a) as the default and the archival source of
  truth; (c) available if you confirm you accept the ToS considerations.
* **Parameters.** `screenshot_backend`, `screenshot_dpi`, `annotation_set`,
  `capture_stages[]`, `retention_days`.
* **False positives / negatives.** Not statistical — the risk here is an archive
  that cannot be regenerated later (headless capture breaks on any TradingView
  UI change) versus one that does not match what you actually saw.
* **Validation.** Render 10 archived setups both ways; confirm the server-side
  render contains every element from the SCREENSHOT REQUIREMENTS list.

## A-20 · Live fill assumptions

* **Concept.** "Entry" in a backtest is not an entry in the market.
* **Proposed implementation.** Limit at the zone with `require_touch_plus_ticks`
  (default 1 tick through the level to assume a fill); stops assumed to fill at
  stop price + `slippage_stop` (default 1.0 pt); targets require a trade
  *through* the level; no fills on the bar that created the signal.
* **Parameters.** `fill_model`, `entry_touch_ticks`, `slippage_entry`,
  `slippage_stop`, `slippage_target`, `commission_rt`, `allow_same_bar_fill`.
* **False positives.** Optimistic fills inflate every metric; assuming a limit
  fill on a 1-tick touch is the single most common backtest lie.
* **False negatives.** Overly harsh assumptions reject real edge.
* **Validation.** Once shadow mode runs, compare shadow-assumed fills against
  observed market prints at those timestamps and recalibrate.

## A-21 · Contract roll and continuous series

* **Concept.** Quarterly NQ rolls shift price by tens of points; PDH/PDL across
  a roll are not comparable.
* **Proposed implementation.** Front-month unadjusted for intraday levels
  (levels must be the prices that actually traded), with roll dates recorded and
  the roll day flagged so PDH/PDL/ONH/ONL are marked `CROSS_CONTRACT` and, by
  default, excluded from liquidity pools that day.
* **Parameters.** `roll_mode`, `roll_calendar`, `exclude_roll_day_levels`.
* **False positives.** Back-adjusted history creates levels that never existed.
* **False negatives.** Excluding roll days loses ~4 sessions/year (acceptable).
* **Validation.** Assert that every stored level price is within the traded
  range of the contract it is attributed to.

## A-22 · Half-days, holidays, thin sessions

* **Concept.** "Holiday/thin session if enabled" is undefined.
* **Proposed implementation.** CME holiday calendar; early-close days flagged;
  optional `thin_session` filter using a percentile of overnight volume.
* **Parameters.** `holiday_calendar`, `skip_half_days`, `thin_vol_pctile`,
  `skip_thin_sessions`.
* **False positives.** Volume percentiles drift seasonally; August is not a
  holiday.
* **False negatives.** Some of the cleanest sweeps happen on thin days.
* **Validation.** Compare expectancy on flagged vs unflagged sessions before
  enabling the filter by default.

## A-23 · ORB size bounds as absolute points

* **Concept.** 10 and 100 points are fixed constants applied across all
  volatility regimes.
* **Proposed implementation.** Keep 10/100 as the default (per spec), and add
  `orb_mode = atr` computing `orb_min = 0.5·ATR(14,daily)/6.5h·0.25h` style
  normalisation as an alternative for research comparison.
* **Parameters.** `orb_min`, `orb_max`, `orb_mode {absolute, atr}`,
  `orb_atr_min_mult`, `orb_atr_max_mult`.
* **False positives.** In high-vol regimes a 90-point ORB is "LIVE" but is
  actually a normal-sized range for that regime.
* **False negatives.** In low-vol regimes almost every day is STAND DOWN.
* **Validation.** Plot ORB size distribution by year/quarter; measure expectancy
  by ORB decile rather than by absolute bucket.

## A-24 · "Both sides swept" scope  ⚠ blocking

* **Concept.** Does a sweep before 08:30 (between 08:15 and 08:30) count?
* **Proposed implementation.** Only sweeps at or after 08:30 count toward the
  both-sides rule; pre-08:30 activity is recorded but not disqualifying.
* **Parameters.** `both_swept_window_start`, `both_swept_action {no_trade,
  warn_only}`, `both_swept_requires_reclaim`.
* **False positives.** Counting the 08:15–08:30 drift disqualifies many days
  that then produce a clean 08:30 reaction.
* **False negatives.** Ignoring it permits trading a range that has already been
  worked on both sides.
* **Validation.** Count sessions disqualified under each rule and compare the
  outcome distribution of the trades each rule would have allowed.

## A-25 · Quality score weights

* **Concept.** Nine components, no stated weights.
* **Proposed implementation.** The declared prior weights in
  `TRADING_MODEL.md` §24, summing to 100.
* **Parameters.** All nine weights, four band boundaries, `min_quality`.
* **False positives.** Weights correlated with confluence mean the score largely
  re-expresses the confluence count — an illusion of two independent checks.
* **False negatives.** A structurally excellent setup with only 3 confluences
  may be scored `DEVELOPING` and skipped.
* **Validation.** After ≥100 archived setups: check the rank correlation between
  quality band and realised R. If A+ does not outperform HIGH QUALITY, the
  weights are decorative and must be refit or the score retired.

---

## How these get resolved

1. You answer B1–B12 (or tell me to proceed with the stated defaults).
2. Every remaining entry stays in this register with status
   `PROPOSED → ACCEPTED → VALIDATED`, and the config carries the same status.
3. No entry reaches `VALIDATED` on opinion. It reaches it on the validation
   method described above, run on real data, with the sample size reported.
4. Any parameter still at `PROPOSED` is rendered on the dashboard's debug panel
   so you always know which numbers are unvalidated guesses.
