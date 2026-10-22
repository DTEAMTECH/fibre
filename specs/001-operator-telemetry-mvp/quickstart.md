# Phase 1 — Quickstart: End-to-End Validation on the Local Devnet

**Purpose**: prove the MVP works end-to-end without mainnet — stand up a local celestia-app devnet with Fibre, point the fibre server at the local FibreStats stack, verify every v10.4.0 metric lands, record the golden fixture, trigger an alert, view the dashboard.
**Nature**: validation/run guide only — no implementation code here (details belong to tasks.md and implementation). Expected outcomes are pass/fail observable.
**References**: `contracts/otlp-ingest.md`, `contracts/adapter-mapping-v10.4.0.md` (metric checklist), `contracts/self-host-packaging.md`, `data-model.md`.

## 0. Prerequisites

| Tool | Check |
|---|---|
| Docker + Compose v2 | `docker compose version` |
| Go 1.24+ (for the devnet binary) | `go version` |
| pnpm 9 (web app) | `pnpm --version` |
| celestia-app source at tag `v10.4.0` | `git clone --depth 1 --branch v10.4.0 https://github.com/celestiaorg/celestia-app.git` |
| This repo | working dir = repo root |

## 1. Stand up the local FibreStats stack

```bash
cd deploy && cp .env.example .env        # set FIBRESTATS_ADMIN_EMAIL
docker compose up -d                     # adapter, otelcol, victoriametrics, postgres, grafana, web
docker compose ps                        # wait until all healthchecks pass
```

**Expected**: `GET http://localhost:4318/healthz`, `:3001/api/health` (Grafana), `:3000/api/healthz` all return OK. Grafana login works with the seeded admin; a `selfhost` org exists with the seven provisioned dashboard groups (FR-014) and an 11-rule alert pack visible but mostly `pending` (no data yet).

## 2. Create a tenant and ingest token

Via web app at `http://localhost:3000`: sign up → verify email (dev mailbox) → "Add server" → create token labeled `devnet-1`.

**Expected**: the token plaintext is shown **once** (`fs_ingest_…`); the UI shows the ready-to-paste flag `--otel-endpoint http://localhost:4318/t/<token>` (contract `otlp-ingest.md` §2 — token-in-path, since the fibre binary cannot set headers). `GET /api/tokens` lists prefix + issued_at, no secret.

## 3. Stand up the devnet fibre server

```bash
cd celestia-app
./scripts/single-node-fibre.sh            # builds & runs local chain + fibre server
```

Alternative for multi-validator scenarios: Huginn's `fibre-devnet` (`github.com/plsgiveup/fibre`).

**Expected**: fibre server up with gRPC on `:7980`, config at `~/.celestia-fibre/config/server_config.toml`, **OTLP off** (default — verified at v10.4.0).

## 4. Point the fibre server at FibreStats

Restart the fibre server with the flag from step 2:

```bash
fibre start --otel-endpoint http://localhost:4318/t/fs_ingest_<token>   # or FIBRE_OTEL_ENDPOINT
```

The binary will POST to `…/t/<token>/v1/metrics` (and `/v1/traces`, which the edge accepts and discards — FR-005).

**Expected (US1 acceptance)**:
1. Within 60 s: the server shows **connected** in the web app (`GET /api/servers`), with `reported_version = v10.4.0` and `support_classification = full`.
2. Adapter logs show token validated, `tenant_id`/`server_id` injected; no `RejectedPushEvent` rows.
3. Negative check: restart with a garbage token → pushes rejected `401`; the rejection appears under the tenant's ingestion-status view (FR-004); other (none here) tenants unaffected.

## 5. Generate activity and verify every metric lands

Drive real traffic on the devnet: submit blobs through a fibre client (`tools/fibre-txsim` in celestia-app, or a small celestia-node blob submit against the devnet), then query a few shards back. Then, in Grafana Explore (or `http://localhost:8428` VM UI) scoped to the tenant, confirm presence of each baseline series (contract `adapter-mapping-v10.4.0.md` §2):

| # | Series | Requires activity? |
|---|---|---|
| 1–3 | `fibre.server.upload_shard.{in_flight,duration,bytes}` | yes — at least one blob upload |
| 4 | `fibre.server.upload_shard.rejected{reason="budget_exceeded"}` | only if budget exceeded (step 7b optional) |
| 5 | `fibre.server.upload_shard.last_success_timestamp` | yes — **absent before first success** (expected, not a bug) |
| 6–7 | `fibre.server.upload_shard.{occupancy_bytes,budget_bytes}` | yes — budget gauge appears once the limiter derives it |
| 8–10 | `fibre.server.download_shard.{in_flight,duration,bytes}` | yes — at least one shard download |
| 11–12 | `fibre.server.store.{put,get}.duration` | yes — follows 1 and 8 |
| 13–15 | `fibre.server.backend.get.{duration,in_flight,bytes}{backend="local",…}` | yes — follows downloads |
| 16 | `fibre.server.sign.duration` | yes — payment-promise signing during upload |
| 17–18 | `fibre.server.prune.{entries,duration}` | `entries` only when something pruned (`pruned > 0`); `duration` each prune tick |
| — | `go.*` runtime series | automatic whenever OTLP is on |
| — | `fibre.client.*` (if the txsim/client binary also points at the stack) | optional; must land under `namespace=client` with no alerts |

