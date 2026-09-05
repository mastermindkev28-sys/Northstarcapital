# DEPLOYMENT PLAN

Deliverable **(16)**. Status: PROPOSED.

---

## 1. Environments

| Environment | Purpose | Mode | Data |
|---|---|---|---|
| `local` | development, tests, research | `backtest` | recorded tapes, SQLite |
| `staging` | continuous shadow running the same code as prod | `shadow` | live feed, separate DB + bot |
| `production` | the operational system | `shadow` (live execution not implemented) | live feed |

Staging and production run **identical images**; only configuration and secrets
differ. Anything that only works in production is a defect.

---

## 2. Topology

```
Internet
   │ 443 TLS
   ▼
Reverse proxy (Caddy or nginx) ── automatic certificates, HSTS
   │  (IP allowlist for /webhook/tradingview)
   ▼
FastAPI (uvicorn, 1–2 workers)  ──►  Postgres 16 (local volume, TLS)
   │                                   ▲
   ├──►  background workers            │
   │      screenshot · brief · eod ────┘
   └──►  object store (S3-compatible, private bucket)
```

Sizing: this system handles tens of events per day. A single small VPS
(2 vCPU / 4 GB / 40 GB SSD) in a US-East region is ample; low latency to
TradingView's webhook egress and to the data vendor is the only locality
requirement. There is no horizontal scaling story because there is no scale
problem — correctness and uptime are the constraints.

---

## 3. Containers

```
docker compose:
  proxy       caddy:2            TLS, allowlist, rate limit
  api         northstar/api      FastAPI + uvicorn, non-root, read-only rootfs
  worker      northstar/api      same image, worker entrypoint
  db          postgres:16        volume-backed, TLS, least-privilege app role
  backup      sidecar            nightly pg_dump → encrypted → object store
```

Image build: multi-stage, pinned base digests, no shell in the runtime layer,
`HEALTHCHECK` hitting `/healthz`.

---

## 4. Configuration and secrets at deploy time

```
NORTHSTAR_ENV=production
NORTHSTAR_MODE=shadow
NORTHSTAR_CONFIG=/app/config/northstar.defaults.yaml
NORTHSTAR_PROFILE=/app/config/profiles/production.yaml
DATABASE_URL=…                       (secret)
TELEGRAM_BOT_TOKEN=…                 (secret)
TELEGRAM_CHAT_ID=…
TELEGRAM_WEBHOOK_SECRET=…            (secret)
WEBHOOK_SECRET=…                     (secret)
OBJECT_STORE_ENDPOINT/KEY/SECRET=…   (secret)
```

Secrets are injected by the host's secret mechanism, never baked into images and
never present in the compose file in plaintext. Boot fails loudly if any
required secret is absent.

---

## 5. Release process

1. PR → CI: lint, typecheck, unit, property, edge, timezone, **no-lookahead**,
   golden, integration, failure suites.
2. Merge → build image tagged with the git SHA.
3. Deploy to **staging**; run one full shadow session; confirm parity and event
   integrity.
4. Promote the identical image to production during a closed-market window
   (after 16:00 ET, never between 07:00 and 12:00 ET).
5. Record deploy: code SHA, `config_hash`, `param_version`, timestamp.
6. Rollback = redeploy the previous image tag; migrations are forward-only and
   additive so a rollback never loses archive data.

**Deploy freeze:** no production deploys inside the trading window. A live
session is never interrupted by a release.

---

## 6. Daily operational timeline (ET)

| Time | Action |
|---|---|
| 06:30 | pre-flight: data feed reachable, DB healthy, disk/quotas, bot reachable, holiday calendar checked |
| 07:00 | pre-market engine builds levels, bias, pools |
| 07:45 | Morning Brief sent |
| 08:00 | ORB build begins |
| 08:15 | ORB locked, verdict published |
| 08:30 | execution window opens; monitoring begins |
| 11:00 | no new entries |
| 11:30 | flat; session wind-down |
| 12:00 | EOD worker: session summary, archive close, integrity audit, review packet |
| 22:00 | nightly: backup, screenshot hash verification, historical + parity suites, replay-recompute on a sample |

Each step emits a heartbeat. A missing heartbeat raises an `OPS_ALERT` to the
operations chat — a silent failure at 08:14 is the expensive one.

---

## 7. Monitoring

| Signal | Alert |
|---|---|
| `/healthz` failing | immediate |
| No bars received for > 2 min inside 07:00–12:00 | immediate |
| Session did not reach `ORB_LOCKED` by 08:16 | immediate |
| Event sequence gap | immediate |
| Alert queue depth > 20 or age > 10 min | immediate |
| Parity break | immediate |
| Screenshot failures > 2 in a session | daily digest |
| Disk > 80%, backup failure, integrity mismatch | daily digest |

Operational alerts go to a **separate Telegram channel** from trading alerts, so
infrastructure noise never contaminates the trading signal stream.

---

## 8. Backup and recovery

* Nightly `pg_dump`, encrypted, uploaded off-host. 30 daily + 12 monthly.
* Object store versioning enabled for screenshots.
* **Quarterly restore drill** into a scratch environment, verifying that a
  random archived setup replays completely from the restored data. An untested
  backup is not a backup.
* RPO 24 h for the archive; RTO 4 h. The trading model is stateless across
  restarts (state is rebuilt by folding persisted events), so recovery is
  primarily a data-restoration exercise.

---

## 9. Phase-by-phase deployment sequence

| Phase | Deployed when |
|---|---|
| 1–20 (engines, state machine) | local only; no service exposed |
| 21–22 (dashboard, alerts) | TradingView only; no backend exposure |
| 23 (webhook) | staging first, IP allowlist verified before production |
| 24 (Telegram) | staging bot first; production bot only after a clean smoke test |
| 25–27 (screenshots, DB, archive) | staging, with a full shadow session before promotion |
| 28–29 (analytics, backtesting) | local/research; no production surface |
| 30 (shadow) | production, 60-session minimum observation |
| 31 (human validation) | concurrent with shadow |
| 32 (execution interface) | interface only; no provider implementation, no credentials |
