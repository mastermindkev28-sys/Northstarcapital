# CHANGELOG

All notable changes to NORTHSTAR CAPITAL™ / NORTHSTAR MOTION MODEL™.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: semver on `param_version` for parameters, separate semver for code.

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

- `[0.2.0]` Phases 2–4 — session, ORB and level engines with full test suites,
  after the blocking questions are answered.
- `[0.3.0]` Phases 5–12 — structure engines.
- `[0.4.0]` Phases 13–20 — qualification, risk, state machine.
- `[0.5.0]` Phases 21–24 — dashboard, alerts, webhook, Telegram.
- `[0.6.0]` Phases 25–29 — screenshots, database, archive, analytics, backtest.
- `[0.7.0]` Phases 30–32 — shadow mode, human validation, execution interface.
