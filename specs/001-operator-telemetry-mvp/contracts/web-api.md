# Contract: Web App Query & Management API

**Audience**: the Next.js control-plane UI (route handlers, server-side session auth). All endpoints are tenant-scoped (FR-007); the session's tenant is the only tenant ever visible — there is no tenant parameter to forge.
**Implements**: US1–US6 control-plane surfaces, FR-002, FR-004, FR-010, FR-018…FR-022, FR-028/029, FR-031/032.
**Data sources**: Postgres only in MVP (research R15) — the app never queries VictoriaMetrics directly; metric visualization lives in per-tenant Grafana orgs (linked via `GET /api/dashboards`).

**Conventions**: JSON; errors `{ "error": { "code": string, "message": string } }`; cursor pagination `?cursor=&limit=` (default 50, max 200) → `{ "items": [...], "next_cursor": string|null }`; all timestamps RFC 3339 UTC.

## Auth

| Endpoint | Notes |
|---|---|
| `POST /api/auth/signup` `{ email, password, tenant_name }` | creates Tenant + owner AccountUser; email verification required before token issuance |
| `POST /api/auth/login` `{ email, password }` | sets HttpOnly Secure session cookie; CSRF token for mutations |
| `POST /api/auth/logout` | |

## Tenant & data handling

| Endpoint | Notes |
|---|---|
| `GET /api/tenant` | profile, status, fleet_opt_in |
| `PATCH /api/tenant` | display_name, fleet_opt_in (self-host contribution toggle visible but informational for hosted) |
| `DELETE /api/tenant` | enters `deletion_pending`; response states purge window (≤ 30 d, FR-031) and aggregate irreversibility |
| `GET /api/data-handling` | published data-handling page content (FR-032): metrics-only collection, never-collected list (keys, blob payloads), retention 30 d/13 mo, isolation, deletion, k≥5 anonymity, single-region hosting |

## Ingest tokens (US1, FR-002/004)

| Endpoint | Notes |
|---|---|
| `POST /api/tokens` `{ label }` | → `{ id, token }` — **plaintext returned once**, never again; UI shows copy-once flow |
| `GET /api/tokens` | `{ id, label, prefix, issued_at, last_used_at, revoked_at }[]` (no secrets) |
| `POST /api/tokens/{id}/revoke` | effective on subsequent pushes (≤ 60 s edge propagation) |
| `GET /api/tokens/rejections` | recent `RejectedPushEvent`s (token prefix, reason, received_at) — misconfiguration debugging (FR-004) |
| `GET /api/ingest/guide` | ready-to-paste flag block: `--otel-endpoint https://otel.<domain>/t/<token>` plus agent-path snippet |

## Monitored servers (US1, FR-003/006/010/015)

| Endpoint | Notes |
|---|---|
| `GET /api/servers` | list with `connection_status`, `reported_version`, `support_classification`, `last_seen_at`, `agent_connected`, `stale` flag (derived; dashboards mark stale visually, FR-015) |
| `GET /api/servers/{id}` | detail + conflict info + unknown-series count (drift flag) + agent-path availability for host/disk view (US2.4, FR-022) |
| `POST /api/servers/{id}/resolve-conflict` `{ action: "keep_first" \| "keep_new" \| "rename", new_instance_id? }` | resolves identity collisions (edge case) |

## Dashboards (US2)

| Endpoint | Notes |
|---|---|
| `GET /api/dashboards` | per-group links into the tenant's Grafana org: `at_a_glance, uploads, downloads, storage_budget, signing, connections, runtime, host_disk`; `host_disk` returns `available: false` with explanation until `agent_connected` (US2.4) |

## Alert rules & incidents (US3, FR-017…FR-022)

| Endpoint | Notes |
|---|---|
| `GET /api/alert-rules` | effective rules = pack defaults overlaid with tenant overrides; each rule carries `pack_key, name, description, severity, enabled, thresholds, agent_path_required, version_gate, channel_ids, available` (false with reason when agent/version-gated) |
| `PATCH /api/alert-rules/{pack_key}` | `{ enabled?, thresholds?, channel_ids?, repeat_interval? }` — upserts tenant override; pack defaults remain fallback |
| `GET /api/incidents?state=&server_id=` | firing/resolved history with `triggering_values`, fired/resolved timestamps (US3.4) |
| `GET /api/incidents/{id}` | detail + per-channel `DeliveryRecord`s (status, attempts, sanitized response) |

## Notification channels (FR-018/021)

| Endpoint | Notes |
|---|---|
| `POST /api/channels` `{ type, config }` | types per `alert-channels.md`; secrets write-only, encrypted at rest |
| `GET /api/channels` | config redacted; includes `health`, `last_error`, `last_delivery_at` (US3.3) |
| `PATCH /api/channels/{id}` / `DELETE /api/channels/{id}` | |
| `POST /api/channels/{id}/test` | sends a test notification; returns delivery outcome; updates health |

## Fleet benchmarks (US6, FR-027…FR-029)

| Endpoint | Notes |
|---|---|
| `GET /api/fleet/benchmarks` | published aggregates: `{ canonical_name, cohort_key, period_date, p10, p50, p90, computed_at }[]`; cohorts < 5 tenants return `{ suppressed: true, explanation }` — never a number, never the cohort size (FR-028); read path re-checks k≥5 (SC-007) |
| `GET /api/fleet/my-comparison?canonical_name=` | tenant's own values alongside the published band; no other tenant's identity or values, ever |

## Registry (annotations context)

| Endpoint | Notes |
|---|---|
| `GET /api/registry?network=` | latest `RegistrySnapshot` (providers, taken_at, source_endpoint); informational — correlation and version-majority context for rule 10 |

## Status codes

`200/201` success · `400` validation · `401` unauthenticated · `403` cross-tenant or insufficient role (indistinguishable from 404 for existence-leak protection) · `404` not found · `409` conflicts (e.g. unresolved server identity) · `410` tenant deletion pending · `429` UI rate limits · `500` generic (no internals leaked).
