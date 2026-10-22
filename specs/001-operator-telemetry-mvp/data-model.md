# Phase 1 — Data Model: Operator-Side Telemetry MVP

**Date**: 2026-10-21 | **Source**: spec.md Key Entities + FRs, research.md decisions.
**Stores**: Postgres (all entities below) and VictoriaMetrics (metric points; only its label/retention contract is modeled here).

Naming: `snake_case` columns; `id` is UUIDv7 (time-ordered) unless noted; all `*_at` are `timestamptz`. Every tenant-owned row carries `tenant_id` with a foreign key and is queried only through tenant-scoped paths (FR-007).

## Entity overview

| Entity | Store | Purpose |
|---|---|---|
| Tenant | Postgres | operator organization (validator, infra provider, or opt-in self-host contributor) |
| AccountUser | Postgres | login identity for the hosted control plane |
| IngestToken | Postgres | revocable push credential (never grants read access) |
| MonitoredServer | Postgres | one fibre server instance reporting to a tenant |
| MetricSeriesCatalog | Postgres (config table) | canonical metric model + per-release mappings |
| UnknownSeriesRecord | Postgres | drift capture: unrecognized names/labels per server |
| AlertRule | Postgres | pack rule definition + tenant overrides |
| NotificationChannel | Postgres | Telegram/Discord/webhook/PagerDuty route + health |
| AlertIncident | Postgres | one firing lifecycle of one rule on one server |
| DeliveryRecord | Postgres | per-incident per-channel delivery attempts |
| FleetAggregate | Postgres | anonymized cross-tenant percentile bands (k≥5) |
| RegistrySnapshot | Postgres | periodic on-chain fibre-provider registry snapshot |
| RejectedPushEvent | Postgres | tenant-visible log of rejected ingest attempts |

---

## Tenant

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| slug | text unique | `^[a-z0-9][a-z0-9-]{2,40}$`; used in URLs |
| display_name | text | 1–80 chars |
| kind | enum(`hosted`,`selfhost_contributor`) | `selfhost_contributor` rows exist only when a self-host install opts in (FR-026) |
| status | enum(`active`,`deletion_pending`,`deleted`) | deletion transition below |
| created_at / updated_at | timestamptz | |
| deletion_requested_at | timestamptz null | set on tenant-initiated deletion (FR-031) |
| fleet_opt_in | bool default false | contribution to fleet aggregates; self-host default off |

Relationships: has many AccountUsers, IngestTokens, MonitoredServers, AlertRules, NotificationChannels, AlertIncidents, RejectedPushEvents.

**State transitions**: `active` → `deletion_pending` (tenant requests deletion; ingestion stops, tokens revoked, purge job scheduled) → `deleted` (all tenant data purged ≤ 30 days: raw + downsampled series, incidents, tokens, config — FR-031). Past contributions to already-published FleetAggregates are not retracted (documented on data-handling page).

## AccountUser

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK → Tenant | |
| email | citext unique | RFC 5322; verification required before first token issue |
| password_hash | text | argon2id; email+password is the MVP auth (SSO post-MVP) |
| role | enum(`owner`,`member`) | MVP: at least one `owner` per tenant; fine-grained RBAC post-MVP |
| created_at / last_login_at | timestamptz | |

## IngestToken

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK → Tenant | |
| label | text | 1–60 chars, operator-chosen (e.g. "mainnet-validator-1") |
| prefix | text | first 8 chars of the token, for UI identification (`fs_ingest_ab12cd34…`) |
| token_hash | bytea | SHA-256 of the full token; **plaintext shown once at creation**, never stored |
| issued_at / last_used_at | timestamptz | `last_used_at` updated by the edge (batched) |
| revoked_at | timestamptz null | revocation effective on subsequent pushes (FR-002); edge cache TTL ≤ 60 s |

Validation: tokens are 32-byte random, URL-safe base64, prefixed `fs_ingest_`; one tenant may hold many active tokens (rotation = issue new → switch flag → revoke old). A token authenticates pushes only — no read access (spec Key Entities).

