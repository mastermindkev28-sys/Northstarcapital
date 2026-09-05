# SECURITY PLAN

Deliverable **(15)**. Status: PROPOSED.

---

## 1. Secrets

**Never in code, never in config YAML, never in git, never in a Pine script,
never in a log line.**

| Secret | Where it lives | Rotation |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | env / secret manager | on suspicion; 12 months routine |
| `TELEGRAM_CHAT_ID` | env (not secret, but not public) | — |
| `TELEGRAM_WEBHOOK_SECRET` | env | 90 days |
| `WEBHOOK_SECRET` (TradingView) | env | 90 days |
| `DATABASE_URL` | env / secret manager | on infrastructure change |
| `OBJECT_STORE_KEY/SECRET` | env / secret manager, scoped to one bucket | 90 days |
| Broker credentials | **not present, not stored** — no live execution exists | n/a |

Mechanisms:

* `.env` is gitignored; `.env.example` contains names and empty values only.
* `pydantic-settings` loads and validates at boot; a missing required secret is
  a hard startup failure, never a silent default.
* Production uses the host's secret manager (Docker secrets / systemd
  `LoadCredential` / cloud KMS), not a plaintext `.env`.
* `detect-secrets` or `gitleaks` runs as a pre-commit hook and in CI.
* Log formatter redacts any value matching a known secret and any key matching
  `token|secret|key|password|authorization`.
* Secrets are never included in event payloads, screenshots, sidecar JSON,
  archive exports, or error responses.

---

## 2. Webhook security (TradingView → backend)

Layered, because no single layer is sufficient:

1. **TLS 1.2+ only**, valid certificate, HSTS. No plaintext endpoint exists.
2. **Source IP allowlist** — TradingView publishes its webhook egress IPs;
   the list is held in configuration and verified against their documentation at
   deploy time, not hard-coded in application code.
3. **Shared secret** in the body, compared with `hmac.compare_digest`.
4. **Strict schema validation** (pydantic), 16 KB body cap, JSON only.
5. **Freshness window** ±120 s to bound replay.
6. **Idempotency** on `event_id`, 48 h.
7. **Rate limit** per source IP.
8. Failures return `202 Accepted` with the event discarded and logged — a
   prober learns nothing from the response.

**Stated weakness, honestly:** a TradingView alert body is visible to anyone who
can open the alert dialog in your TradingView account, so the shared secret is a
low-trust credential. This is mitigated by the IP allowlist, TLS, the freshness
window, rotation, and — most importantly — by the fact that **the webhook has no
code path that can place an order or change configuration.** Its maximum
authority is to record an event and send you a message.

---

## 3. Telegram security

* Webhook endpoint verifies `X-Telegram-Bot-Api-Secret-Token`.
* Every inbound update's `chat.id` is compared against `TELEGRAM_CHAT_ID`;
  anything else is dropped silently, not answered.
* All commands are read-only. There is no command that changes configuration,
  parameters, mode, or state, and no command that can trigger an order.
* Command rate limit per chat.
* Message bodies never contain secrets, account identifiers, or position sizes
  in a way that identifies an account.

---

## 4. Application security

| Area | Control |
|---|---|
| Dependencies | pinned with hashes; `pip-audit` in CI; Dependabot |
| Container | non-root user, read-only root filesystem, no shell in the runtime image, minimal base |
| Database | least-privilege role (no DDL at runtime), TLS connection, no superuser |
| Admin endpoints | separate auth, bound to localhost or a VPN interface, never public |
| Object store | private bucket, no public ACLs, server-side encryption, scoped credentials |
| Input | every external input schema-validated at the boundary; no `eval`, no dynamic imports, no shell interpolation |
| Logging | structured JSON, redaction filter, no PII, no secrets |
| Backups | encrypted at rest, restore-tested quarterly |
| Egress | outbound allowlist — Telegram API, data vendor, object store; nothing else |

---

## 5. Execution safety (defence in depth)

Because this is the category where a bug costs money:

1. `execution.provider = "null"` by default; `NullExecutionProvider` raises on
   every mutating method.
2. `execution.allow_live = false` is a second, independent gate.
3. No broker credentials exist in any environment, so even a bypass of both
   gates has nothing to authenticate with.
4. The webhook and Telegram surfaces have no reachable path to any
   `ExecutionProvider` method — enforced by an architecture test.
5. Risk invariants (`RISK_RULES.md` §6) are asserted immediately before any
   order would be constructed, and an assertion failure aborts rather than
   degrades.
6. A kill switch (`ops.halt = true`) stops all state advancement and alerting
   within one bar, and is checked on every bar.

---

## 6. Operational security

* SSH key-only access, no password auth, firewall default-deny inbound except
  443 and the management interface.
* Automatic OS security updates; container image rebuilt weekly.
* Separate staging environment with its own secrets; production secrets never
  appear in staging.
* Audit: every config change is a git commit with a `param_version` bump; every
  deploy records the code SHA and config hash.

---

## 7. Data sensitivity

The archive contains your trading behaviour, position sizes and P&L. That is
commercially sensitive personal data:

* Private repository, private database, private bucket.
* Exports are local by default; nothing is uploaded to third-party services.
* No analytics, telemetry or crash reporting sends trade data anywhere.
* Screenshots may contain account information if the headless TradingView
  backend is ever used — one more reason the server-side renderer is the
  default.

---

## 8. Incident response

| Event | Action |
|---|---|
| Suspected secret leak | rotate immediately, invalidate the bot token, review access logs, purge git history if committed |
| Unexpected webhook traffic | tighten the allowlist, rotate `WEBHOOK_SECRET`, inspect quarantined events |
| Unexplained state transition | halt via kill switch, replay the session from the archive, compare to the recorded event stream |
| Parity break in production | disable Pine-sourced events, run Python-only, investigate before re-enabling |
| Data-integrity alert | halt the session; never trade through a gap |
