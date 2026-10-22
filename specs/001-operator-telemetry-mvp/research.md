# Phase 0 — Research & Decisions: Operator-Side Telemetry MVP

**Date**: 2026-10-21 | **Feature**: `001-operator-telemetry-mvp` | **Sources**: AGENTS.md §3–§5 (incl. §3 errata), `docs/protocol-verification-v10.4.0.md` (ground truth where they conflict), spec.md FRs, sibling celestia workspace (read-only reference).

All Technical Context items are resolved; no NEEDS CLARIFICATION remains. R1 (backend ingest language) was **decided by the owner on 2026-10-21: Node.js/TypeScript**.

---

## R1. Backend ingest language

**Decision**: **Node.js 22 LTS + TypeScript strict** — the ingest edge, versioned metric adapter, drift detector, status writer, and fleet-aggregation worker are one TS service (`fibrestats/services/ingest`, package `@fibrestats/ingest`) in the same pnpm workspace as the web app. HTTP edge on **Fastify 5**; OTLP/HTTP protobuf decode/encode via **`@opentelemetry/otlp-transformer`** (the official OTel JS transformer, underpinned by protobufjs + the `opentelemetry-proto` message definitions); Postgres access via a shared strict-typed data layer (Drizzle ORM, `fibrestats/packages/db`); tests via vitest. *(Owner decision 2026-10-21.)*

**Rationale**:
- Owner preference: a single-language codebase across edge, mapping, fleet worker, and web app — one toolchain (pnpm + tsc strict + vitest + eslint), shared types between ingest and control plane, unified hiring/maintenance.
- Constitution-compliant by construction: Principles IV (TypeScript strict for all app code) and V (pnpm only) hold with zero exceptions.
- AGENTS.md §0 explicitly accepts this path: "the OTel Collector handles the protocol anyway; the adapter is just a mapping service." The collector remains downstream for batching/export resilience; our service owns auth, mapping, classification — plain JSON-shaped transformations once the OTLP payload is decoded.
- `@opentelemetry/otlp-transformer` deserializes OTLP `ExportMetricsServiceRequest` protobuf into plain JS structures; the golden-fixture contract tests (FR-013) assert mapping behavior against recorded real payloads — fixture realism does not depend on the implementation language.

