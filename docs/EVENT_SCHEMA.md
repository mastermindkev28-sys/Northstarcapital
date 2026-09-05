# EVENT SCHEMA

Deliverable **(8)**. Status: PROPOSED.

Events are the system's only inter-component currency. An engine result becomes
a fact only when it is emitted as an event, and an event is immutable.

---

## 1. Identifiers

```
session_id  = {YYYY-MM-DD}-{SYMBOL}                      2026-09-05-NQ
setup_id    = {session_id}-SETUP{NNN}                    2026-09-05-NQ-SETUP001
event_id    = {setup_id|session_id}-{SEQ:04d}-{TYPE}     2026-09-05-NQ-SETUP001-0007-DISPLACEMENT
signal_id   = {setup_id|session_id}-{ALERT_CLASS}        2026-09-05-NQ-SETUP001-LIQUIDITY
```

* `event_id` is unique and is the **idempotency key** for ingestion.
* `signal_id` is the **anti-spam key**: at most one alert per `signal_id`, ever.
  A repeated `signal_id` is recorded as `SUPPRESSED`, not sent.
* `SETUP{NNN}` increments per session and is **never reused**, even after
  invalidation.
* `sequence` is monotonic per session across all events.

---

## 2. Envelope

Every event, from any producer, has the same envelope:

```json
{
  "schema_version": "1.0",
  "event_id": "2026-09-05-NQ-SETUP001-0007-DISPLACEMENT",
  "signal_id": "2026-09-05-NQ-SETUP001-DISPLACEMENT",
  "session_id": "2026-09-05-NQ",
  "setup_id": "2026-09-05-NQ-SETUP001",
  "sequence": 7,
  "milestone": 6,
  "type": "DISPLACEMENT",
  "timestamp": "2026-09-05T12:31:15Z",
  "session_time_et": "08:31:15",
  "bar_time": "2026-09-05T12:31:00Z",
  "timeframe": "1m",
  "symbol": "NQ",
  "contract": "NQZ2026",
  "state": "DISPLACEMENT_DETECTED",
  "prev_state": "LIQUIDITY_SWEPT",
  "price": 23485.25,
  "direction": "LONG",
  "reason": "Displacement 84/100 after ORB low sweep; FVG 5.75 pts created.",
  "confluence": 4,
  "quality_score": 71,
  "next_condition": "RETEST INTO FVG / ORDER BLOCK",
  "mode": "shadow",
  "source": "python_engine",
  "config_hash": "3f9a…",
  "param_version": "0.1.0-proposed",
  "payload": { }
}
```

**Required in every event** (per your specification): `event_id`, `session_id`,
`setup_id`, `timestamp`, `symbol`, `state`, `price`, `direction`, `reason`,
`confluence`, `quality_score`. All eleven are non-null on setup-scoped events;
on session-scoped events (`PREMARKET_READY`, `ORB_LOCKED`, `DAY_DONE`)
`setup_id` is the literal string `"NONE"` and `direction` is `"NONE"` rather
than null, so the shape never varies.

---

## 3. Milestone catalogue

| # | `type` | Scope | Emitted when |
|---|---|---|---|
| 01 | `PREMARKET_READY` | session | levels + bias computed, ≤ 08:00 |
| 02 | `ORB_LOCKED` | session | 08:15:00, includes size + status |
| 03 | `LIQUIDITY_IDENTIFIED` | session | pools mapped after lock |
| 04 | `LIQUIDITY_SWEPT` | setup | qualified sweep → mints `setup_id` |
| 05 | `ABNORMAL_WICK` | setup | wick test passes on the sweep bar |
| 06 | `DISPLACEMENT` | setup | score ≥ threshold |
| 07 | `FVG_CREATED` | setup | valid gap inside the leg |
| 08 | `STRUCTURE_CONFIRMED` | setup | BOS/MSS published |
| 09 | `RETEST_DETECTED` | setup | zone touched |
| 10 | `CONFIRMATION` | setup | ≥1 trigger fired (CISD etc.) |
| 11 | `CONFLUENCE_MET` | setup | ≥ `min_confluence` |
| 12 | `RISK_VALIDATED` | setup | stop + size valid |
| 13 | `ENTRY_READY` | setup | full `TradePlan` persisted |
| 14 | `TP1` | setup | TP1 reached |
| 15 | `RUNNER` | setup | runner active, stop at BE |
| 16 | `EXIT` | setup | TP2 / trail / stop / time |
| 17 | `DAY_DONE` | session | terminal, with reason |
| — | `INVALIDATION` | setup | any invalidation, with reason code |
| — | `REJECTION` | either | a gate failed but the setup continues |
| — | `HEARTBEAT` | session | internal only, never alerted |
| — | `OPS_ALERT` | system | data gap, feed loss, parity break |

---

## 4. Type-specific payloads

Payloads are additive; consumers must ignore unknown keys.

**`ORB_LOCKED`**
```json
{"orb_high":23470.50,"orb_low":23427.25,"range":43.25,"status":"LIVE",
 "equilibrium":23448.875,"locked_at_et":"08:15:00"}
```

