# PARAMETER CONFIGURATION TABLE

Deliverable **(9)**. Every threshold in the system appears here. Nothing is
hard-coded in an engine.

**Status column:** `SPEC` = value stated in your specification ·
`PROPOSED` = my starting point, unvalidated · `DERIVED` = mechanical consequence
of another value.

Master file: [`../config/northstar.defaults.yaml`](../config/northstar.defaults.yaml)

---

## 1. Instrument

| Key | Default | Unit | Status | Notes |
|---|---|---|---|---|
| `instrument.primary` | `NQ` | — | SPEC | |
| `instrument.secondary` | `MNQ` | — | SPEC | |
| `instrument.NQ.point_value` | 20 | USD/pt | SPEC | CME contract spec |
| `instrument.NQ.tick_size` | 0.25 | pt | SPEC | |
| `instrument.NQ.commission_rt` | 4.50 | USD | PROPOSED | must be set to your real broker rate |
| `instrument.NQ.slippage_est` | 1.00 | pt | PROPOSED | per side |
| `instrument.MNQ.point_value` | 2 | USD/pt | SPEC | |
| `instrument.MNQ.commission_rt` | 1.20 | USD | PROPOSED | |
| `instrument.roll_mode` | `front_unadjusted` | — | PROPOSED | `[A-21]` |
| `instrument.exclude_roll_day_levels` | true | bool | PROPOSED | `[A-21]` |

## 2. Session and clock

| Key | Default | Status | Notes |
|---|---|---|---|
| `session.tz` | `America/New_York` | SPEC | DST-aware |
| `session.premarket_start` | `07:00` | PROPOSED | brief generation window |
| `session.orb_start` | `08:00` | SPEC | |
| `session.orb_end` | `08:15` | SPEC | lock time |
| `session.exec_start` | `08:30` | SPEC | |
| `session.no_new_entries` | `11:00` | SPEC | |
| `session.flat_time` | `11:30` | SPEC | hard |
| `session.prev_day_mode` | `cme_day` | PROPOSED | `[A-02]` B4 |
| `session.overnight_start` | `18:00` | PROPOSED | `[A-02]` |
| `session.overnight_end` | `08:00` | PROPOSED | `[A-02]` |
| `session.daily_open_ref` | `1800` | PROPOSED | `[A-02]` |
| `session.holiday_calendar` | `CME_EQUITY` | PROPOSED | `[A-22]` |
| `session.skip_half_days` | true | PROPOSED | `[A-22]` |
| `session.skip_thin_sessions` | false | PROPOSED | `[A-22]` |
| `session.thin_vol_pctile` | 10 | PROPOSED | `[A-22]` |

## 3. Timeframes

| Key | Default | Status | Notes |
|---|---|---|---|
| `tf.exec` | `1m` | PROPOSED | `[A-01]` **B1** |
| `tf.context` | `15m` | PROPOSED | |
| `tf.bias` | `4H` | PROPOSED | |
| `tf.atr_len` | 14 | PROPOSED | on `tf.exec` |

## 4. ORB engine

| Key | Default | Unit | Status |
|---|---|---|---|
| `orb.min_points` | 10 | pt | SPEC |
| `orb.max_points` | 100 | pt | SPEC |
| `orb.mode` | `absolute` | — | PROPOSED `[A-23]` |
| `orb.atr_min_mult` | 0.6 | × | PROPOSED |
| `orb.atr_max_mult` | 3.0 | × | PROPOSED |
| `orb.lock_immutable` | true | bool | SPEC (never modify after 08:15) |

## 5. Liquidity engine

| Key | Default | Unit | Status |
|---|---|---|---|
| `liquidity.pen_mode` | `ticks` | — | PROPOSED `[A-03]` |
| `liquidity.pen_min_pts` | 0.25 | pt | PROPOSED |
| `liquidity.pen_atr_mult` | 0.10 | × | PROPOSED |
| `liquidity.reclaim_window_bars` | 3 | bars | PROPOSED |
| `liquidity.reclaim_buffer_pts` | 0.0 | pt | PROPOSED |
| `liquidity.require_close_back` | true | bool | PROPOSED |
| `liquidity.require_wick` | true | bool | PROPOSED |
| `liquidity.pool_kinds` | all 10 sources | — | SPEC |
| `liquidity.both_swept_action` | `no_trade` | — | SPEC (default) |
| `liquidity.both_swept_window_start` | `08:30` | — | PROPOSED `[A-24]` **B9** |
| `liquidity.eq_tol_pts` | 2.0 | pt | PROPOSED `[A-11]` |
| `liquidity.eq_min_sep_bars` | 3 | bars | PROPOSED |
| `liquidity.eq_min_count` | 2 | n | PROPOSED |

## 6. Wick engine

| Key | Default | Status |
|---|---|---|
| `wick.ratio_min` | 0.50 | SPEC (explicitly "research threshold") `[A-04]` |
| `wick.min_pts` | 5.0 | PROPOSED |
| `wick.min_range_atr` | 0.8 | PROPOSED |
| `wick.body_ratio_min` | 2.0 | PROPOSED (secondary test, off by default) |

