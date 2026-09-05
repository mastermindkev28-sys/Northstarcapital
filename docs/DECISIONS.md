# DECISIONS LOG

Answers to the Phase 1 blocking questions, and the scope change they implied.
This file is authoritative over any conflicting statement in the other docs.

---

## D-001 · Scope: TradingView indicator only  (2026-09-05)

**Decision.** Build a TradingView (Pine v6) indicator. No trading bot, no
backend service, no Telegram bot, no database, no execution layer at this stage.

**Consequences.**
- `tradingview/northstar_motion_model.pine` is the deliverable.
- The backend / Telegram / screenshot / database / archive designs in `docs/`
  remain valid as forward architecture, but are **not being built now**.
- The alert payloads are still emitted in the documented JSON envelope shape, so
  a backend can be added later without changing the indicator.
- Risk figures (stop, TP1, TP2, R) are **displayed**, not enforced. No order is
  ever placed; nothing in this repository can trade.

**Resolves:** B5, B6, B7, B8, B10 — deferred, not applicable to an indicator.

---

## D-002 · Execution timeframe: 1 minute  (resolves B1 / A-01)

**Decision.** 1-minute chart is the execution timeframe.

**Consequences.** All engines — displacement, FVG, structure, order blocks,
rejection blocks, retest — evaluate on 1m. The indicator warns in the dashboard
if it is loaded on any other timeframe, because the ORB, sweep and displacement
definitions assume 1m bars.

---

## D-003 · Trading day boundary: 00:00–23:59 Pacific  (resolves B4 / A-02)

**Decision.** The trading day for previous-day high/low/open/close runs from the
daily open at **00:00 PT to 23:59 PT**.

**Consequences.**
- `PDH` / `PDL` / `PDO` / `PDC` are computed over the completed 00:00–23:59 PT
  day. In Eastern terms that is 03:00–02:59 ET.
- Pacific and Eastern shift together across DST, so this boundary is a constant
  three hours ahead of ET year-round; the indicator uses a timezone-aware date,
  not a fixed UTC offset, so DST is handled automatically.
- `DAY OPEN` = the open of the 00:00 PT bar.
- `MIDNIGHT OPEN (ET)` is retained as a separate key-open level (00:00 ET =
  21:00 PT), tracked as the most recent occurrence rather than reset at the
  Pacific day boundary, because it falls in the previous Pacific day.
- Superseded: the earlier proposal of a CME futures day (18:00–17:00 ET).

---

## D-004 · ORB window: 05:00–05:15 PT  (confirms the spec)

**Decision.** The opening range is built from **05:00 to 05:15 PT**.

**Consequences.** This equals 08:00–08:15 ET, so it matches the original
specification exactly — it was simply restated in Pacific time. The window is a
session input, changeable in one click if a different range is wanted. The range
locks at 05:15 PT and is never modified afterwards.

All other session times follow in Pacific: execution window 05:30–08:00 PT
(= 08:30–11:00 ET), flat 08:30 PT (= 11:30 ET), overnight 15:00–05:00 PT
(= 18:00–08:00 ET).

---

## D-005 · Stop-rule contradiction: deferred  (defers B2 / A-15)

**Decision.** Not a blocker — no bot is being built, so no stop rule is being
enforced.

**Consequences.** The indicator computes and draws stop, TP1 and TP2 for
reference. The 10-point minimum, 30-point maximum and ORB/3 checks are
implemented as **optional, default-off filters**. With them off, the
contradiction documented in `RISK_RULES.md` §1.1 cannot arise. It must be
resolved before any automated execution is built.

---

## D-006 · CISD: removed  (closes B3 / A-10)

**Decision.** Drop CISD from the model.

**Consequences.**
- No `cisdEngine`, no CISD confluence factor, no CISD confirmation trigger.
- The confluence set drops from 12 factors to **9 available factors**:
  liquidity sweep, displacement, FVG, order block, rejection block, daily bias,
  premium/discount, key open, clean target path.
- Delta divergence and absorption remain in the model as declared factors but
  are permanently `UNAVAILABLE` in a Pine indicator — TradingView provides no
  true bid/ask order-flow data, and volume is never substituted for it. They are
  displayed as `N/A` and excluded from the denominator.
- Display is therefore `X / 9`, with the three unavailable factors named
  explicitly rather than silently dropped.
- The confirmation engine now triggers on structure break, rejection candle,
  order block or FVG — not CISD.
- Quality score keeps its nine components; the confirmation component is scored
  from the remaining triggers.

---

## Still open (not blocking the indicator)

| ID | Question | Why it can wait |
|---|---|---|
| B9 / A-24 | Do pre-05:30 PT ORB sweeps count toward "both sides swept"? | Default: only sweeps from 05:30 PT count. Configurable input. |
| B11 | Does an unfilled invalidation consume a trade slot? | No trade counting in an indicator. |
| B12 / A-23 | Absolute vs ATR-normalised ORB bounds | Absolute 10/100 implemented; ATR mode is a config switch for later. |
| A-03 … A-25 | All threshold defaults | Every one is an indicator input, tuneable live on the chart. |