**`LIQUIDITY_SWEPT`**
```json
{"pool":{"kind":"ORB_LOW","level":23427.25,"side":"SELL_SIDE","strength":3},
 "penetration_pts":6.75,"wick_ratio":0.62,"reclaimed":true,"reclaim_bars":1,
 "close_back":true,"other_side_state":"IDENTIFIED","both_swept":false}
```

**`DISPLACEMENT`**
```json
{"score":84,"classification":"STRONG","direction":"LONG","bars":3,
 "leg_move":38.5,"atr_ratio":2.1,"body_ratio":0.81,"consistency":1.0,
 "close_location":0.94,"components":{"atr":30,"body":20,"avg_body":12,
 "consistency":10,"close_loc":10,"opp_wick":4,"structure":5,"fvg":5}}
```

**`FVG_CREATED`**
```json
{"fvg_id":"…","direction":"BULLISH","low":23452.00,"high":23457.75,
 "midpoint":23454.875,"size":5.75,"from_displacement":true}
```

**`RETEST_DETECTED`**
```json
{"zone_type":"FVG_MID","zone_low":23453.50,"zone_high":23456.25,
 "fvg_midpoint":23454.875,"order_block":true,"rejection_block":true,
 "cisd":true,"bars_since_displacement":7,"mfe_before_retest_r":0.6}
```

**`ENTRY_READY`** — the only event that carries a complete `TradePlan`
```json
{"setup_type":"NEWS_REVERSAL","direction":"LONG",
 "entry":23455.00,"stop":23425.25,"stop_distance":29.75,
 "tp1":23514.50,"tp2":23574.00,"tp2_source":"fixed_4R",
 "contracts":1,"risk_usd":595.00,"point_value":20,
 "confluence":7,"confluence_detail":{"liquidity_sweep":true,"displacement":true,
   "fvg":true,"order_block":true,"rejection_block":true,"cisd":true,
   "daily_bias":true,"premium_discount":false,"key_open":false,
   "delta_divergence":"UNAVAILABLE","absorption":"UNAVAILABLE",
   "clean_target_path":false},
 "quality":87,"quality_band":"HIGH_QUALITY","bias":"BULLISH","bias_score":72,
 "location":"DISCOUNT","path_status":"PATH_CLEAR","expires_after_bars":5}
```

**`INVALIDATION`**
```json
{"code":"MOVED_1R_BEFORE_RETEST","message":"Price travelled 1.4R from the
 displacement leg before any retest; setup invalidated.",
 "milestone_reached":8,"context":{"mfe_r":1.4,"limit_r":1.0}}
```

**`DAY_DONE`**
```json
{"reason":"MAX_TRADES","trades":2,"wins":0,"losses":2,
 "realized_r":-1.5,"realized_pnl_usd":-750.00,"setups_observed":3,
 "rejection_summary":{"NO_RETEST":1,"CONFLUENCE_BELOW_THRESHOLD":1}}
```

---

## 5. Transport: TradingView → backend

TradingView alert message body (Pine builds this string; only whitelisted
placeholders are interpolated):

```json
{
  "v": "1.0",
  "secret": "<WEBHOOK_SECRET>",
  "nonce": "{{timenow}}-{{ticker}}-0007",
  "event": { ...full envelope above... }
}
```

`POST /webhook/tradingview`

Validation pipeline, in order — a failure at any step returns `202 Accepted`
with the event dropped and logged, so a probing client learns nothing:

1. **Source IP** in the TradingView allowlist (verify the current list against
   TradingView's own documentation at deploy time; it is not hard-coded in
   application code but held in config).
2. **`secret`** compared with `hmac.compare_digest` against `WEBHOOK_SECRET`.
3. **Body size** ≤ 16 KB; JSON parse with a strict schema (pydantic).
4. **Freshness**: `|now − timestamp| ≤ 120 s`.
5. **Idempotency**: `event_id` unseen (Redis/DB set, 48 h TTL).
6. **Ordering**: `sequence` = last + 1, or the event is quarantined for review
   rather than applied out of order.
7. **Session sanity**: `session_id` matches today's ET session; symbol is
   tracked.

**Honest limitation:** TradingView alert bodies are visible to anyone who can
open the alert dialog in your account, so the shared secret is a *low-trust*
credential. It is defence-in-depth alongside the IP allowlist and TLS, and it is
rotated on a schedule. The webhook is only permitted to *record and notify* —
it can never place an order, and the endpoint has no code path that touches an
`ExecutionProvider`.

---

## 6. Ordering, replay and idempotency guarantees

| Guarantee | Mechanism |
|---|---|
| Exactly-once side effects | `event_id` unique index + `signal_id` unique index on alerts |
| No duplicate Telegram message | alert insert happens in the same transaction as the send-intent; the sender is a queue consumer keyed by `signal_id` |
| Deterministic replay | events are pure functions of `(bars, config)`; replaying a session reproduces identical `event_id`s |
| Out-of-order safety | `sequence` gap → quarantine + `OPS_ALERT`, never silent reordering |
| Crash recovery | on boot, the state machine rebuilds current state by folding the session's persisted events; it never guesses |

---

## 7. Versioning

`schema_version` is semver. Minor version = additive payload keys only.
Consumers must tolerate unknown keys. A major bump requires a migration note in
`CHANGELOG.md` and a parallel-run period where both versions are accepted.