## 7. Displacement engine

| Key | Default | Status |
|---|---|---|
| `displacement.min_score` | 55 | PROPOSED `[A-05]` |
| `displacement.strong_score` | 80 | PROPOSED |
| `displacement.min_bars` | 1 | PROPOSED |
| `displacement.max_bars` | 5 | PROPOSED |
| `displacement.atr_target` | 1.5 | PROPOSED |
| `displacement.w_atr` | 30 | PROPOSED |
| `displacement.w_body` | 20 | PROPOSED |
| `displacement.w_avg_body` | 15 | PROPOSED |
| `displacement.w_consistency` | 10 | PROPOSED |
| `displacement.w_close_loc` | 10 | PROPOSED |
| `displacement.w_opp_wick` | 5 | PROPOSED |
| `displacement.w_structure` | 5 | PROPOSED |
| `displacement.w_fvg` | 5 | PROPOSED |
| `displacement.sweep_to_disp_max_bars` | 10 | PROPOSED |

## 8. FVG engine

| Key | Default | Status |
|---|---|---|
| `fvg.min_pts` | 3.0 | PROPOSED `[A-06]` |
| `fvg.min_atr_mult` | 0.35 | PROPOSED (alt sizing mode) |
| `fvg.sizing_mode` | `points` | PROPOSED |
| `fvg.mitig_full` | 1.0 | PROPOSED |
| `fvg.require_displacement` | true | PROPOSED |
| `fvg.max_age_bars` | 60 | PROPOSED |
| `fvg.required_for_setup` | true | SPEC (`NO FVG when required`) |

## 9. Structure engine

| Key | Default | Status |
|---|---|---|
| `structure.pivot_left` | 3 | PROPOSED `[A-09]` |
| `structure.pivot_right` | 3 | PROPOSED |
| `structure.break_on` | `close` | PROPOSED |
| `structure.timeout_bars` | 15 | PROPOSED |
| `structure.trend_pivot_count` | 2 | PROPOSED |

## 10. Order block / rejection block

| Key | Default | Status |
|---|---|---|
| `ob.zone_mode` | `body_to_wick` | PROPOSED `[A-07]` |
| `ob.lookback_bars` | 5 | PROPOSED |
| `ob.require_bos` | false | PROPOSED |
| `ob.max_mitigations` | 1 | PROPOSED |
| `rb.wick_min` | 0.50 | PROPOSED `[A-08]` |
| `rb.pen_min_pts` | 0.25 | PROPOSED |
| `rb.require_reclaim` | true | PROPOSED |
| `rb.zone_mode` | `extreme_to_body` | PROPOSED |

## 11. CISD engine

| Key | Default | Status |
|---|---|---|
| `cisd.ref_mode` | `leg_first_open` | PROPOSED `[A-10]` **B3** |
| `cisd.leg_max_bars` | 10 | PROPOSED |
| `cisd.min_body_atr` | 0.5 | PROPOSED |
| `cisd.max_bars_after_sweep` | 15 | PROPOSED |
| `cisd.require_close` | true | SPEC (definition uses "closes above/below") |

## 12. Bias engine

| Key | Default | Status |
|---|---|---|
| `bias.threshold` | 25 | PROPOSED `[A-12]` |
| `bias.w_daily_open` | 15 | PROPOSED |
| `bias.w_midnight_open` | 10 | PROPOSED |
| `bias.w_pd_position` | 15 | PROPOSED |
| `bias.w_pd_close_loc` | 10 | PROPOSED |
| `bias.w_htf_structure` | 20 | PROPOSED |
| `bias.w_overnight` | 10 | PROPOSED |
| `bias.w_liquidity_objective` | 10 | PROPOSED |
| `bias.w_premium_discount` | 10 | PROPOSED |
| `bias.w_news` | 0 | PROPOSED (news gates, does not direct) |
| `bias.adr_len` | 14 | PROPOSED |

## 13. Premium/discount, key opens

| Key | Default | Status |
|---|---|---|
| `pd.reference` | `orb` | SPEC `[A-14]` |
| `pd.eq_band_pts` | 2.0 | PROPOSED |
| `key_open.proximity_pts` | 5.0 | SPEC |
| `key_open.set` | midnight, daily, 0830, 0930 | SPEC |

## 14. Retest engine

| Key | Default | Status |
|---|---|---|
| `retest.zone_priority` | fvg_mid, fvg_edge, ob, rb, orb_level | PROPOSED |
| `retest.zone_pad_pts` | 1.0 | PROPOSED |
| `retest.touch_tolerance_pts` | 0.5 | PROPOSED |
| `retest.mfe_max_R` | 1.0 | SPEC (>1R invalidates) |
| `retest.timeout_bars` | 20 | PROPOSED |
| `retest.fail_buffer_pts` | 1.0 | PROPOSED |
| `retest.entry_ref` | `zone_touch` | PROPOSED |
| `retest.entry_ttl_bars` | 5 | PROPOSED |