**Alternatives considered**:
- **Go standalone service** (this plan's earlier recommendation): the OTLP/OTel ecosystem and all Celestia code are Go-native, and Go contract tests would mirror the upstream structs the fibre binary serializes. **Rejected by the owner (2026-10-21) in favor of the single-language TypeScript codebase.** Retained as the documented fallback if TS OTLP decoding proves inadequate during contract-test implementation (R3/R15 exit criteria).
- **No custom service — OTel Collector transform processor only**: cannot do dynamic per-tenant token validation against a database, version classification with full/partial/unknown semantics, unknown-series capture for drift review, or fleet aggregation. Rejected.

## R2. OTLP edge authentication mechanism

**Decision**: The TS ingest service terminates OTLP/HTTP at the edge. Tokens travel **in the URL path** — `--otel-endpoint https://otel.<domain>/t/<ingest-token>` — because the fibre binary has no header/auth configuration and only appends `/v1/metrics` and `/v1/traces` to the base URL (path prefix preserved; verified at v10.4.0). The edge validates the token against Postgres (cached, with revocation propagation ≤ 60 s), injects `tenant_id` as a resource attribute, enforces metrics-only (traces accepted → discarded per FR-005), applies per-tenant rate/cardinality limits, and forwards to the OTel Collector over localhost OTLP.

**Rationale**: dynamic token lifecycle (issue/rotate/revoke, FR-002) with per-tenant attribution (FR-003) and rejection visibility (FR-004) requires database-backed auth at the ingestion point; embedding the credential in the endpoint URL is the *only* mechanism the unmodified binary supports, and it matches how operators already think about the flag (one value to paste).

**Alternatives considered**:
- *Collector `bearertokenauth` with per-tenant tokens provisioned into config* (AGENTS.md §5 table): works for static/small tenant sets and is retained as the **self-host simple mode** (compose ships a generated config); hosted multi-tenant operation would need config reloads or restarts on every token event and gives no rejection log. Rejected as the hosted edge, kept downstream.
- *Reverse proxy (nginx) auth subrequest → header injection*: viable but splits the auth contract across two components and still needs a service to answer subrequests; the ingest service already speaks HTTP. Rejected as redundant.

## R3. Versioned metric adapter architecture

**Decision**: The ingest service maintains **per-release manifests** (`fibrestats/services/ingest/manifests/v10.4.0.yaml`, `main-drift.yaml`, …) describing every known instrument — name, OTel instrument kind, unit, attribute schema, canonical mapping — and maps incoming series onto a **canonical model** that dashboards and alerts reference exclusively (FR-008). Canonical names equal the v10.4.0 dotted names for the baseline; future releases map onto the same canonical names via rename entries, yielding continuous history across upgrades (US4 scenario 3). Unknown names/labels are passed through untouched to storage, recorded per tenant/server, and surfaced for contract-drift review (FR-011) — they never block the pipeline.

**Rationale**: the metric surface is explicitly unstable across releases (drift between v10.2.0 docs, v10.4.0, and `main` already observed); a manifest-driven mapper makes onboarding a new release a data change (one YAML + one golden fixture) rather than a code change, satisfying SC-005 (new release supported ≤ 7 days).

**Alternatives considered**:
- *Hard-coded mapping in service code*: simplest for one version, fails FR-012/extensibility and SC-005. Rejected.
- *VictoriaMetrics stream relabeling*: can rename but cannot classify versions, cannot capture unknowns to Postgres, cannot run golden-fixture contract tests. Rejected.
- *OTel transform processor (OTTL)*: same limits as relabeling for classification/capture; kept for trivial batching only. Rejected as the mapping home.

## R4. v10.4.0 baseline content (verification corrections applied)

**Decision**: The v10.4.0 manifest contains **18 instruments**, exactly per `docs/protocol-verification-v10.4.0.md` §3 appendix — including `fibre.server.upload_shard.last_success_timestamp` and `fibre.server.backend.get.{duration,in_flight,bytes}` (both **ship in v10.4.0**; AGENTS.md originally mislabeled them "new on main", corrected via its §3 errata). The only genuinely main-only items are `fibre.server.upload_shard.dupe_hits` (Counter, attr `stage`) and the `fibre.server.store.backend` info gauge — both carried in `manifests/main-drift.yaml` so the pipeline understands them the day they ship. `occupancy_bytes` semantics recorded precisely: limiter-tracked bytes = on-disk + in-flight reserved (not a disk gauge; host disk remains agent-path data).

**Rationale**: the verification doc is code-level ground truth at the deployed tag; AGENTS.md §3 conflicts are resolved in its favor (46 confirmed / 5 DIFFERS).

**Alternatives considered**: mapping only what the v10.4.0 README table lists — the README omits `rejected`, `occupancy_bytes`, `budget_bytes` (upstream doc gap); code is ground truth. Rejected.

## R5. OTLP namespace handling policy

**Decision**: One fibre endpoint emits **three namespaces**; the ingest service defines explicit policy for each:
- `fibre.server.*` → mapped to canonical model; drives dashboards, alerts, fleet aggregates. Full scope.
- `fibre.client.*` (same binary in client mode, e.g. `fibre.client.upload.duration`) → **ingested and stored** under the tenant, labeled `namespace="client"`; out of MVP dashboard/alert scope (spec: client-side metrics are an explicit MVP exclusion); available for post-MVP use.
- `go.*` (Go runtime instrumentation starts whenever OTLP is enabled) → **ingested and stored**, labeled `namespace="runtime"`; powers a runtime dashboard (derived from official `fibre-runtime.json`); out of MVP alert scope.
- Anything else → stored as unknown, recorded for drift review, flagged.

**Rationale**: dropping client/runtime series would lose free operational signal and complicate future scope expansion; alerting on them now would violate the spec's MVP boundary. Explicit policy prevents "unexpected data" confusion during onboarding.

**Alternatives considered**: discard non-server namespaces at the edge — rejected (data loss without savings; VM handles the volume trivially).

## R6. Version detection & support classification

**Decision**: Detect the server version from the OTLP resource attribute `service.version` (the binary sets `service.name=fibre`, `service.version=<release>`, `service.instance.id=<hostname>` on every export). Classification per server: **full** = version string exactly matches a manifest with 100% baseline coverage; **partial** = version matches a manifest marked drifted/deprecated, or a known version with unmapped additions; **unknown** = never-seen version string — recognized series still ingest and render, the server is flagged, and the operator is notified (US4 scenario 2, SC-006 ≤ 24 h surfacing). Missing/garbled version → unknown.

**Rationale**: exact-string matching against a manifest registry is deterministic and testable; classification drives the prominent support-state badge (FR-010) and the version-drift alert rule.

**Alternatives considered**: semver-range matching — upstream uses release tags and the metric contract is per-tag; ranges imply untested coverage claims. Rejected.

## R7. Time-series storage

**Decision**: **VictoriaMetrics single-node** for MVP. Tenant isolation via the injected `tenant_id` label plus `vmauth`/per-tenant Grafana datasource scoping; retention `-retentionPeriod=30d` raw with VM **downsampling** rules (e.g. 5 m step after 30 d) retained 13 months; queries transparently span tiers (FR-016, FR-030). Identical settings ship in the self-host compose/Helm defaults.

**Rationale**: single-binary operational simplicity (self-host parity matters), native downsampling, low resource footprint at MVP scale (~10² servers × low-hundreds of series — trivial load), multi-tenant labels sufficient without enterprise features.

**Alternatives considered**:
- *Grafana Mimir*: horizontally scalable but operationally heavy (object store, multiple services); unjustified at MVP scale — complexity budget better spent on the adapter. Rejected for MVP.
- *Prometheus + Thanos*: two systems to reach the same downsampling/retention story; VM is simpler. Rejected.
- *Managed TSDB (Grafana Cloud etc.)*: breaks self-host parity and data-control story. Rejected.

## R8. Entity store

**Decision**: **Postgres 16** for tenants, users, ingest tokens (hashed), monitored servers, alert rules, incidents, notification channels, delivery records, fleet aggregates, registry snapshots, rejected-push events. Same engine hosted and self-host (FR-025 parity). Access goes through a shared strict-typed TypeScript data layer (**Drizzle ORM**, workspace package `fibrestats/packages/db`) used by both the Next.js route handlers and the ingest service — one schema definition, one migration source, no `any` escapes (constitution IV).

**Rationale**: relational integrity fits the entity graph (tenant → servers/rules/incidents), mature tooling, runs everywhere compose/Helm run; a shared TS data layer is the payoff of the single-language decision (R1).

**Alternatives considered**: SQLite for self-host slimness — rejected (parity + concurrency for edge token cache misses and status writes). MySQL — no advantage; ecosystem convention and JSONB needs favor Postgres.

## R9. Dashboards

**Decision**: **Hosted Grafana with per-tenant organizations** for MVP metric visualization. Dashboard JSON is **derived from the official upstream dashboards at `observability/docker/grafana/dashboards/fibre.json` ("Fibre Network", uid `fibre-network`) and `fibre-runtime.json` ("Fibre Go Runtime")** — the `fibre/dashboards/fibre-dashboards.json` path does not exist at v10.4.0 or main (upstream README link is broken; confirmed in the verification doc and AGENTS.md §3 errata). Derivations: re-point panels at canonical series, inject `tenant_id` datasource scoping, re-theme to the brand palette (no Celestia logos), and extend with the seven FR-014 dashboard groups (at-a-glance, uploads incl. rejected-by-reason and dupe-hits, downloads, storage & budget from the exported gauges, signing, connections, host & disk [agent-path]). The Next.js app links into the tenant's Grafana org; deep iframe embedding is post-MVP.

**Rationale**: Grafana is the mandated stack component, ships alerting/contact points we reuse (R10), and the upstream dashboards are a maintained semantic starting point; per-tenant orgs give hard isolation (FR-007, SC-008).

**Alternatives considered**:
- *Custom panels in Next.js querying VM*: full brand control but re-implements a dashboarding product; retained as a post-MVP evolution for the at-a-glance view only. Rejected for MVP.
- *One shared Grafana org with label filtering*: weaker isolation story. Rejected.

## R10. Alerting engine & alert pack defaults

**Decision**: **Grafana Alerting** (unified) evaluates the 11-rule pack per tenant against canonical series; contact points deliver to Telegram, Discord, generic webhook, PagerDuty (FR-018). The ingest service's status data (Postgres) backs rules Grafana cannot see (telemetry silence via series-absence on the edge's heartbeat marker; channel-health surfacing). Incident lifecycle (firing → repeat-throttled → resolved) is mirrored to Postgres for the web app's incident history (FR-020).

