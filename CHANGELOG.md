# CHANGELOG

All notable changes to NORTHSTAR CAPITAL™ / NORTHSTAR MOTION MODEL™.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: semver on `param_version` for parameters, separate semver for code.

---

## [0.2.0] — 2026-09-05 — TRADINGVIEW INDICATOR

Scope narrowed to a TradingView indicator (decision D-001). Backend, Telegram,
screenshot, database and archive designs remain as forward architecture and are
not built.

### Added
- `tradingview/northstar_motion_model.pine` — Pine v6 overlay indicator,
  ~1,500 lines, implementing the model end to end:
  session and trading-day engine (Pacific day boundary, DST-aware), level
  engine, opening-range engine with hard lock, liquidity pools with sweep /
  reclaim / fail states, abnormal wick, eight-component displacement score,
  FVG registry with mitigation tracking, non-repainting pivots with BOS/MSS,
  order blocks, rejection blocks, six-component daily bias, premium/discount,
  key-open alignment, retest engine with the >1R chase invalidation, target-path
  obstruction scan, confluence (X/9), nine-component quality score, display-only
  risk plan, the 23-state machine with reason-coded invalidations, the
  institutional dashboard, the Northstar checklist panel, and text/JSON alerts.
- `tradingview/README.md` — install, session table, dashboard guide, input
  reference, non-repainting guarantees, stated limitations.
- `docs/DECISIONS.md` — decisions log, authoritative over the other documents.

### Decided
- **D-002** execution timeframe is 1 minute.
- **D-003** trading day is 00:00–23:59 Pacific (03:00–02:59 ET) for previous-day
  high, low, open and close.
- **D-004** ORB is 05:00–05:15 PT, which is the 08:00–08:15 ET of the original
  specification restated in Pacific time.
- **D-005** the stop-rule contradiction is deferred; the min/max/ORB-third stop
  filters ship default-off so the impossible state cannot arise.
- **D-006** CISD removed. Confluence drops from 12 factors to 9 available ones;
  delta divergence and absorption are reported as N/A because TradingView has no
  true order-flow data, and are excluded from the denominator rather than
  silently scored zero.

### Notes
- No `request.security()` call exists in the script, so no higher-timeframe
  value can leak backwards. Higher-timeframe structure is therefore excluded
  from the bias score.
- All engine thresholds are inputs and remain unvalidated starting points.
- Risk figures are display only. Nothing here can place an order.

---

## [0.1.0-proposed] — 2026-09-05 — PHASE 1: ARCHITECTURE

**Status: awaiting approval. No implementation code exists.**

### Added — documentation set
- `README.md` — project overview, status, stated limitations.
- `docs/ARCHITECTURE.md` — layered architecture, runtime topology, data-flow
  diagram, per-bar evaluation order, module dependency map, dual-runtime parity
  contract, anti-lookahead enforcement, order-flow abstraction, execution
  interface, project layout.
- `docs/STATE_MACHINE.md` — state diagram, 42-row transition table, guard
  reference, eight runtime invariants.
- `docs/TRADING_MODEL.md` — exact algorithmic definitions for all 26 concepts.
- `docs/AMBIGUITY_REGISTER.md` — 12 blocking questions and 25 ambiguity entries,
  each with definition, proposed maths, parameters, expected false positives,
  expected false negatives and a validation method.
- `docs/PARAMETERS.md` — complete parameter table, every value marked
  `SPEC` / `PROPOSED` / `DERIVED`.
- `docs/DATABASE_SCHEMA.md` — 25-table schema, ER diagram, indexes, retention.
- `docs/EVENT_SCHEMA.md` — identifiers, envelope, 17 milestones, per-type
  payloads, webhook transport and validation pipeline, ordering guarantees.
- `docs/TELEGRAM_SETUP.md` — architecture, alert policy, message templates,
  11 read-only commands, failure behaviour.
- `docs/SCREENSHOT_SYSTEM.md` — three capture options with an explicit
  recommendation, 17 stages, required content, annotation rules, naming and
  storage.
- `docs/RISK_RULES.md` — stop rules, sizing with worked examples, targets,
  daily governor, 20 no-trade filters, eight risk invariants.
- `docs/SETUP_ARCHIVE.md` — record contents, replay, human-vs-algorithm review,
  query surface, export, integrity.
- `docs/TESTING.md` — 11 test layers, per-engine requirements, the mandatory
  no-lookahead suite, phase gate.
- `docs/BACKTESTING.md` — data requirements, pessimistic fill model, metrics,
  10 analysis dimensions, seven-step validation protocol, reproducibility.
- `docs/SHADOW_MODE.md` — what runs, what is logged, nine go-live criteria.
- `docs/SECURITY.md` — secrets handling, layered webhook security, execution
  safety, incident response.
- `docs/DEPLOYMENT.md` — environments, topology, release process, daily
  operational timeline, monitoring, backup, phase sequence.
- `config/northstar.defaults.yaml` — proposed parameter surface, no secrets.
- `.env.example`, `.gitignore`, directory scaffold.

### Flagged for your decision
- **B1** execution timeframe (1m assumed).
- **B2** the stop-rule contradiction: `stop ≥ 10 pts` and `stop ≤ ORB/3` are
  jointly unsatisfiable for any ORB below 30 points, which makes a declared-LIVE
  range untradeable as specified.
- **B3** CISD reference-leg definition (three candidates).
- **B4** session boundaries for prior day / overnight / daily open.
- **B5** market-data vendor. **B6** contract-roll policy.
- **B7** screenshot mechanism (server-rendered recommended).
- **B8** news data source and restricted-news list.
- **B9** whether pre-08:30 ORB sweeps count toward "both sides swept".
- **B10** whether an order-flow feed will ever be attached.
- **B11** whether an unfilled invalidation consumes a trade slot.
- **B12** absolute vs ATR-normalised ORB size bounds.

### Explicitly not done
- No engine implementation, no Pine script, no backend service, no database
  migrations, no Telegram bot — per the specification's first-response
  requirement.
- No broker integration and no credentials anywhere in the repository.

---

## Planned

- Tune the indicator's thresholds against live sessions and record which ones
  hold up; `docs/AMBIGUITY_REGISTER.md` lists what each is likely to get wrong.
- Optional next steps, only if wanted: webhook backend consuming the JSON alert
  envelope, then Telegram, screenshots, archive and backtesting per the existing
  architecture documents.
