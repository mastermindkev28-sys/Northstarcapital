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
| Current phase | **PHASE 1 — ARCHITECTURE. AWAITING APPROVAL.** |

---

## Status

This repository currently contains **specification and architecture only**.

No engine code, no Pine strategy, no backend service has been written. That is
deliberate: the specification requires the full architecture, schemas, exact
algorithmic definitions and the ambiguity register to be reviewed and approved
before Phase 2 begins.

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

## Decisions required before Phase 2

Twelve blocking questions and 24 ambiguity entries are listed in
[`docs/AMBIGUITY_REGISTER.md`](docs/AMBIGUITY_REGISTER.md). The blocking set is
summarised at the top of that file. Phase 2 does not start until those are
answered, because each one changes what the code computes.

## Honest limitations stated up front

1. **Order flow.** True absorption and delta require bid/ask trade data. Bar
   volume is not order flow. Until a real feed is attached, the system reports
   `ORDER FLOW DATA UNAVAILABLE` and those two confluence factors score zero.
   They are never synthesised from volume.
2. **Confluence count is not a probability.** `6/12` is an inventory of present
   conditions, not a 50% edge, until the archive contains enough validated
   samples to publish measured hit rates.
3. **Quality score weights are guesses today.** They are declared, versioned and
   configurable specifically so they can be replaced by fitted weights later.
4. **Two-brain risk.** A Pine indicator and a Python engine can drift apart. The
   architecture treats Python as the source of truth and enforces a parity test
   harness; see `docs/ARCHITECTURE.md` § Dual-Runtime Parity.
5. **This is not financial advice and carries no expectation of profit.**
   Futures trading involves substantial risk of loss.

## Licence / confidentiality

Proprietary. NORTHSTAR CAPITAL™ and NORTHSTAR MOTION MODEL™ are used here as
internal product names for this private system.