**Expected**: every applicable series present with labels `tenant_id`, `server_id`, `namespace`; zero unknown-series flags for v10.4.0 (any flag = contract-drift bug to file). Timestamps track devnet time within the skew policy.

## 6. Record the golden fixture (becomes the adapter contract test)

Capture one full `ExportMetricsServiceRequest` from the devnet session into `fibrestats/services/ingest/test/fixtures/golden/v10.4.0.otlp.pb` (ingest-service debug flag or a tcpdump-to-protobuf capture; mechanics decided in tasks).

**Expected (FR-013/SC-005)**: `pnpm --filter @fibrestats/ingest exec vitest run golden_v10_4_0` passes — 100% of fixture `fibre.server.*` series covered by the manifest; attribute schemas match; no unexpected names.

## 7. Trigger alert rules (deterministic)

**7a. Telemetry silence (rule 11 — primary, fully deterministic)**: stop the fibre server.

**Expected**: after 3 missed export intervals the incident fires (`GET /api/incidents?state=firing`); the configured test channel (a generic webhook pointed at a local request bin) receives a **firing** payload per `contracts/alert-channels.md` §1 marked "telemetry stopped"; dashboards mark the server's data **stale** (not last-value-as-current, FR-015). Restart the server → **resolution** notice arrives; incident shows fired_at/resolved_at/triggering values (US3.4).

**7b. Budget rejection (rule 2 — optional)**: with the devnet's small derived budget, submit blobs until occupancy exceeds it.

**Expected**: `fibre.server.upload_shard.rejected{reason="budget_exceeded"}` increments; the `budget_exceeded_rejections` incident fires with the rejection count in `triggering_values`; utilization panels read `occupancy_bytes / budget_bytes` — **the exported gauges, never stake math**.

**7c. Channel failure surfacing (US3.3)**: point the webhook channel at a dead URL, trigger 7a again.

**Expected**: after 3 attempts the channel shows `health=failing` with a sanitized error in the web app (FR-021); the failure is visible on the incident's delivery records.

## 8. View the dashboards (US2)

Open the tenant's Grafana org → **at-a-glance**: upload/download activity and latency, occupancy vs budget (gauges from step 5), signing latency, connection state — all updating live. Select a past range: data renders from the raw tier (devnet is minutes old; the 30 d → 13 mo tier fallback is validated by VM retention config inspection, FR-016/030). **Host & disk** group shows the "requires the agent path" notice (US2.4) — then optionally run the published Alloy config (`deploy/agent/alloy/`) against node-exporter + the devnet consensus node and watch `agent_connected` flip and the panels populate (FR-022/023).

**Isolation check**: create a second tenant + token, point a second devnet fibre instance at it; confirm tenant A's dashboards, servers, incidents, and API show nothing of tenant B (FR-007/SC-008).

## 9. Fleet aggregation dry run (US6 mechanics)

Seed ≤ 4 tenants (steps above × N with distinct tokens) and run `docker compose exec adapter adapter aggregate --now`.

**Expected**: with < 5 contributing tenants, `GET /api/fleet/benchmarks` returns `suppressed: true` with an explanation — never a number, never the cohort size (FR-028). With ≥ 5 (repeat seeding): p10/p50/p90 bands appear; own-values comparison renders alongside (FR-029); no tenant identifiable. Self-host opt-in default-off check: contribution preview endpoint (contract `self-host-packaging.md` §4) shows payload but publisher stays disabled unless `FIBRESTATS_FLEET_OPT_IN=true`.

## 10. Teardown

```bash
docker compose down -v     # also proves purge: volumes removed
pkill -f single-node-fibre # devnet
```

## Validation summary (all must pass)

1. Zero-install onboarding: flag-only connection, connected ≤ 1 min, version `v10.4.0` classified `full` (SC-001 flow).
2. All 18 baseline instruments land; client/runtime namespaces handled per policy; no false drift flags.
3. Golden fixture recorded; contract test green.
4. Alert fires ≤ 5 min from condition (SC-003), resolution always delivered, failures surfaced as channel health.
5. Dashboards tenant-isolated, staleness marked, agent-gated views honest.
6. k≥5 suppression demonstrably holds in both directions.