## 15. Confirmation / confluence / quality

| Key | Default | Status |
|---|---|---|
| `confirmation.min_triggers` | 1 | PROPOSED |
| `confirmation.vol_mult` | 1.5 | PROPOSED `[A-17]` |
| `confirmation.vol_enabled` | true (display only) | PROPOSED |
| `confluence.min_required` | 3 | SPEC |
| `confluence.max_display` | 12 | SPEC |
| `confluence.factor_weights` | all 1.0 | SPEC |
| `quality.min_required` | 60 | PROPOSED `[A-25]` |
| `quality.bands` | 39/59/74/89 | SPEC |
| `quality.w_*` (9 weights) | 12/12/15/10/12/10/12/9/8 | PROPOSED |

## 16. Target path

| Key | Default | Status |
|---|---|---|
| `path.scan_mode` | `tp2` | PROPOSED `[A-13]` |
| `path.fvg_min_pts` | 3.0 | PROPOSED |
| `path.ob_min_quality` | 0.5 | PROPOSED |
| `path.pivot_touches` | 2 | PROPOSED |
| `path.clear_max_severity` | 1.0 | PROPOSED |
| `path.tp1_weight` | 2.0 | PROPOSED |
| `path.obstructed_action` | `invalidate` | SPEC (default) |

## 17. Risk, sizing, targets

| Key | Default | Status |
|---|---|---|
| `risk.stop_buffer_pts` | 2.0 | PROPOSED |
| `risk.stop_min_pts` | 10 | SPEC |
| `risk.stop_max_pts` | 30 | SPEC |
| `risk.stop_orb_fraction` | 0.3333 | SPEC |
| `risk.conflict_policy` | `reject` | PROPOSED `[A-15]` **B2** |
| `risk.risk_per_trade_usd` | 500 | PROPOSED — **you must set this** |
| `risk.daily_loss_limit_usd` | 1000 | PROPOSED — **you must set this** |
| `targets.tp1_R` | 2.0 | SPEC |
| `targets.tp1_fraction` | 0.50 | SPEC |
| `targets.be_after_tp1` | true | SPEC |
| `targets.tp2_hierarchy` | opposite_orb → 4R → measured_move | SPEC |
| `targets.tp2_min_R` | 3.0 | PROPOSED |
| `runner.trail_tf` | `1m` | SPEC |
| `runner.trail_pivot_left/right` | 2 / 2 | PROPOSED |

## 18. Daily governor

| Key | Default | Status |
|---|---|---|
| `daily.max_trades` | 2 | SPEC |
| `daily.stop_after_first_win` | true | SPEC |
| `daily.second_trade_size_mult` | 0.5 | SPEC |
| `daily.stop_after_two_losses` | true | SPEC |
| `daily.second_trade_after` | `loss_only` | PROPOSED **B11** |

## 19. News

| Key | Default | Status |
|---|---|---|
| `news.source` | *(unset)* | **B8** |
| `news.window_min` | 5 | PROPOSED `[A-16]` |
| `news.impact_min` | `high` | PROPOSED |
| `news.blacklist` | [] | PROPOSED |
| `news.blackout_before_min` | 2 | PROPOSED |
| `news.blackout_after_min` | 0 | PROPOSED |

## 20. Order flow (gated)

| Key | Default | Status |
|---|---|---|
| `orderflow.provider` | `null` | SPEC-driven (unavailable by default) |
| `orderflow.absorb_vol_pctile` | 90 | PROPOSED |
| `orderflow.absorb_max_move_atr` | 0.25 | PROPOSED |
| `orderflow.never_synthesize_from_volume` | true | SPEC — **immutable, not overridable** |

## 21. Alerts, screenshots, ops

| Key | Default | Status |
|---|---|---|
| `alerts.dedupe_window_sec` | 900 | PROPOSED |
| `alerts.min_state_change_only` | true | SPEC (no spam) |
| `alerts.morning_brief_time` | `07:45` | PROPOSED |
| `alerts.rate_limit_per_min` | 10 | PROPOSED |
| `screenshot.backend` | `server_render` | PROPOSED `[A-19]` **B7** |
| `screenshot.stages` | A–Q (17 stages) | SPEC |
| `screenshot.retention_days` | 1825 | PROPOSED |
| `mode` | `shadow` | SPEC (live not implemented) |
| `execution.provider` | `null` | SPEC |
| `param_version` | `0.1.0-proposed` | DERIVED |

---

## Parameter governance

1. Any engine reading a literal number that is not in this table is a bug.
2. `param_version` bumps on any default change; the changelog records why.
3. Each session stores the sha256 of its fully-resolved config.
4. Backtest results are invalid to compare across different `param_version`
   values unless the sweep explicitly varies the parameter under study.
5. The dashboard's debug panel lists every parameter still marked `PROPOSED`.