## MonitoredServer

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK → Tenant | |
| instance_id | text | self-reported `service.instance.id` (hostname) — identity key within the tenant |
| reported_version | text null | self-reported `service.version` (e.g. `v10.4.0`) |
| support_classification | enum(`full`,`partial`,`unknown`) | from manifest registry (research R6); FR-010 |
| connection_status | enum(`connected`,`silent`,`conflict`) | transitions below |
| first_seen_at / last_seen_at | timestamptz | `last_seen_at` drives staleness display (FR-015) |
| network | enum(`mainnet`,`mocha`,`devnet`,`unknown`) default `unknown` | inferred from config/operator input |
| validator_consensus_address | text null | optional correlation to RegistrySnapshot |
| agent_connected | bool default false | true once agent-path series (disk, privval) observed; gates FR-014(g)/rule 7 |
| conflict_with_id | uuid FK → MonitoredServer null | set when two servers report the same `instance_id` |

Uniqueness: (`tenant_id`, `instance_id`) unique **except** during conflict state — a second distinct source reporting an already-seen `instance_id` creates a `conflict` row pair the tenant must resolve (edge case: cloned configs must surface, never silently merge).

**State transitions**: (new) → `connected` (first accepted point; US1 requires display ≤ 1 min) → `silent` (no points for 3 consecutive export intervals; triggers telemetry-silence rule) → `connected` (data resumes). `connected` → `conflict` (identity collision) → tenant resolves (rename/retire) → `connected`.

## MetricSeriesCatalog

Config table (not per-point data) — the canonical model FR-008; rows seeded from adapter manifests.

| Field | Type | Notes |
|---|---|---|
| canonical_name | text PK | e.g. `fibre.server.upload_shard.duration` (v10.4.0 names are canonical baseline) |
| namespace | enum(`server`,`client`,`runtime`,`unknown`) | handling policy per research R5 |
| instrument | enum(`counter`,`updown`,`histogram`,`gauge`,`observable_gauge`) | |
| unit | text | `s`, `By`, or empty |
| attribute_schema | jsonb | attribute names, types, enum values, conditionality (e.g. `upload_size` only when > 0) |
| resolution_tiers | jsonb | raw (30d) + downsampled steps (13mo) |
| alert_scope | bool | server-namespace canonical series only in MVP |
| mappings | jsonb | per-release source names → canonical (rename entries give continuous history, US4.3) |

Metric **points** live in VictoriaMetrics with labels `{tenant_id, server_id, namespace, …original attributes}`; retention: raw 30 d, downsampled 13 mo (FR-030); downsampled tier serves queries beyond the raw window transparently (FR-016).

## UnknownSeriesRecord

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| tenant_id / server_id | uuid FKs | |
| metric_name | text | unrecognized name or unexpected label set |
| first_seen_at / last_seen_at / sample_count | | for drift review; feeds SC-006 surfacing ≤ 24 h |

Validation: capture must never block or error ingestion of recognized series (FR-011).

## AlertRule

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK null | null = pack default row (the 11 rules, research R10); tenant rows override per `pack_key` |
| pack_key | text | stable rule identifier, e.g. `budget_utilization`, `sign_latency_p50` |
| name / description | text | description includes data-source caveats (e.g. sign p50 bucket edge at 10 ms) |
| condition_expr | text | MetricsQL against canonical series |
| thresholds | jsonb | per-rule parameters (e.g. `{"utilization": 0.8}`, `{"p50_ms": 10}`) |
| severity | enum(`info`,`warning`,`critical`) | pack defaults shipped |
| enabled | bool | FR-019: disable/enable per rule; pack works zero-tuned |
| agent_path_required | bool default false | rule 7 (disk headroom) — shown unavailable until `agent_connected` (FR-022) |
| version_gate | text null | e.g. rule 9 (`dupe_hits`) requires a version exposing the series; auto-disabled otherwise |
| channel_ids | uuid[] | per-rule channel assignment (FR-018) |
| repeat_interval | interval default `4h` | re-notify throttle while firing (FR-020) |

## NotificationChannel

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK → Tenant | |
| type | enum(`telegram`,`discord`,`webhook`,`pagerduty`) | FR-018 minimum set |
| config | jsonb (encrypted at rest) | per contract `alert-channels.md`: bot token+chat id / webhook URL / URL+HMAC secret / routing key |
| health | enum(`ok`,`failing`,`unknown`) | transitions below; FR-021 |
| last_error | text null | sanitized (no secrets) |
| last_delivery_at | timestamptz null | |