**Opinionated defaults** (FR-017/FR-019; tuned from verified operational facts; all per-rule overridable):
1. Budget utilization > 80% — `occupancy_bytes / budget_bytes` (exported gauges only, **never stake math**; the real budget is ≈3× liveness-scaled and clamped).
2. `budget_exceeded` rejections > 0 over 15 m (only `reason` value at v10.4.0).
3. Occupancy growth-rate anomaly — derivative of `occupancy_bytes` vs 7-day baseline.
4. Prune lag — `prune.entries` stalls or `prune.duration` p90 breach across expected 1-minute prune intervals (the ~1m45s empirical lag is **not** baked in as a constant).
5. `sign.duration` p50 > 10 ms over 15 m (SLO; p50 sits on the 0.01 s bucket edge — note bucket-resolution caveat in the rule description).
6. `last_success_timestamp` stale > 2× expected upload interval during active windows (gauge absent before first success — handled as "no uploads yet", not stale).
7. Free disk < budget headroom — agent-path rule, marked unavailable until agent data exists (FR-022).
8. Connection saturation / download starvation — `upload_shard.in_flight` ≥ connection-limit-derived threshold sustained (default limit 16; one upload can occupy all slots).
9. Dupe-hit spike — activates only for versions exposing `dupe_hits` (main-drift); auto-disabled on v10.4.0.
10. Version drift — server's `service.version` ≠ network-majority version (majority from RegistrySnapshot + reported versions).
11. Telemetry silence — no points from a previously-reporting server for 3 consecutive export intervals (distinct from metric thresholds; US3 scenario 5).

