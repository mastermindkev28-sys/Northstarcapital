# SHADOW MODE PLAN

Deliverable **(14)**. Status: PROPOSED.

Shadow mode is the bridge between "the backtest says" and "the market did".
It runs the complete live pipeline on live data, computes everything, and places
nothing.

---

## 1. What runs and what does not

| Component | Shadow mode |
|---|---|
| Data feed | **Live** |
| All 28 engines | **Live** |
| State machine | **Live** |
| Events, database, archive | **Live** |
| Telegram alerts | **Live**, every message stamped `MODE: SHADOW` |
| Screenshots | **Live** |
| Entry / stop / targets / sizing | **Computed in full** |
| Fills | **Simulated** by the backtest fill model |
| Broker connection | **None.** `NullExecutionProvider` raises on any mutating call |

The `mode` field is present on every event, every alert, every archive row and
every screenshot. There is no configuration in which a shadow message could be
mistaken for a live one.

---

## 2. What is logged

For every session, whether or not a setup appears:

1. **What the system saw** — levels, ORB, bias, pools, every engine output.
2. **Why it qualified** — the confluence detail, quality components, risk
   validation, the complete positive path.
3. **Why it rejected** — every rejection code with its context, including the
   ones that fired after an earlier blocker (the full picture, not the first
   failure).
4. **What would have happened** — simulated fills, R, P&L, MAE/MFE, whether TP1
   and TP2 were reached, and where the runner would have trailed out.
5. **What the market did afterwards** — a 60-minute forward window recorded for
   every setup, traded or not, so rejected setups can be evaluated too.

Point 5 is the part most shadow systems omit, and it is the only way to measure
false negatives.

---

## 3. Success criteria before considering live execution

Minimums, all of which must hold simultaneously:

| Criterion | Threshold |
|---|---|
| Sessions observed | ≥ 60 trading sessions (≈ 3 months) |
| Setups archived | ≥ 40, with human review completed on all of them |
| Human/algorithm agreement | ≥ 80% on sweep, displacement and CISD classification |
| No-lookahead suite | passing continuously, zero exceptions |
| Pine ↔ Python parity | zero unexplained breaks over the period |
| Operational reliability | zero missed sessions from infrastructure failure; every event delivered |
| Fill-model calibration | shadow-assumed fills compared against observed prints; slippage assumptions revised and then re-validated |
| Expectancy | positive and consistent with the backtest's out-of-sample estimate — **or the discrepancy is explained**, not averaged away |
| Every `PROPOSED` parameter | either `VALIDATED` or explicitly accepted by you as an unvalidated risk |

If shadow expectancy materially undershoots the backtest, that gap is the
finding. The correct response is to understand it, not to go live and hope.

---

## 4. Shadow → live transition (not in current scope)

Even after the criteria are met, live execution is a separate project phase with
its own gates: an execution provider implementation, order-state reconciliation,
partial-fill handling, connection-loss policy, a kill switch, position
reconciliation on restart, and a period of minimum-size live trading (MNQ, one
contract) before any size is used.

Nothing in this repository today can place an order, and the transition is
deliberately not one configuration flag: `execution.allow_live` alone does
nothing without an implemented provider that does not yet exist.

---

## 5. Daily shadow review ritual

Each afternoon the system produces a review packet:

* Session summary and final state, with the reason for the day's outcome.
* Every setup with its screenshots and the full confluence/quality detail.
* The rejection ledger for the day — what was missing, in order.
* The forward-window outcome for each rejected setup.
* A one-line prompt for your human verdict on each setup.

Twenty minutes a day for sixty days produces the dataset that converts this
system from a set of plausible definitions into a validated one. That work is
the actual product; the code is just what makes it repeatable.
