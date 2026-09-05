# NORTHSTAR CAPITAL™

**Algorithmic Market Intelligence System**
Trading methodology: **NORTHSTAR MOTION MODEL™**

| Field | Value |
|---|---|
| Primary market | NQ futures (CME, E-mini Nasdaq-100) |
| Secondary market | MNQ futures (Micro E-mini Nasdaq-100) |
| Timezone | `America/New_York` (all internal session logic; storage in UTC) |
| Opening range | 08:00–08:15 ET |
| Primary execution window | 08:30–11:00 ET |
| Flat by | 11:30 ET |
| Current phase | **PHASE 2 — TRADINGVIEW INDICATOR (built).** |

---

## Status

The deliverable is a **TradingView indicator**:
[`tradingview/northstar_motion_model.pine`](tradingview/northstar_motion_model.pine)
— see [its usage guide](tradingview/README.md).

The backend, Telegram, screenshot, database and archive designs in `docs/`
remain as forward architecture and are **not built**. Scope decisions are
recorded in [`docs/DECISIONS.md`](docs/DECISIONS.md).

**Nothing in this repository can place an order, and nothing is connected to a
broker.**

## What this system is

This is not a buy/sell indicator. It is a decision-support and research
platform that answers, at every moment of the session:

> Where are we? What happened? What is developing? What is confirmed?
> What is missing? What invalidates this? What am I waiting for?

Every state is derived from an explicit, configurable, testable definition.
Every signal carries its reason. Every setup carries a permanent ID.

## Read in this order

| # | Document | Covers spec deliverable |
|---|---|---|
| 0 | [`tradingview/README.md`](tradingview/README.md) | **The indicator: install and usage** |
| 0 | [`docs/DECISIONS.md`](docs/DECISIONS.md) | **Decisions log — authoritative over the rest** |
| 1 | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System architecture, data-flow diagram, module dependency map |
| 2 | [`docs/STATE_MACHINE.md`](docs/STATE_MACHINE.md) | State-machine diagram and full transition table |
| 3 | [`docs/TRADING_MODEL.md`](docs/TRADING_MODEL.md) | Exact algorithmic definitions |
| 4 | [`docs/AMBIGUITY_REGISTER.md`](docs/AMBIGUITY_REGISTER.md) | **Ambiguous definitions requiring your validation** |
| 5 | [`docs/PARAMETERS.md`](docs/PARAMETERS.md) | Parameter configuration table |
| 6 | [`docs/DATABASE_SCHEMA.md`](docs/DATABASE_SCHEMA.md) | Database schema |
| 7 | [`docs/EVENT_SCHEMA.md`](docs/EVENT_SCHEMA.md) | Event schema |
| 8 | [`docs/TELEGRAM_SETUP.md`](docs/TELEGRAM_SETUP.md) | Telegram architecture |
| 9 | [`docs/SCREENSHOT_SYSTEM.md`](docs/SCREENSHOT_SYSTEM.md) | Screenshot architecture |
| 10 | [`docs/RISK_RULES.md`](docs/RISK_RULES.md) | Risk, sizing, targets, daily governor |
| 11 | [`docs/SETUP_ARCHIVE.md`](docs/SETUP_ARCHIVE.md) | Setup archive and replay |
| 12 | [`docs/TESTING.md`](docs/TESTING.md) | Testing plan |
| 13 | [`docs/BACKTESTING.md`](docs/BACKTESTING.md) | Backtesting plan |
| 14 | [`docs/SHADOW_MODE.md`](docs/SHADOW_MODE.md) | Shadow-mode plan |
| 15 | [`docs/SECURITY.md`](docs/SECURITY.md) | Security plan |
| 16 | [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | Deployment plan |

Proposed default parameter surface: [`config/northstar.defaults.yaml`](config/northstar.defaults.yaml)

## Decisions taken

| ID | Decision |
|---|---|
| D-001 | Scope is a TradingView indicator; no bot, backend or execution layer |
| D-002 | Execution timeframe is 1 minute |
| D-003 | Trading day is 00:00–23:59 Pacific (= 03:00–02:59 ET) for prior-day levels |
| D-004 | ORB is 05:00–05:15 PT (= 08:00–08:15 ET), locked at the close of the window |
| D-005 | Stop-rule contradiction deferred; stop filters ship default-off |
| D-006 | CISD removed; confluence is now X / 9 |

Full reasoning and consequences in [`docs/DECISIONS.md`](docs/DECISIONS.md).
The remaining threshold ambiguities are all indicator inputs, documented in
[`docs/AMBIGUITY_REGISTER.md`](docs/AMBIGUITY_REGISTER.md).

## Honest limitations stated up front

1. **Order flow.** True absorption and delta require bid/ask trade data. Bar
   volume is not order flow. Until a real feed is attached, the system reports
   `ORDER FLOW DATA UNAVAILABLE` and those two confluence factors score zero.
   They are never synthesised from volume.
2. **Confluence count is not a probability.** `6/9` is an inventory of present
   conditions, not an edge, until it has been measured against outcomes on a
   real sample.
3. **Quality score weights are guesses today.** They are declared, versioned and
   configurable specifically so they can be replaced by fitted weights later.
4. **Non-repainting costs latency.** Pivots are confirmed `pivot_right` bars
   late and state advances only on closed bars, so signals appear later than a
   hindsight-repainting script would show them. That is the correct trade.
5. **This is not financial advice and carries no expectation of profit.**
   Futures trading involves substantial risk of loss.

## Licence / confidentiality

Proprietary. NORTHSTAR CAPITAL™ and NORTHSTAR MOTION MODEL™ are used here as
internal product names for this private system.
