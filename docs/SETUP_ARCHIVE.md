# NORTHSTAR SETUP ARCHIVE™

Status: PROPOSED.

The archive is the point of the whole system. Signals are transient; the archive
is what turns a year of trading into a dataset you can actually learn from.

---

## 1. What a setup record contains

Every setup — **including those that never became trades** — is permanently
stored under its `setup_id` (`2026-09-05-NQ-SETUP001`), which is never reused.

| Group | Fields |
|---|---|
| Identity | setup_id, session_id, sequence_no, symbol, contract, date, times |
| Classification | setup type, direction, final state, max milestone, outcome |
| Context | ORB high/low/size/status, bias, bias score, all session levels, news |
| Liquidity | pools identified, which was swept, penetration, reclaim, wick % |
| Delivery | displacement score + all components, FVGs with fill %, structure events |
| Zones | order blocks, rejection blocks, CISD reference and confirmation |
| Retest | zone used, time to retest, MFE before retest, depth, failure reason |
| Confirmation | each boolean trigger, with timestamps |
| Confluence | all 12 factors with true/false/UNAVAILABLE and the reason for each |
| Quality | nine component scores, weights used, total, band |
| Risk | entry, stop, stop distance, ORB fraction, contracts, dollar risk, TP1, TP2, TP2 source, path status and severity |
| Outcome | result, R, P&L, MAE, MFE, TP1/TP2 hit, exit reason, fills |
| Evidence | every screenshot + sidecar JSON |
| Communication | every alert sent, with signal_id and timestamp |
| Process | every state transition with guard detail and duration |
| Rejections | the complete reason ledger for this setup |
| Provenance | config_hash, param_version, code git SHA, mode, data source |
| Review | human verdict, human classification, notes, disagreement tags |

**Rejected and invalidated setups are the most valuable rows in the database.**
A system that only stores its trades cannot tell you what it is systematically
missing.

---

## 2. Replay

```
northstar replay 2026-09-05-NQ-SETUP001
```

Returns, from persisted data alone:

1. The bar tape for the session window.
2. Every state transition in order, with timestamps and guard details.
3. Every event with its full payload.
4. Every screenshot plus its context sidecar.
5. Every alert as sent.
6. All levels, zones and pools as they existed at each moment.
7. Every confluence and quality evaluation.
8. Every risk calculation.
9. Every rejection reason.
10. The outcome.

**Deterministic re-derivation:** because `config_hash` and the code SHA are
stored, `northstar replay --recompute` re-runs the engines over the stored bars
and asserts the regenerated event stream is identical to the archived one. A
mismatch means either the code changed behaviour or the archive is corrupt —
either way you want to know, and the nightly job checks a sample automatically.

---

## 3. Human vs algorithm review

Reviewing is a first-class workflow, not an afterthought.

```
/review 2026-09-05-NQ-SETUP001
  verdict:        PASS | FAIL | UNSURE
  human_type:     CONTINUATION | NEWS_REVERSAL | NEITHER
  human_direction:LONG | SHORT | NONE
  disagreement:   [not_a_real_sweep, cisd_too_late, zone_too_wide, ...]
  notes:          free text
```

`analyticsEngine` then reports:

* Agreement rate overall and per engine.
* **Which engine causes disagreement most often** — this is the refinement
  signal. If 70% of your `FAIL` verdicts carry `not_a_real_sweep`, the sweep
  definition (`A-03`) is the thing to fix, not the confluence weights.
* Setups the algorithm rejected that you marked as valid (false negatives) and
  those it accepted that you marked invalid (false positives).
* Drift over time: agreement should improve as definitions are validated.

Target: review every setup for the first 60 sessions. That is the sample that
turns the `PROPOSED` parameters into `VALIDATED` ones.

---

## 4. Query surface

Exposed through a small CLI and read-only SQL views:

```sql
-- every A+ setup that was rejected, and why
SELECT s.setup_id, q.total, r.code, r.message
FROM setups s
JOIN quality_snapshots q USING (setup_id)
JOIN rejections r USING (setup_id)
WHERE q.total >= 90 AND s.outcome <> 'TRADED';

-- realised R by confluence count (the question the confluence score exists to answer)
SELECT c.total AS confluence, count(*) n,
       avg(t.r_multiple) avg_r, percentile_cont(0.5)
       WITHIN GROUP (ORDER BY t.r_multiple) median_r
FROM confluence_snapshots c JOIN trades t USING (setup_id)
GROUP BY 1 ORDER BY 1;

-- invalidation reasons ranked
SELECT code, count(*) FROM rejections GROUP BY 1 ORDER BY 2 DESC;
```

Standard views shipped: `v_setup_full` (one wide row per setup),
`v_daily_summary`, `v_confluence_performance`, `v_quality_performance`,
`v_rejection_ledger`, `v_human_agreement`.

---

## 5. Export

* `northstar export --setup <id> --format bundle` → a zip containing the JSON
  record, all screenshots, sidecars, and a rendered HTML review page.
* `northstar export --range 2026-01-01:2026-06-30 --format parquet` → research
  dataset for notebooks.
* Exports always include `config_hash`, `param_version` and code SHA. A dataset
  without its provenance is not usable for comparison.

---

## 6. Integrity guarantees

1. Archive rows are append-only; corrections are new rows referencing the
   original, never in-place edits.
2. Screenshot hashes are verified nightly.
3. Every setup must reach a terminal `outcome`; a stuck setup raises `OPS_ALERT`.
4. Backups: nightly encrypted dump, 30 daily / 12 monthly retention, with a
   quarterly restore test. An untested backup is not a backup.