**Rationale**: dashboards and alerts in one system keeps per-tenant routing coherent (per-rule channel assignment, FR-018/019) and reuses Grafana's built-in Telegram/Discord/webhook/PagerDuty integrations.

**Alternatives considered**: *VMAlert + Alertmanager* — more Prometheus-native but a second alerting stack to operate and no built-in per-tenant org routing. Rejected for MVP; the pack's rule expressions are written in MetricsQL-compatible form so a later migration is mechanical.

## R11. Notification delivery & channel health

**Decision**: Delivery through Grafana contact points; every delivery attempt is recorded (channel, incident, status, response, attempts) and surfaced as **channel health** in the web app (FR-021): ok / failing(last error) / unknown. Failures: 3 attempts with backoff, then health=failing + an in-app notice. Throttle: repeat notification every 4 h while firing (default, per-rule overridable); resolution always sent (SC-010: 100%).

**Alternatives considered**: custom dispatcher service — duplicates Grafana's maintained integrations; rejected for MVP, delivery-record mirroring gives us the visibility without owning the transports.

## R12. Agent path

**Decision**: Publish a ready-to-use **Grafana Alloy** configuration (FR-023) that scrapes node-exporter (host disk — the fibre server has no disk gauge) and the consensus node's `cometbft_privval_signing_latency_*` (KMS SLO: median ≤ 10 ms), attaches the tenant token, and pushes into the same ingest edge. An equivalent plain OTel Collector config is documented as a drop-in alternative. Direct push alone remains fully functional minus the explicitly agent-gated view/rule (FR-024).

**Alternatives considered**: a custom agent binary — unjustified build/maintenance cost for MVP; rejected. VM agent — would add a third agent syntax for no gain; rejected.

## R13. Fleet aggregation

**Decision**: A nightly job — the ingest service's `aggregate` command (Node CLI in the same package; cron/systemd timer in self-host, platform scheduler hosted) — computes percentile bands (p10/p50/p90) per canonical health metric per cohort (network × server-version-majority cohorts) across all opted-in tenants, writing `FleetAggregate` rows. **k-anonymity ≥ 5 enforced twice**: cohorts with < 5 distinct contributing tenants are never written as values (a suppressed marker row is written for UI explanation, FR-028), and the read API re-checks the threshold before returning (SC-007). Self-host tenants contribute only with explicit opt-in (FR-026); their rows are excluded otherwise. Published aggregates are immutable history — a departing tenant's past contributions are not retracted (documented on the data-handling page).

