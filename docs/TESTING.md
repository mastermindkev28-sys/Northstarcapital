# TESTING PLAN

Deliverable **(12)**. Status: PROPOSED.

Rule for the whole project: **an engine is not done until its tests are done.**
`DO NOT SKIP VALIDATION` is enforced by the phase gate in §7.

---

## 1. Test layers

| Layer | Directory | Purpose | Runs |
|---|---|---|---|
| Unit | `tests/unit/` | one engine, synthetic bars, exact expected output | every commit |
| Property | `tests/property/` | invariants over generated inputs (Hypothesis) | every commit |
| Edge | `tests/edge/` | degenerate inputs: zero range, gaps, limit moves, single-bar sessions | every commit |
| Timezone | `tests/timezone/` | DST transitions, half days, holidays, roll days | every commit |
| No-lookahead | `tests/nolookahead/` | causality — the most important suite here | every commit |
| Golden | `tests/golden/` | recorded real sessions with frozen expected event streams | every commit |
| Historical | `tests/historical/` | multi-month replay, statistical sanity | nightly |
| Parity | `tests/parity/` | Pine output vs Python output | on Pine change + nightly |
| Failure | `tests/failure/` | feed loss, API errors, DB down, partial writes | every commit |
| Integration | `tests/integration/` | webhook → router → state → alert → DB, end to end | every commit |
| Layering | `tests/test_layering.py` | no upward module dependencies | every commit |

---

## 2. Unit tests — per engine

Each of the 28 engines needs, at minimum:

* **Positive case** — the textbook example, hand-computed, asserted exactly.
* **Negative case** — one condition removed, assert not detected *and* assert
  the correct rejection code.
* **Boundary cases** — exactly at each threshold, one tick either side. For a
  10-point minimum stop: 9.75 rejects, 10.00 accepts, 10.25 accepts.
* **Configurability** — change the parameter, assert the outcome changes
  accordingly. This catches hard-coded constants.
* **Explainability** — assert `reasons` is non-empty and machine-readable.
* **Purity** — same inputs twice ⇒ identical outputs; no engine touches I/O or
  the wall clock.

Worked examples that must be encoded as fixtures:

| Engine | Fixture |
|---|---|
| orbEngine | 15 one-minute bars → known high/low/range; assert immutability by attempting a post-lock mutation and expecting a raise |
| liquidityEngine | sweep with reclaim / sweep without reclaim / clean break; assert `RECLAIMED` vs `FAILED` |
| wickEngine | wick ratios 0.49 / 0.50 / 0.51 against `ratio_min = 0.50` |
| displacementEngine | a hand-scored leg; assert each component contribution, then the total |
| fvgEngine | 3-bar bullish gap, then partial fill 40%, then full fill, then close-through invalidation |
| structureEngine | pivot published exactly `pivot_right` bars late, never earlier |
| cisdEngine | all three `ref_mode` values on the same bars produce three different reference prices |
| riskEngine | the B2 conflict case: `R_orb = 20` must return `STOP_INVALID` under `reject` policy |
| positionSizingEngine | the worked example from `RISK_RULES.md` §2, both NQ and MNQ |
| confluenceEngine | order-flow unavailable ⇒ factors 10/11 are `UNAVAILABLE`, not `false`, and total is 10-max not 12-max in meaning |

---

## 3. No-lookahead suite (mandatory, blocking)

The definitive test:

```
for t in every bar of a recorded session:
    engine_output_prefix = run_engine(bars[0..t])
    assert engine_output_prefix == full_day_output[0..len(prefix)]
```

If any event's type, bar_time or price changes when more data becomes
available, the test fails. This mechanically detects repainting.

Additional causality tests:

* `CausalView.bar(t+1)` raises `LookaheadError`.
* Loading a news record with `available_at > view.now` raises.
* `OPEN_0830` is `Unavailable` at 08:29 and `Measured` at 08:30.
* Pivots are never published before `bar_index + pivot_right`.
* ORB values are unchanged by any bar after 08:15.
* Backtest and live paths produce identical events for the same tape.

**Any failure in this suite blocks release. No exceptions, no skips.**

---

## 4. Timezone and calendar suite

| Case | Expectation |
|---|---|
| 2026-03-08 (US DST start) | session windows still 08:00/08:15/08:30 ET; no duplicated or missing hour |
| 2026-11-01 (US DST end) | same |
| Days when US and EU DST differ | ET is authoritative; no UTC-offset assumptions anywhere |
| Half day (e.g. day after Thanksgiving) | flagged; `skip_half_days` honoured |
| CME holiday | no session created |
| Quarterly roll day | levels flagged `CROSS_CONTRACT`; pools excluded per config |
| Bar timestamp convention | open-time vs close-time convention asserted explicitly for the data source |

The last one deserves emphasis: an off-by-one-bar timestamp convention error
silently shifts the ORB by a minute and is nearly invisible in review. It gets
an explicit test against real vendor data.

---

## 5. Failure tests

| Scenario | Required behaviour |
|---|---|
| Missing bars mid-session | `DATA_INTEGRITY` rejection; setup invalidated; never interpolated |
| Duplicate webhook delivery | second is ignored via `event_id`; no second Telegram message |
| Out-of-order `sequence` | quarantined + `OPS_ALERT`; state unchanged |
| Telegram 429 / 5xx | backoff, queue preserved, no duplicate on retry |
| DB unavailable at event time | event buffered to disk WAL, replayed on recovery, order preserved |
| Screenshot renderer crash | trading unaffected; row marked `FAILED`; `OPS_ALERT` |
| Process restart mid-session | state rebuilt by folding persisted events; no re-alerting |
| Clock skew > 2 s | `OPS_ALERT`; events with implausible timestamps quarantined |
| Config hash mismatch mid-session | session halted rather than continuing under two configs |

---

## 6. Parity tests (Pine ↔ Python)

For each golden session: replay the recorded Pine alert payloads and the Python
engine output; diff field by field.

Tolerances: booleans and state names exact; prices ±0.01; 0–100 scores ±1;
timestamps exact to the bar. Any mismatch is a release blocker and gets an
`OPS_ALERT` if detected in production.

---

## 7. Phase gate (how "DO NOT SKIP VALIDATION" is enforced)

A phase is complete only when **all** of these are true:

1. Unit + property + edge tests pass, with ≥ 90% line coverage on that engine.
2. The no-lookahead suite passes for that engine.
3. At least one golden fixture exercises it on real bars.
4. Every parameter it uses appears in `PARAMETERS.md` and the YAML.
5. Every ambiguity it touches has a register entry with a validation method.
6. Its rejection codes are in the canonical list and produce human sentences.
7. `CHANGELOG.md` records what was added and which parameters are still
   `PROPOSED`.

CI enforces 1–4 mechanically; 5–7 are checked in review.

---

## 8. Tooling

`pytest` · `hypothesis` (property) · `freezegun` (clock) · `pytest-cov` ·
`ruff` + `mypy --strict` on `strategy/` and `risk/` · `pytest-postgresql` for
integration · fixtures stored as compressed CSV under `tests/fixtures/`.

CI runs on every push: lint → type → unit → property → edge → timezone →
nolookahead → golden → integration → failure. Nightly adds historical + parity.