**State transitions**: `unknown` → `ok` (first successful delivery or explicit test) → `failing` (3 failed attempts with backoff) → `ok` (next success). `failing` raises an in-app channel-health notice so alerting gaps are never silent (US3.3).

## AlertIncident

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| rule_id / tenant_id / server_id | uuid FKs | |
| state | enum(`firing`,`resolved`) | |
| fired_at / resolved_at | timestamptz | resolved_at null while firing |
| triggering_values | jsonb | the values that breached (e.g. `{"utilization": 0.83}`) — shown in history (US3.4) |
| next_repeat_at | timestamptz null | repeat throttling (FR-020) |

**State transitions**: (condition true) → `firing` (notify assigned channels) → repeat-notify per `repeat_interval` while true → `resolved` (condition false; resolution notice always sent — SC-010 100%).

## DeliveryRecord

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| incident_id / channel_id | uuid FKs | |
| kind | enum(`firing`,`repeat`,`resolved`,`test`) | |
| status | enum(`success`,`failed`) | |
| attempts | int | ≤ 3 with backoff before channel marked failing |
| response_detail | text null | sanitized transport response |
| attempted_at | timestamptz | |

## FleetAggregate

| Field | Type | Validation / notes |
|---|---|---|
| id | uuid PK | |
| canonical_name | text | metric aggregated |
| cohort_key | text | e.g. `mainnet` or `mainnet:v10.4.x` |
| period_date | date | nightly computation date |
| p10 / p50 / p90 | double | percentile band values |
| contributing_tenant_count | int | **must be ≥ 5 for any published row** (FR-028) |
| suppressed | bool | true + null percentiles when cohort < 5 — UI shows explanation instead of a value |
| computed_at | timestamptz | |

Validation: k≥5 enforced at write (suppressed marker instead of values) **and** re-checked at read (SC-007); no query may attribute a contribution to a specific tenant; self-host tenants counted only when `fleet_opt_in` (US6.4).

## RegistrySnapshot

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| network | enum(`mainnet`,`mocha`) | |
| taken_at | timestamptz | nightly cadence |
| providers | jsonb | `[{validator_consensus_address, host}]` from `GET /valaddr/v1/all-bonded-fibre-providers` |
| source_endpoint | text | the REST endpoint queried (auditability: chain-record vs observation separation) |

Purpose: correlation of MonitoredServers to registered validators, version-drift majority context, and future annotation work (bonus queries available when needed: `GET /valaddr/v1/fibre-provider-info/{validator_consensus_address}`, `POST /fibre/v1/validate-payment-promise`, `GET /fibre/v1/is-payment-processed/{promise_hash}` — annotation-only, never core ingestion).

## RejectedPushEvent

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| tenant_id | uuid FK null | null when token unknown (cannot attribute) |
| token_prefix | text | identifies which token failed |
| reason | enum(`missing_token`,`unknown_token`,`revoked_token`,`rate_limited`,`payload_rejected`,`skew_rejected`) | |
| received_at | timestamptz | retained 30 days; powers FR-004 debugging view |

---

## Relationship diagram (text)

```text
Tenant 1─* AccountUser
Tenant 1─* IngestToken
Tenant 1─* MonitoredServer 1─* UnknownSeriesRecord
   │              │
   │              └─* AlertIncident *─1 AlertRule *─ Tenant (pack defaults: tenant_id null)
   │                            │
Tenant 1─* NotificationChannel 1─* DeliveryRecord *─1 AlertIncident
Tenant 1─* RejectedPushEvent
FleetAggregate  (no FK to Tenant — anonymity boundary)
RegistrySnapshot (global per network)
MetricSeriesCatalog (global config; points in VictoriaMetrics labeled tenant_id/server_id)
```

## Retention & purge rules (from FRs)

- Raw metric points: 30 days; downsampled: 13 months (automatic, hosted and self-host) — FR-030, SC-009.
- RejectedPushEvent: 30 days. DeliveryRecord/AlertIncident: life of tenant.
- Tenant deletion: all of the above + tokens + series purged ≤ 30 days (FR-031); published FleetAggregates immutable (documented).