**Alternatives considered**: continuous/streaming aggregation — premature; nightly matches the decision cadence ("am I normal?") and simplifies k-suppression. Rejected.

## R14. Self-host packaging

**Decision**: Two artifacts from one component set: **docker-compose** (ingest service, otelcol, VictoriaMetrics, Postgres, Grafana, web app) as the primary path, and a **Helm chart** (`deploy/helm/fibrestats`) whose values mirror the compose env surface. The `ingest` container image is built from the pnpm workspace (Node 22 LTS runtime) and exposes three commands: `serve` (edge + mapping), `aggregate` (nightly job), `migrate` (Postgres schema). Defaults: no outbound traffic (FR-026), retention 30d/13mo preconfigured, dashboards + alert pack provisioned, static bearer-token collector mode documented for small installs (R2). Opt-in fleet contribution: `FIBRESTATS_FLEET_OPT_IN=true` enables a publisher that POSTs only the anonymized aggregate payload to the hosted endpoint; the exact payload is inspectable via a local endpoint/dump before anything sends (US5 scenario 3). Upgrade path: pull new images → run `ingest migrate` → new manifests activate without tenant action (FR-012, US5 scenario 4).

**Alternatives considered**: Helm-only — excludes the majority docker-only operator; compose-only — excludes k8s infra providers. Both shipped.

## R15. Web-app data path & dev loop

**Decision (web app)**: Next.js route handlers (server-side, session-authenticated) over Postgres — via the shared `packages/db` data layer — implement `contracts/web-api.md`. The web app **never queries VictoriaMetrics directly in MVP**: metric visualization lives in Grafana orgs; server status/version/classification/last-seen flow from the ingest service into Postgres, which the app reads. Fleet benchmarks read `FleetAggregate` rows only. This keeps every app read path trivially tenant-scoped (SC-008).

**Decision (dev loop)**: Local celestia-app devnet via `scripts/single-node-fibre.sh` (or Huginn's `plsgiveup/fibre` `fibre-devnet` for multi-validator) generates real `fibre.server.*` traffic without mainnet (Go is required only to build/run the devnet binary itself — not part of our codebase). First devnet session **records a golden OTLP fixture** (one captured `ExportMetricsServiceRequest` per release) which becomes the ingest-service contract test (FR-013, SC-005), run by vitest against `@opentelemetry/otlp-transformer` decoding. Dogfooding then moves to mocha, then our own mainnet node; design partners (3–5 validators) recruited from the mocha v10 cohort.

**Alternatives considered**: synthetic metric generators for development — rejected as primary (fixture realism is the whole point of FR-013; synthetics only supplement edge cases). Web app embedding VM queries through a tenant proxy — deferred post-MVP (adds an authz layer for marginal UX gain).

---

## Resolved Technical Context summary

| Item | Resolution |
|---|---|
| Backend language | Node.js 22 LTS + TypeScript strict, same pnpm workspace as the web app (owner decision 2026-10-21; Go retained as documented fallback, R1) |
| Edge auth | Token-in-path, terminated by the TS ingest service (R2) |
| Adapter | Manifest-driven mapper + vitest golden-fixture contract tests (R3, R4) |
| Namespaces | server → canonical; client/go → ingested, out of MVP alert scope (R5) |
| Classification | exact `service.version` match → full/partial/unknown (R6) |
| TSDB | VictoriaMetrics, 30d raw + downsampled 13mo (R7) |
| Entity store | Postgres 16 via shared Drizzle data layer (R8) |
| Dashboards | Grafana per-tenant orgs, derived from official `observability/docker/grafana/dashboards/fibre*.json` (R9) |
| Alerting | Grafana Alerting, 11-rule pack with verified defaults (R10, R11) |
| Agent | Published Alloy config (R12) |
| Fleet | nightly percentiles, k≥5 enforced at write and read (R13) |
| Packaging | compose + Helm, opt-in off by default (R14) |
| Dev loop | devnet + golden fixture; web app reads Postgres only (R15) |
