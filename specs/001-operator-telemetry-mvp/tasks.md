# Tasks: Operator-Side Telemetry MVP for Celestia Fibre Servers

**Input**: Design documents from `/specs/001-operator-telemetry-mvp/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/ (otlp-ingest, adapter-mapping-v10.4.0, web-api, alert-channels, self-host-packaging), quickstart.md

**Tests**: Tests are REQUESTED — the spec/plan mandate golden-fixture adapter contract tests (FR-013/SC-005), tenant-isolation tests (SC-008), alert/channel lifecycle tests, and k≥5 fleet-suppression tests (SC-007). Write test tasks FIRST in each story phase and confirm they FAIL before implementing.

**Organization**: Tasks are grouped by user story (6 stories, priority order P1, P1, P2, P2, P3, P3) so each story is independently implementable, testable, and deliverable as an increment.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: US1–US6, required on user-story phase tasks only (never on Setup, Foundational, or Polish tasks)
- Every description names the exact file path(s) it produces

## Path Conventions

- **Web app (Next.js 16 control plane)**: `fibrestats/app/` (App Router pages + route handlers), `fibrestats/components/` (shadcn/ui), `fibrestats/lib/` (session auth, domain logic), app tests in `fibrestats/test/`
- **Ingest service** (`@fibrestats/ingest`, Node 22 LTS + TS strict): `fibrestats/services/ingest/src/{edge,mapping,status,fleetagg}/`, manifests in `fibrestats/services/ingest/manifests/`, tests in `fibrestats/services/ingest/test/`, golden fixtures in `fibrestats/services/ingest/test/fixtures/golden/`
- **Shared data layer** (Drizzle ORM, Postgres 16): `fibrestats/packages/db/src/`
- **Infra/packaging**: `deploy/` at repo root (docker-compose, otelcol, VictoriaMetrics, Grafana provisioning, Alloy agent, Helm)
- One pnpm workspace rooted at `fibrestats/` (`pnpm-workspace.yaml`: app + `packages/*` + `services/*`); pnpm only; TypeScript `strict: true` + `noUncheckedIndexedAccess` everywhere; Celestia palette/typography tokens only — **no Celestia logos/marks anywhere** (UI, Grafana JSON, alert templates, docs)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: pnpm workspace initialization, service skeletons, and local infra so the dev loop (quickstart.md steps 0–1) can run. Implements research decisions R1, R7, R14 (dev-loop half), R15.

- [ ] T001 Convert `fibrestats/` into a pnpm workspace: create `fibrestats/pnpm-workspace.yaml` (`packages: ["packages/*", "services/*"]` plus the app at root) and update `fibrestats/package.json` with workspace scripts (`build`, `lint`, `typecheck` → `tsc --noEmit`, `test` → `vitest run`); keep `packageManager: "pnpm@9.15.0"` and the single `pnpm-lock.yaml`
- [ ] T002 [P] Scaffold the ingest service package `@fibrestats/ingest` in `fibrestats/services/ingest/package.json` (dependencies: `fastify@^5`, `@opentelemetry/otlp-transformer`, `protobufjs`, `drizzle-orm`, `pg`; devDeps: `typescript@^5`, `vitest`, `@types/node`) with `fibrestats/services/ingest/tsconfig.json` extending the strict base and `src/` subdirs `edge/`, `mapping/`, `status/`, `fleetagg/` per plan Project Structure
- [ ] T003 [P] Scaffold the shared data layer package `@fibrestats/db` in `fibrestats/packages/db/package.json` (deps `drizzle-orm`, `pg`; devDep `drizzle-kit`), `fibrestats/packages/db/tsconfig.json`, `fibrestats/packages/db/src/client.ts` (Postgres 16 connection from env), and `fibrestats/packages/db/drizzle.config.ts`
- [ ] T004 [P] Create the shared strict TS base config `fibrestats/tsconfig.base.json` (`strict: true`, `noUncheckedIndexedAccess: true` — constitution Principle IV) and extend the existing `fibrestats/eslint.config.mjs` to cover `packages/` and `services/`; no `any` escapes without inline justification, no `@ts-ignore`
- [ ] T005 [P] Add the web-app health endpoint `fibrestats/app/api/healthz/route.ts` returning `{ "status": "ok" }` (quickstart expects `:3000/api/healthz`)
- [ ] T006 Create `deploy/docker-compose.yml` (services: `adapter` image `fibrestats/adapter:<version>`, `otelcol` pinned `otel/opentelemetry-collector-contrib`, `victoriametrics` pinned `victoria-metrics/victoria-metrics`, `postgres:16`, `grafana` pinned, `web`) with named volumes `vm-data`, `pg-data`, `grafana-data`, ports adapter `:4318` / grafana `:3001` / web `:3000`, VM+Postgres internal-only, healthchecks gating `depends_on`; and `deploy/.env.example` containing exactly the contract env surface: `FIBRESTATS_DOMAIN` (default `localhost`), `FIBRESTATS_ADMIN_EMAIL`, `FIBRESTATS_RETENTION_RAW_DAYS` (default `30`), `FIBRESTATS_RETENTION_DOWNSAMPLED_MONTHS` (default `13`), `FIBRESTATS_FLEET_OPT_IN` (default `false`), `FIBRESTATS_FLEET_ENDPOINT`, `FIBRESTATS_STATIC_TOKENS`, `FIBRESTATS_ALERT_REPEAT_INTERVAL` (default `4h`) per contracts/self-host-packaging.md §2
- [ ] T007 [P] Create the OTel Collector pipeline config `deploy/otelcol/config.yaml`: OTLP receiver on localhost (accepts from the ingest service only) → batch processor → VictoriaMetrics exporter; include a `bearertokenauth` extension wired to `FIBRESTATS_STATIC_TOKENS` for the self-host static-token simple mode (research R2)
- [ ] T008 [P] Create Grafana datasource provisioning `deploy/grafana/provisioning/datasources/victoriametrics.yaml` pointing at the internal VM service, plus the provisioning directory layout `deploy/grafana/provisioning/{dashboards,alerting}/` per contracts/self-host-packaging.md §2
- [ ] T009 Configure VictoriaMetrics retention/downsampling in `deploy/docker-compose.yml` (depends on T006): raw `-retentionPeriod=30d` from `FIBRESTATS_RETENTION_RAW_DAYS` and downsampling rules to 13 months from `FIBRESTATS_RETENTION_DOWNSAMPLED_MONTHS` (FR-030, research R7) — identical settings must hold in hosted and self-host
- [ ] T010 [P] Create `fibrestats/services/ingest/Dockerfile`: node:22 base, pnpm workspace-aware build producing the `fibrestats/adapter` image whose entrypoint exposes the three commands `serve` (edge + mapping), `aggregate` (nightly job), `migrate` (Postgres schema) per contracts/self-host-packaging.md §1
- [ ] T011 [P] Write the devnet runbook `deploy/docs/devnet.md`: clone celestia-app at tag `v10.4.0`, run `scripts/single-node-fibre.sh` (Go 1.24+ required for the devnet binary only — never part of our codebase), expected fibre config at `~/.celestia-fibre/config/server_config.toml`, OTLP off by default; note Huginn's `fibre-devnet` (`github.com/plsgiveup/fibre`) as the multi-validator alternative (quickstart steps 0+3, research R15)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Database schema for all 13 entities, the token-in-path OTLP edge, and the v10.4.0 manifest-driven adapter. Implements research R2–R6, R8 and contracts otlp-ingest.md + adapter-mapping-v10.4.0.md.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T012 [P] Define Drizzle schema for Tenant + AccountUser in `fibrestats/packages/db/src/schema/tenants.ts`: Tenant `slug` text unique matching `^[a-z0-9][a-z0-9-]{2,40}$`, `display_name` 1–80 chars, `kind` enum(`hosted`,`selfhost_contributor`), `status` enum(`active`,`deletion_pending`,`deleted`), `fleet_opt_in` bool default false, `deletion_requested_at` timestamptz null; AccountUser `email` citext unique (RFC 5322), `password_hash` argon2id, `role` enum(`owner`,`member`); `id` UUIDv7 PKs, all `*_at` timestamptz per data-model.md
- [ ] T013 [P] Define Drizzle schema for IngestToken + MonitoredServer in `fibrestats/packages/db/src/schema/tokens.ts` and `fibrestats/packages/db/src/schema/servers.ts`: IngestToken `label` 1–60 chars, `prefix` = first 8 chars of the token, `token_hash` bytea (SHA-256; plaintext never stored), `issued_at`/`last_used_at`, `revoked_at` null (revocation effective on subsequent pushes, edge cache TTL ≤ 60 s); MonitoredServer `instance_id` text (self-reported `service.instance.id`, identity key within tenant), `reported_version` null, `support_classification` enum(`full`,`partial`,`unknown`), `connection_status` enum(`connected`,`silent`,`conflict`), `network` enum(`mainnet`,`mocha`,`devnet`,`unknown`) default `unknown`, `agent_connected` bool default false, `conflict_with_id` self-FK null, unique (`tenant_id`,`instance_id`) except during conflict pairs per data-model.md
- [ ] T014 [P] Define Drizzle schema for MetricSeriesCatalog + UnknownSeriesRecord + RejectedPushEvent in `fibrestats/packages/db/src/schema/catalog.ts`, `fibrestats/packages/db/src/schema/unknown-series.ts`, `fibrestats/packages/db/src/schema/rejected-pushes.ts`: catalog `canonical_name` text PK, `namespace` enum(`server`,`client`,`runtime`,`unknown`), `instrument` enum(`counter`,`updown`,`histogram`,`gauge`,`observable_gauge`), `attribute_schema`/`resolution_tiers`/`mappings` jsonb, `alert_scope` bool; RejectedPushEvent `tenant_id` null when token unknown, `token_prefix`, `reason` enum(`missing_token`,`unknown_token`,`revoked_token`,`rate_limited`,`payload_rejected`,`skew_rejected`), retained 30 days
- [ ] T015 [P] Define Drizzle schema for AlertRule + NotificationChannel + AlertIncident + DeliveryRecord in `fibrestats/packages/db/src/schema/alert-rules.ts`, `fibrestats/packages/db/src/schema/channels.ts`, `fibrestats/packages/db/src/schema/incidents.ts`, `fibrestats/packages/db/src/schema/deliveries.ts`: AlertRule `tenant_id` null = pack default row, `pack_key` stable identifier (e.g. `budget_utilization`, `sign_latency_p50`), `condition_expr` MetricsQL, `thresholds` jsonb (e.g. `{"utilization": 0.8}`, `{"p50_ms": 10}`), `severity` enum(`info`,`warning`,`critical`), `agent_path_required` bool default false, `version_gate` text null, `channel_ids` uuid[], `repeat_interval` interval default `4h`; NotificationChannel `type` enum(`telegram`,`discord`,`webhook`,`pagerduty`), `config` jsonb encrypted at rest, `health` enum(`ok`,`failing`,`unknown`), `last_error` sanitized; AlertIncident `state` enum(`firing`,`resolved`), `triggering_values` jsonb; DeliveryRecord `kind` enum(`firing`,`repeat`,`resolved`,`test`), `status` enum(`success`,`failed`), `attempts` ≤ 3
- [ ] T016 [P] Define Drizzle schema for FleetAggregate + RegistrySnapshot in `fibrestats/packages/db/src/schema/fleet-aggregates.ts` and `fibrestats/packages/db/src/schema/registry.ts`: FleetAggregate `p10`/`p50`/`p90` double, `contributing_tenant_count` int (**must be ≥ 5 for any published row**), `suppressed` bool (true + null percentiles when cohort < 5), **no FK to Tenant** (anonymity boundary); RegistrySnapshot `network` enum(`mainnet`,`mocha`), `providers` jsonb `[{validator_consensus_address, host}]`, `source_endpoint` text
- [ ] T017 Generate the initial migration with drizzle-kit into `fibrestats/packages/db/migrations/` and implement the idempotent migrate runner `fibrestats/packages/db/src/migrate.ts` (safe to re-run, used by the `adapter migrate` command and the Helm pre-upgrade hook) — depends on T012–T016
- [ ] T018 Implement the ingest CLI entry `fibrestats/services/ingest/src/cli.ts` dispatching `serve` / `aggregate` / `migrate` (migrate delegates to packages/db runner from T017) — the three commands the `fibrestats/adapter` image exposes
- [ ] T019 Implement the Fastify 5 edge server `fibrestats/services/ingest/src/edge/server.ts`: routes `POST /t/:token/v1/metrics`, `POST /t/:token/v1/traces`, `GET /healthz` (port 4318), raw-body protobuf handling, response codes exactly per contracts/otlp-ingest.md §4 (`400 malformed_payload`, `401 unauthorized` with reason, `413 payload_too_large`, `415 unsupported_media_type`, `429 rate_limited` with `retry_after_seconds`, 5xx retryable)
- [ ] T020 [P] Implement token-in-path authentication `fibrestats/services/ingest/src/edge/auth.ts` (depends on T019): extract `fs_ingest_` + 32-byte URL-safe base64 token from the path segment (bearer header `Authorization: Bearer fs_ingest_…` also accepted for the agent path), exact-match lookup by SHA-256 hash against packages/db with an edge cache where revocation propagates ≤ 60 s, write a `RejectedPushEvent` (`missing_token`/`unknown_token`/`revoked_token`, with token prefix, `tenant_id` null when unattributable) on every rejection — per contracts/otlp-ingest.md §2–§4
- [ ] T021 [P] Implement OTLP decode + metrics-only enforcement `fibrestats/services/ingest/src/edge/otlp.ts` (depends on T019): decode `ExportMetricsServiceRequest` via `@opentelemetry/otlp-transformer`; traces route responds 200 and discards (FR-005); non-protobuf content-type → 415; malformed protobuf → 400; body > 8 MiB → 413; logs/blob payloads/key material are never decoded, stored, or displayed
- [ ] T022 [P] Implement the per-tenant rate/cardinality guard `fibrestats/services/ingest/src/edge/ratelimit.ts` (depends on T019, T020): request body ≤ 8 MiB, ≤ 5,000 active series per server, burst ≤ 30 requests/min with 2/min sustained; breach → 429 + quarantine flag visible to the affected tenant only — other tenants' ingestion unaffected (SC-004)
- [ ] T023 [P] Implement the timestamp/dedup policy `fibrestats/services/ingest/src/edge/timestamps.ts` (depends on T019): points in `[now−7d, now+10m]` accepted as-is; future > +10 min clamped to receipt time and flagged (`skew_rejected`/flag surface); older than −7 d dropped and flagged; OTLP cumulative temporality makes retries idempotent — identical (series, timestamp) points are last-write-wins so duplicate pushes never double-count (contracts/otlp-ingest.md §3.5–3.6)
- [ ] T024 Author the v10.4.0 adapter manifest `fibrestats/services/ingest/manifests/v10.4.0.yaml` with `release: "v10.4.0"`, `support: full`, and exactly the 18 normative instruments from contracts/adapter-mapping-v10.4.0.md §2 (verbatim names/kinds/units/attributes/buckets): `upload_shard.in_flight` (updown), `upload_shard.duration` (histogram s, attrs `success` bool required + `upload_size` int only when > 0, buckets 0.001…5), `upload_shard.bytes` (counter By), `upload_shard.rejected` (counter, `reason` — only value at tag `budget_exceeded`), `upload_shard.last_success_timestamp` (observable_gauge s, **absent before first success**), `upload_shard.occupancy_bytes` + `upload_shard.budget_bytes` (observable_gauge By — budget gauge is the only authoritative budget source, stake math forbidden), `download_shard.{in_flight,duration,bytes}` (`shard_size` attr), `store.{put,get}.duration` (histogram s, buckets 0.001…1), `backend.get.{duration,in_flight,bytes}` (attrs `backend` = `local`|`object`, `outcome` = `success`|`not_found`|`timeout`|`canceled`|`throttled`|`error`; duration buckets 0.001…60), `sign.duration` (histogram s, buckets 0.0001…1 — p50 SLO 10 ms sits exactly on the 0.01 bucket edge), `prune.entries` (counter, only when `pruned > 0`), `prune.duration` (histogram s); plus the `namespaces:` block (`fibre.server.` → canonical, `fibre.client.` → passthrough, `go.` → passthrough) and `unknown: { policy: record_and_flag }` per contract §1/§3
- [ ] T025 Implement the manifest loader + mapper `fibrestats/services/ingest/src/mapping/manifest.ts` and `fibrestats/services/ingest/src/mapping/mapper.ts` (depends on T024): load all YAML manifests into a registry, map incoming series to canonical names/kinds/units (canonical = v10.4.0 dotted names for baseline; rename entries yield continuous history across releases), inject `namespace` label (`server`/`client`/`runtime`), validate attribute schemas (names, types, enum values, conditionality e.g. `upload_size` only when > 0)
- [ ] T026 [P] Implement version detection + support classification `fibrestats/services/ingest/src/mapping/classify.ts` (depends on T025): exact `service.version` string match against the manifest registry → `full` (100% baseline coverage manifest) / `partial` (drifted/deprecated manifest or known version with unmapped additions) / `unknown` (never-seen or missing/garbled version) per research R6 and contract §1
- [ ] T027 [P] Implement unknown-series capture `fibrestats/services/ingest/src/mapping/unknown.ts` (depends on T025): unrecognized names/labels pass through untouched to storage, upsert `UnknownSeriesRecord` (metric_name, first_seen_at/last_seen_at/sample_count per tenant+server) and never block or error ingestion of recognized series (FR-011)
- [ ] T028 Implement the edge→collector forwarder `fibrestats/services/ingest/src/edge/forward.ts` (depends on T020–T027): inject resource attributes `tenant_id=<uuid>` + resolved `server_id` (from `service.instance.id`; `service.name=fibre` expected, other values accepted under the agent path with namespace labeling), then batch to the OTel Collector over localhost OTLP with labels `{tenant_id, server_id, namespace, …original attributes}` per contracts/otlp-ingest.md §3.4/§3.7
- [ ] T029 Implement the status writer + catalog seed `fibrestats/services/ingest/src/status/writer.ts` and `fibrestats/services/ingest/src/mapping/seed-catalog.ts` (depends on T025, T026): upsert MonitoredServer on accepted points (`first_seen_at`/`last_seen_at`, `reported_version`, `support_classification`, `connection_status: connected` on first point), create conflict row pairs when a second distinct source reports an already-seen `instance_id` (never silently merge), write the heartbeat marker series used by telemetry-silence detection, and seed MetricSeriesCatalog rows from manifests at migrate/first boot (contracts/self-host-packaging.md §6)

**Checkpoint**: `docker compose up -d` healthy; a devnet fibre server pointed at `http://localhost:4318/t/<token>` flows series into VictoriaMetrics with `tenant_id`/`server_id`/`namespace` labels — quickstart steps 1–5 smoke-pass. Foundation ready: user story implementation can now begin in parallel.

---

## Phase 3: User Story 1 — Operator connects a fibre server and sees telemetry flowing (Priority: P1) 🎯 MVP

**Goal**: Zero-install onboarding: account creation → self-service ingest token (issue/rotate/revoke) → one startup flag on the fibre server → "connected / receiving" status with version recognition within 1 minute; rejected pushes visible for debugging; multiple servers per tenant distinct.

**Independent Test** (quickstart.md steps 1–4): sign up → create token labeled `devnet-1` → restart devnet fibre with `--otel-endpoint http://localhost:4318/t/fs_ingest_<token>` → within 60 s server shows `connected` with `reported_version = v10.4.0`, `support_classification = full`; restart with a garbage token → 401s appear in the tenant's ingestion-status view.

### Tests for User Story 1 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T030 [P] [US1] Write the edge-auth contract test `fibrestats/services/ingest/test/edge_auth.test.ts`: valid token → 200; missing/unknown/revoked → 401 with the exact reason enum; revoked token effective on the next push after cache TTL ≤ 60 s; every rejection writes a RejectedPushEvent with the token prefix; one tenant's rejections never affect another tenant's pushes
- [ ] T031 [P] [US1] Write the tenant-isolation API test `fibrestats/test/api/isolation.test.ts` (SC-008): seed two tenants; assert `GET /api/tokens`, `GET /api/servers`, `GET /api/tokens/rejections` return only the session tenant's rows and cross-tenant ids are indistinguishable from 404 (403/404 existence-leak protection per contracts/web-api.md status codes)
- [ ] T032 [P] [US1] Write the token-lifecycle test `fibrestats/test/api/tokens.test.ts`: `POST /api/tokens` returns plaintext once and never again; `GET /api/tokens` lists `{ id, label, prefix, issued_at, last_used_at, revoked_at }` with no secrets; rotation flow (issue new → switch flag → revoke old) keeps ingestion working

### Implementation for User Story 1

- [ ] T033 [P] [US1] Implement session auth `fibrestats/lib/auth/session.ts`: signup/login/logout with argon2id password hashing, HttpOnly Secure session cookie, CSRF token for mutations, email verification required before first token issuance (data-model AccountUser; MVP email+password, SSO post-MVP)
- [ ] T034 [P] [US1] Implement auth route handlers `fibrestats/app/api/auth/signup/route.ts` (creates Tenant + owner AccountUser; derives and validates `slug` against `^[a-z0-9][a-z0-9-]{2,40}$`), `fibrestats/app/api/auth/login/route.ts`, `fibrestats/app/api/auth/logout/route.ts` per contracts/web-api.md §Auth
- [ ] T035 [US1] Implement the token service `fibrestats/lib/tokens.ts`: generate `fs_ingest_` + 32-byte random URL-safe base64, persist only SHA-256 `token_hash` + first-8-char `prefix`, enforce label 1–60 chars; plaintext returned exactly once at creation
- [ ] T036 [US1] Implement token route handlers `fibrestats/app/api/tokens/route.ts` (POST issue / GET list without secrets) and `fibrestats/app/api/tokens/[id]/revoke/route.ts` (sets `revoked_at`; effective on subsequent pushes ≤ 60 s) — FR-002
- [ ] T037 [US1] Implement the rejected-push debugging endpoint `fibrestats/app/api/tokens/rejections/route.ts` returning recent RejectedPushEvents (token prefix, reason, received_at; 30-day retention) — FR-004
- [ ] T038 [US1] Implement the ingest guide endpoint `fibrestats/app/api/ingest/guide/route.ts` rendering the ready-to-paste flag block `--otel-endpoint https://otel.<domain>/t/<token>` plus the agent-path snippet (contracts/web-api.md §Ingest tokens)
- [ ] T039 [US1] Implement server list/detail endpoints `fibrestats/app/api/servers/route.ts` and `fibrestats/app/api/servers/[id]/route.ts`: `connection_status`, `reported_version`, `support_classification`, `last_seen_at`, `agent_connected`, derived `stale` flag, conflict info, unknown-series count (FR-003/006/010/015)
- [ ] T040 [US1] Implement identity-conflict resolution `fibrestats/app/api/servers/[id]/resolve-conflict/route.ts` accepting `{ action: "keep_first" | "keep_new" | "rename", new_instance_id? }` and returning 409 while a conflict is unresolved (edge case: cloned configs surface, never merge)
- [ ] T041 [P] [US1] Build the auth pages `fibrestats/app/(auth)/signup/page.tsx` and `fibrestats/app/(auth)/login/page.tsx` with shadcn/ui forms on brand tokens (Frost `#FDFCFF` / Void `#0E1014`, Indigo `#5640D1` primary; no Celestia marks)
- [ ] T042 [P] [US1] Build the onboarding pages `fibrestats/app/(app)/servers/page.tsx` ("Add server" flow) and `fibrestats/app/(app)/tokens/page.tsx` (copy-once token display with the exact paste-ready `--otel-endpoint` flag, list with prefix/issued_at, revoke action) — the SC-001 under-10-minutes flow
- [ ] T043 [US1] Build the server detail page `fibrestats/app/(app)/servers/[id]/page.tsx`: prominent support-classification badge (`full` / `partial` / `unknown`), reported version, connection status with staleness, recent rejected pushes from T037
- [ ] T044 [US1] Run the US1 validation per `specs/001-operator-telemetry-mvp/quickstart.md` steps 1–4 and record results: connected ≤ 1 min, version `v10.4.0` classified `full` (SC-001 flow), garbage-token negative check rejected and visible, two servers on one token appear as distinct MonitoredServers (US1.4)

**Checkpoint**: User Story 1 fully functional and independently testable — an operator goes from account creation to first visible metric in under 10 minutes with only a flag change.

---

## Phase 4: User Story 2 — Operator monitors their server's health on per-tenant dashboards (Priority: P1)

**Goal**: Private per-tenant Grafana dashboards covering the 7 FR-014 groups (at-a-glance, uploads incl. rejected-by-reason + dupe-hits, downloads, storage & budget, signing, connections, host & disk agent-gated) plus the runtime dashboard; arbitrary time ranges across raw/downsampled tiers; stale data visually marked; zero cross-tenant visibility.

**Independent Test** (quickstart.md steps 5+8): drive blob upload/download traffic on the devnet, confirm every applicable series of the 18-instrument baseline checklist lands with `tenant_id`/`server_id`/`namespace` labels, each dashboard group renders expected values for the correct tenant, host-and-disk honestly shows "requires the agent path", and a second tenant sees nothing of the first (FR-007/SC-008).

### Tests for User Story 2 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T045 [P] [US2] Write the dashboards API test `fibrestats/test/api/dashboards.test.ts`: `GET /api/dashboards` returns only the session tenant's org links for the 8 groups (`at_a_glance, uploads, downloads, storage_budget, signing, connections, runtime, host_disk`); `host_disk` returns `available: false` with explanation until `agent_connected` (US2.4)
- [ ] T046 [P] [US2] Write the provisioning validation test `fibrestats/test/infra/provisioning.test.ts`: every provisioned dashboard JSON parses, every panel query carries the `tenant_id` label selector (no panel can ever render another tenant's data), and no Celestia logo/mark asset is referenced anywhere in `deploy/grafana/provisioning/`

### Implementation for User Story 2

- [ ] T047 [P] [US2] Derive the at-a-glance, uploads, and downloads dashboards `deploy/grafana/provisioning/dashboards/at-a-glance.json`, `uploads.json`, `downloads.json` from upstream `observability/docker/grafana/dashboards/fibre.json` (uid `fibre-network`): re-point every panel at canonical series (upload/download activity + latency, `upload_shard.rejected` by `reason`, dupe-hits panel version-aware), inject `tenant_id` scoping, re-theme to the brand palette only (research R9)
- [ ] T048 [P] [US2] Derive the storage & budget dashboard `deploy/grafana/provisioning/dashboards/storage-budget.json`: utilization computed **only** from the exported `occupancy_bytes` / `budget_bytes` gauges (stake-share math forbidden — real budget ≈3× liveness-scaled, clamped), plus prune-loop panels (`prune.entries` — absence ≠ dead loop; `prune.duration`)
- [ ] T049 [P] [US2] Derive the signing and connections dashboards `deploy/grafana/provisioning/dashboards/signing.json` and `deploy/grafana/provisioning/dashboards/connections.json`: `sign.duration` p50 panel annotated with the 10 ms SLO and the bucket-edge resolution caveat; `upload_shard.in_flight` saturation panel against the connection-limit threshold (default limit 16 — one upload can occupy all slots)
- [ ] T050 [P] [US2] Derive the runtime dashboard `deploy/grafana/provisioning/dashboards/runtime.json` from upstream `fibre-runtime.json` scoped to the `go.*` namespace series (`namespace="runtime"`; no MVP alerts)
- [ ] T051 [P] [US2] Create the host & disk dashboard `deploy/grafana/provisioning/dashboards/host-disk.json` for the agent path (node-exporter disk + `cometbft_privval_signing_latency_*`) with the built-in "requires the agent path" notice mechanism instead of empty/misleading panels (FR-014(g), FR-022)
- [ ] T052 [US2] Implement per-tenant Grafana org management `fibrestats/lib/grafana/orgs.ts` (hosted path): create an isolated org per tenant with datasource scoping on the injected `tenant_id` label, and provision the derived dashboards into it; self-host uses the single seeded `selfhost` org (research R9; FR-007 isolation)
- [ ] T053 [US2] Implement `fibrestats/app/api/dashboards/route.ts`: per-group links into the tenant's Grafana org; `host_disk` returns `available: false` + explanation until the server's `agent_connected` flag is true (US2.4, FR-022); the web app never queries VictoriaMetrics directly (research R15)
- [ ] T054 [US2] Build the dashboards page `fibrestats/app/(app)/dashboards/page.tsx`: group cards linking into Grafana, agent-path explainer empty state for host & disk, and a note that time-range selection spans raw (30 d) and downsampled (13 mo) tiers transparently (FR-016)
- [ ] T055 [US2] Implement staleness marking (FR-015) in the provisioned dashboard JSON under `deploy/grafana/provisioning/dashboards/`: threshold annotations/opacity so series with no recent points render visually distinct from live data, wired to the server's `last_seen_at`/stale derivation — never show last-value-as-current
- [ ] T056 [US2] Verify the retention tier contract (FR-016/FR-030, SC-009): inspect VM flags from T009 in `deploy/docker-compose.yml` (`-retentionPeriod=30d` + downsampling to 13 mo) and document/query-check that ranges beyond the raw window are served at downsampled resolution with no error and no fabricated resolution
- [ ] T057 [US2] Run the US2 validation per `specs/001-operator-telemetry-mvp/quickstart.md` steps 5+8: all 18 baseline instruments land (activity-dependent ones driven by devnet blob traffic; `last_success_timestamp` absent before first success is expected, not a bug), zero false drift flags for v10.4.0, dashboards live-update, two-tenant isolation check passes

**Checkpoint**: User Stories 1 AND 2 both work independently — complete self-serve visibility into one's own fibre server (joint MVP core).

---

## Phase 5: User Story 3 — Operator receives actionable alerts on their own channels (Priority: P2)

**Goal**: The opinionated 11-rule alert pack with zero-tuning defaults, per-rule enable/disable/threshold overrides, routing to Telegram/Discord/webhook/PagerDuty, full incident lifecycle (firing → 4 h repeat throttle → resolution always sent), and channel-health surfacing so alerting gaps are never silent.

**Independent Test** (quickstart.md step 7): stop the devnet fibre server → telemetry-silence incident fires within SC-003's 5-minute budget and a distinct "telemetry stopped" payload reaches a local webhook request bin; restart → resolution notice; point the channel at a dead URL → after 3 attempts the channel shows `health=failing` with a sanitized error; disable a rule → it stays silent while the rest of the pack works.

### Tests for User Story 3 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T058 [P] [US3] Write the alert-pack validation test `fibrestats/test/infra/alertpack.test.ts`: all 11 provisioned rule expressions parse as MetricsQL, reference only canonical series present in the seeded MetricSeriesCatalog, and rules 7/9 carry their `agent_path_required`/`version_gate` metadata (FR-022)
- [ ] T059 [P] [US3] Write the channel delivery/health test `fibrestats/test/api/channels.test.ts`: delivery retries ≤ 3 with backoff 30 s / 2 min / 10 min; after 3 failures `health=failing` with sanitized `last_error` (no secrets); any success (incl. `POST /api/channels/{id}/test`) resets `health=ok`; secrets are write-only and never rendered after save
- [ ] T060 [P] [US3] Write the incident lifecycle test `fibrestats/test/api/incidents.test.ts`: firing notification on breach → repeat ≤ 1 per `repeat_interval` (default `4h`) while firing → resolution notification on 100% of resolved incidents (SC-010); history exposes fired_at/resolved_at/`triggering_values`; disabled rules produce no incidents (US3.2)

### Implementation for User Story 3

- [ ] T061 [US3] Author the alert pack provisioning `deploy/grafana/provisioning/alerting/pack-rules.yaml` with all 11 FR-017 rules at research-R10 defaults: (1) budget utilization > 80% via `occupancy_bytes / budget_bytes` gauges only; (2) `budget_exceeded` rejections > 0 over 15 m; (3) occupancy growth-rate anomaly vs 7-day baseline; (4) prune lag (`prune.entries` stall or `prune.duration` p90 breach across 1-minute prune intervals); (5) `sign.duration` p50 > 10 ms over 15 m with the bucket-edge caveat in the description; (6) `last_success_timestamp` stale > 2× expected upload interval (gauge absent before first success = "no uploads yet", NOT stale); (7) free disk < budget headroom — agent-path only, unavailable until agent data exists; (8) connection saturation — `upload_shard.in_flight` ≥ limit-derived threshold sustained (default 16); (9) dupe-hit spike — auto-disabled unless the version exposes `dupe_hits`; (10) version drift vs network majority; (11) telemetry silence — no points for 3 consecutive export intervals
- [ ] T062 [US3] Seed the pack into Postgres `fibrestats/packages/db/src/seed/alert-pack.ts`: 11 `tenant_id`-null default rows with `pack_key`, MetricsQL `condition_expr`, `thresholds` jsonb verbatim (`{"utilization": 0.8}`, `{"p50_ms": 10}`, …), `severity`, `repeat_interval` default `4h`, `agent_path_required` on rule 7, `version_gate` on rule 9 — wired into the first-boot seed (contracts/self-host-packaging.md §6)
- [ ] T063 [US3] Implement alert rules endpoints `fibrestats/app/api/alert-rules/route.ts` (effective rules = pack defaults overlaid with tenant overrides; each rule carries `pack_key, name, description, severity, enabled, thresholds, agent_path_required, version_gate, channel_ids, available` with `available:false` + reason when agent/version-gated) and `fibrestats/app/api/alert-rules/[pack_key]/route.ts` (PATCH upserts the tenant override: `enabled?`, `thresholds?`, `channel_ids?`, `repeat_interval?`; pack defaults remain fallback) — FR-018/019/022
- [ ] T064 [US3] Implement notification channel endpoints `fibrestats/app/api/channels/route.ts` (POST create / GET list with redacted config + `health`/`last_error`/`last_delivery_at`), `fibrestats/app/api/channels/[id]/route.ts` (PATCH/DELETE), and `fibrestats/app/api/channels/[id]/test/route.ts` (sends a `kind:"test"` notification, returns outcome, updates health) for the four types: telegram (`bot_token` validated via `getMe` on save, `chat_id`, `parse_mode` default `MarkdownV2`), discord (`webhook_url`), webhook (https url; optional `secret` enabling `X-FibreStats-Signature: sha256=<hmac>`), pagerduty (`routing_key`, optional `severity_map`) — secrets write-only, encrypted at rest (contracts/alert-channels.md §2)
- [ ] T065 [US3] Author contact-point + message templates `deploy/grafana/provisioning/alerting/contact-points.yaml` implementing the common payload of contracts/alert-channels.md §1 verbatim (`incident_id`, `kind`, `rule{pack_key,name,severity}`, `server{instance_id,reported_version,network}`, `fired_at`, `resolved_at`, `triggering_values`, `summary`, `links`): Discord embed colors from the brand palette (Indigo `#5640D1` firing, Cool Gray resolved — **no Celestia logos/marks** in avatars or embeds), webhook body = common payload verbatim with `X-FibreStats-Attempt: n`, PagerDuty Events API v2 `trigger`/`resolve` sharing `dedup_key` = `incident_id`
- [ ] T066 [US3] Implement delivery-record mirroring + channel health `fibrestats/services/ingest/src/status/deliveries.ts`: every delivery attempt recorded as a DeliveryRecord (kind, status, attempts ≤ 3 with backoff 30 s/2 min/10 min, sanitized `response_detail`); health transitions `unknown` → `ok` (first success or test) → `failing` (3 failed attempts) → `ok` (next success); `failing` raises the in-app channel-health notice (FR-021, US3.3)
- [ ] T067 [US3] Implement incident endpoints `fibrestats/app/api/incidents/route.ts` (`?state=&server_id=` filters, firing/resolved history with `triggering_values` and fired/resolved timestamps) and `fibrestats/app/api/incidents/[id]/route.ts` (detail + per-channel DeliveryRecords) — FR-020, US3.4
- [ ] T068 [P] [US3] Build the alert rules page `fibrestats/app/(app)/alerts/rules/page.tsx`: per-rule enable/disable switch, threshold override form, per-rule channel assignment, availability badges explaining agent-gated (rule 7) and version-gated (rule 9) rules (FR-022)
- [ ] T069 [P] [US3] Build the channels page `fibrestats/app/(app)/alerts/channels/page.tsx` (health badges `ok`/`failing`/`unknown` with sanitized last error, test-delivery button) and the incidents page `fibrestats/app/(app)/alerts/incidents/page.tsx` (timeline with triggering values, per-channel delivery outcomes)
- [ ] T070 [US3] Wire telemetry-silence detection end to end (rule 11): evaluate absence of the edge heartbeat marker series (T029) for 3 consecutive export intervals and emit the visually distinct "telemetry stopped" notification template in `deploy/grafana/provisioning/alerting/` (`"telemetry stopped" ≠ "metric breached"` per contracts/alert-channels.md §4)
- [ ] T071 [US3] Run the US3 validation per `specs/001-operator-telemetry-mvp/quickstart.md` step 7 (7a silence + resolution, 7b optional `budget_exceeded` rejection alert with count in `triggering_values`, 7c channel-failure surfacing): alert delivered ≤ 5 min from condition (SC-003), resolution always delivered, disabled rule silent

**Checkpoint**: A notification-only operator gets the full protective value of the alert pack with zero dashboard usage.

---

## Phase 6: User Story 4 — Operator upgrades their fibre server without telemetry gaps (Priority: P2)

**Goal**: Upgrade-safe monitoring: per-release manifests keep dashboards/alerts on canonical series across upstream upgrades; unknown future versions are explicitly flagged (recognized series still ingested) instead of panels quietly going empty; version-drift rule compares against network majority.

**Independent Test**: replay golden fixtures from two different releases against one tenant and verify identical canonical-series semantics on dashboards and alerts (US4.1/US4.3); replay a fabricated unknown-version fixture and verify classification `unknown` + operator-visible unsupported-version state with recognized series still ingested (US4.2); contract `pnpm --filter @fibrestats/ingest exec vitest run golden_v10_4_0` green at 100% coverage (FR-013/SC-005).

### Tests for User Story 4 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T072 [P] [US4] Record the golden fixture from a devnet v10.4.0 session into `fibrestats/services/ingest/test/fixtures/golden/v10.4.0.otlp.pb` (one full `ExportMetricsServiceRequest` protobuf, captured via ingest debug flag or tcpdump-to-protobuf — quickstart step 6) and write the golden contract test `fibrestats/services/ingest/test/golden_v10_4_0.test.ts` asserting per contracts/adapter-mapping-v10.4.0.md §5: every manifest canonical instrument appears in the fixture and maps to its declared canonical name/kind/unit; every `fibre.server.*` fixture series is covered by the manifest (coverage = 100%, failures name the uncovered series); attribute schemas match (names, types, enum values, conditionality)
- [ ] T073 [P] [US4] Write the unknown-version test `fibrestats/services/ingest/test/unknown_version.test.ts`: a fixture with a fabricated future `service.version` yields classification `unknown`, recognized series still ingest and render, unknown series produce `UnknownSeriesRecord` entries (not errors), and the drift event is surfaced for SC-006's ≤ 24 h operator notification
- [ ] T074 [P] [US4] Write the cross-release replay test `fibrestats/services/ingest/test/cross_release.test.ts`: replaying two different releases' fixtures against one tenant produces identical canonical-series semantics (rename entries give one continuous history where semantically identical — US4.1/US4.3)

### Implementation for User Story 4

- [ ] T075 [US4] Author the pre-staged drift manifest `fibrestats/services/ingest/manifests/main-drift.yaml` (`support: partial`) covering the two main-only instruments verbatim from contracts/adapter-mapping-v10.4.0.md §4: `fibre.server.upload_shard.dupe_hits` (counter, attr `stage` — activates alert rule 9 only for versions exposing it) and `fibre.server.store.backend` (observable_gauge const 1, attrs `backend`, `object_configured` bool — info gauge, no MVP alerts)
- [ ] T076 [US4] Implement drift surfacing: unknown-series list + unsupported-version banner on `fibrestats/app/(app)/servers/[id]/page.tsx`, the unknown-series count field on `GET /api/servers/{id}` (T039), and the operator notification path that guarantees 100% of unknown-version/contract-drift events surface ≤ 24 h with zero silent gaps (SC-006, FR-011)
- [ ] T077 [US4] Implement the registry snapshot job `fibrestats/services/ingest/src/status/registry.ts` (nightly fetch of `GET /valaddr/v1/all-bonded-fibre-providers` into RegistrySnapshot with `source_endpoint` recorded for auditability) plus `fibrestats/app/api/registry/route.ts`, and compute the network-majority version (RegistrySnapshot + reported versions) feeding alert rule 10 (US4.4)
- [ ] T078 [US4] Write the new-release onboarding runbook `fibrestats/services/ingest/manifests/README.md` following contract §6: record golden fixture on the new tag → diff vs existing manifests → generate/update `<release>.yaml` (renames map onto the same canonical names) → add contract test → mark drifted prior manifest `partial` → ship with zero tenant action (FR-012); target support ≤ 7 days after upstream publication (SC-005)
- [ ] T079 [US4] Run the US4 validation: full replay matrix of `specs/001-operator-telemetry-mvp/contracts/adapter-mapping-v10.4.0.md` §5 green (T072–T074), plus a devnet upgrade simulation per `specs/001-operator-telemetry-mvp/quickstart.md` showing dashboards/alerts keep working on canonical series with no operator action and no interpretation gap

**Checkpoint**: Monitoring survives upstream upgrades by construction; drift is always loud, never silent.

---

## Phase 7: User Story 5 — Operator self-hosts the entire stack (Priority: P3)

**Goal**: One-command packaged deployment (docker-compose primary, Helm for k8s) running the same component set as hosted; zero outbound traffic by default; explicit opt-in fleet contribution with an inspectable payload; documented retention and upgrade procedures.

**Independent Test** (quickstart.md step 1 in a clean environment): `cp .env.example .env && docker compose up -d` → all healthchecks pass, `selfhost` Grafana org has the 7 dashboard groups + 11-rule pack pending, a devnet fibre server connects and dashboards/alerts work end-to-end with no hosted-service account; network-policy inspection proves nothing leaves the environment; the contribution preview endpoint renders the exact JSON while the publisher stays disabled unless `FIBRESTATS_FLEET_OPT_IN=true`.

### Tests for User Story 5 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T080 [P] [US5] Write the compose contract test `fibrestats/test/infra/compose.test.ts`: `deploy/docker-compose.yml` parses; env surface covers every key of contracts/self-host-packaging.md §2 with the contract defaults (`FIBRESTATS_FLEET_OPT_IN=false`, retention 30/13); named volumes `vm-data`/`pg-data`/`grafana-data` present; VM/Postgres expose no host ports; no enabled outbound job exists by default (FR-026)
- [ ] T081 [P] [US5] Write the Helm template test `fibrestats/test/infra/helm.test.ts`: `helm template deploy/helm/fibrestats` renders one workload per §1 component, `values.yaml` mirrors every compose env key 1:1 (same names), PVCs render for the three stateful services, and the `adapter migrate` pre-upgrade hook is present

### Implementation for User Story 5

- [ ] T082 [US5] Complete the compose package in `deploy/docker-compose.yml`: healthchecks gating `depends_on` for every service (adapter `/healthz`, Grafana `/api/health`, web `/api/healthz` per contract §6), first-boot seed job running `adapter migrate` (idempotent) + pack rules + MetricSeriesCatalog + `selfhost` Grafana org creation, and the `web` service documented as omittable (tokens/rules then driven via config files)
- [ ] T083 [US5] Implement the static-token simple mode: generated `deploy/otelcol/config.yaml` `bearertokenauth` section from `FIBRESTATS_STATIC_TOKENS` with documentation in `deploy/docs/install.md` for small installs without the web app (research R2 fallback)
- [ ] T084 [US5] Implement the fleet contribution preview + opt-in publisher `fibrestats/services/ingest/src/fleetagg/preview.ts` and `fibrestats/services/ingest/src/fleetagg/publisher.ts`: `GET /internal/fleet-contribution/preview` renders exactly the JSON the publisher would send (contract §4 shape: `installation_id` random regeneratable uuid, `period_date`, `aggregates[]` with p10/p50/p90 + `contributing_server_count` — no raw series, no labels, no tenant/server identifiers); the publisher POSTs only when `FIBRESTATS_FLEET_OPT_IN=true`, to `FIBRESTATS_FLEET_ENDPOINT`; opt-in withdrawable anytime, past aggregates not retractable (documented)
- [ ] T085 [P] [US5] Create the Helm chart `deploy/helm/fibrestats/Chart.yaml`, `deploy/helm/fibrestats/values.yaml` (mirrors every compose env key 1:1), and `deploy/helm/fibrestats/templates/`: one Deployment/StatefulSet per component, PVCs for vm/pg/grafana, Ingress for adapter OTLP + Grafana + web with TLS notes, `adapter migrate` pre-upgrade hook, conservative resource requests sized for ~10 servers with documented scaling values (contract §3)
- [ ] T086 [P] [US5] Write the self-host docs `deploy/docs/install.md` (compose install guide), `deploy/docs/helm.md`, `deploy/docs/agent-path.md`, and the retention + upgrade sections (US5.4): how the 30 d/13 mo policy is applied, and `docker compose pull && up -d` / `helm upgrade` (new manifests activate with zero tenant action, FR-012); manifest releases published standalone so operators can verify support for a new celestia-app release before upgrading (SC-005)
- [ ] T087 [US5] Run the US5 validation in a clean environment per `specs/001-operator-telemetry-mvp/quickstart.md` step 1 and the Independent Test above: end-to-end dashboards + alerts with no hosted account, default no-egress proof, opt-in preview accurate with publisher disabled, enable opt-in → only anonymized aggregates published (US5.1–5.4)

**Checkpoint**: Full product under the operator's exclusive control — trust rule FR-025/026/033 satisfied.

---

## Phase 8: User Story 6 — Operator compares their server against anonymized fleet benchmarks (Priority: P3)

**Goal**: Nightly anonymized percentile bands (p10/p50/p90) for key health metrics per cohort; hard k≥5 suppression at write AND read; own-values comparison alongside the band; self-host tenants contribute only with explicit opt-in.

**Independent Test** (quickstart.md step 9): seed 4 tenants → `docker compose exec adapter adapter aggregate --now` → `GET /api/fleet/benchmarks` returns `{ suppressed: true, explanation }` — never a number, never the cohort size; seed a 5th → bands appear; own values render alongside; non-opt-in self-host data contributes nothing.

### Tests for User Story 6 ⚠️ (write FIRST, ensure they FAIL)

- [ ] T088 [P] [US6] Write the k≥5 suppression test `fibrestats/services/ingest/test/fleetagg_suppression.test.ts`: cohort of 4 distinct contributing tenants → suppressed marker row with null percentiles written (never values); cohort of 5 → published row; read path re-checks the threshold before returning (SC-007); suppression leaks neither a value nor the exact cohort size; published aggregates immutable when a tenant later leaves
- [ ] T089 [P] [US6] Write the fleet API test `fibrestats/test/api/fleet.test.ts`: `GET /api/fleet/benchmarks` returns only published bands or the suppressed explanation; `GET /api/fleet/my-comparison` returns the tenant's own values alongside the band with no other tenant's identity or values; tenants without `fleet_opt_in` are excluded from every aggregate (US6.4)

### Implementation for User Story 6

- [ ] T090 [US6] Implement the nightly aggregation job `fibrestats/services/ingest/src/fleetagg/aggregate.ts` (the `adapter aggregate` command, `--now` flag for manual runs): compute p10/p50/p90 per canonical health metric per `cohort_key` (network × server-version-majority) across opted-in tenants only, enforce k≥5 at write (suppressed marker + null percentiles for cohorts < 5 distinct tenants, `contributing_tenant_count` recorded), and never attribute a contribution to a specific tenant (research R13, FR-027/028)
- [ ] T091 [US6] Implement the fleet endpoints `fibrestats/app/api/fleet/benchmarks/route.ts` (published aggregates `{ canonical_name, cohort_key, period_date, p10, p50, p90, computed_at }[]`; suppressed cohorts return `{ suppressed: true, explanation }` — never a number, never the cohort size; read path re-checks k≥5 per SC-007) and `fibrestats/app/api/fleet/my-comparison/route.ts` (own server's values alongside the published band; no other tenant identifiable, ever — FR-029)
- [ ] T092 [P] [US6] Build the fleet benchmark page `fibrestats/app/(app)/fleet/page.tsx`: percentile bands vs the operator's own server per metric, the suppressed-with-explanation state (hidden entirely until ≥ 5 tenants contribute per spec assumption), and a plain-language anonymity note
- [ ] T093 [US6] Wire the nightly schedule: cron/systemd timer in self-host compose (`deploy/docker-compose.yml`) and documented hosted scheduler, running `adapter aggregate` after the RegistrySnapshot refresh (T077) so cohort keys stay current (FR-027 "at least nightly")
- [ ] T094 [US6] Run the US6 validation per `specs/001-operator-telemetry-mvp/quickstart.md` step 9: suppression demonstrably holds in both directions (≤ 4 → suppressed, ≥ 5 → published), own-comparison renders, self-host non-opt-in contributes nothing, opt-in tenants counted

**Checkpoint**: "Am I normal?" answered with provable anonymity — k≥5 enforced twice, trust rules demonstrably hold.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Data-lifecycle completeness, trust-page publication, constitution compliance, and full-suite validation across all stories.

- [ ] T095 [P] Implement tenant profile/lifecycle endpoints `fibrestats/app/api/tenant/route.ts`: GET (profile, status, `fleet_opt_in`), PATCH (`display_name`, `fleet_opt_in`), DELETE → enters `deletion_pending` with a response stating the ≤ 30-day purge window and aggregate irreversibility (FR-031); 410 on deletion-pending tenants per contracts/web-api.md status codes
- [ ] T096 [P] Implement the tenant purge job `fibrestats/services/ingest/src/status/purge.ts`: on `deletion_pending`, stop ingestion, revoke tokens, and purge all tenant data (raw + downsampled VM series with the tenant's `tenant_id` label, incidents, delivery records, tokens, config) within the documented ≤ 30-day window (FR-031); published FleetAggregates immutable (no FK — anonymity boundary holds)
- [ ] T097 [P] Publish the data-handling page `fibrestats/app/(app)/data-handling/page.tsx` backed by `fibrestats/app/api/data-handling/route.ts` (FR-032): metrics-only collection, never-collected list (keys, blob payloads), retention 30 d raw / 13 mo downsampled, tenant isolation, deletion window, k≥5 anonymity rules, single-region hosting statement
- [ ] T098 [P] Run the brand/constitution audit across `fibrestats/`, `deploy/grafana/provisioning/`, and `deploy/docs/`: no Celestia logos/marks anywhere (web UI, Grafana JSON, alert templates/avatars, docs — verified by T046's provisioning test extended repo-wide), palette/typography tokens only, shadcn/ui + Tailwind exclusively, no non-pnpm lockfiles, no `@ts-ignore`/`any` escapes without justification
- [ ] T099 Make the full quality gate green: `pnpm lint`, `tsc --noEmit`, and `pnpm test` (vitest across app + `@fibrestats/ingest` + `@fibrestats/db`) from the workspace root `fibrestats/`
- [ ] T100 Execute the complete `specs/001-operator-telemetry-mvp/quickstart.md` validation end-to-end and record the six-item summary: (1) zero-install onboarding ≤ 1 min connected, version classified `full`; (2) all 18 baseline instruments land with namespace policy honored; (3) golden fixture recorded, contract test green; (4) alert ≤ 5 min with resolution + channel-health surfacing; (5) dashboards tenant-isolated, staleness marked, agent-gated views honest; (6) k≥5 suppression holds in both directions
- [ ] T101 [P] Update run documentation `fibrestats/README.md` (workspace install/build/dev commands, devnet pointer to `deploy/docs/devnet.md`) and sync `AGENTS.md` if the documented repo layout changed (workspace packages, services/ingest, deploy/ contents)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion (T017 needs T012–T016; T019–T029 build on the service skeletons) — **BLOCKS all user stories**
- **User Stories (Phases 3–8)**: All depend on Foundational completion; then proceed in priority order or in parallel if staffed
- **Polish (Phase 9)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Foundational only — no story dependencies
- **US2 (P1)**: Foundational only; consumes canonical series from the Phase 2 adapter (not from US1 code) — independently testable by replaying streams; recommended after US1 because onboarding is the natural demo path
- **US3 (P2)**: Foundational only; uses Postgres status data + heartbeat marker from Phase 2 — independently testable by replaying breaching/silent streams; soft demo dependency on US1 (a connected server to alert on)
- **US4 (P2)**: Foundational adapter only — independently testable purely via fixture replay; no US1–US3 code required
- **US5 (P3)**: Packaging of Phase 1–2 components; verifying "same dashboards and alert pack" naturally follows US2/US3 but the compose/Helm artifacts and no-egress guarantees are independently testable
- **US6 (P3)**: Foundational schema + VM data; independently testable via seeded synthetic tenants — no US1–US5 code required

### Within Each User Story

- Tests (T030–T032, T045–T046, T058–T060, T072–T074, T080–T081, T088–T089) MUST be written and FAIL before implementation
- Schema/library tasks before route handlers; route handlers before pages
- Provisioning JSON before API routes that link to it (US2); pack rules + seed before rules API (US3)
- Story validation task (T044, T057, T071, T079, T087, T094) runs last in its phase

### Parallel Opportunities

- Phase 1: T002, T003, T004, T005, T007, T008, T010, T011 in parallel (T006 → T009 sequential; both before Phase 2)
- Phase 2: T012–T016 (schema files) in parallel; T020–T023 (edge modules) in parallel after T019; T026 ∥ T027 after T025
- After Foundational: US1 ∥ US2 can start simultaneously; US3 ∥ US4 ∥ US5 ∥ US6 with enough staffing
- Within stories: all [P] test tasks first in parallel; all [P] implementation tasks in parallel (e.g. US2's five dashboard derivations T047–T051; US3's two pages T068 ∥ T069)

---

## Parallel Execution Examples

### User Story 1

```bash
# Launch all US1 tests together (write-first, must fail):
Task: "Edge-auth contract test in fibrestats/services/ingest/test/edge_auth.test.ts"
Task: "Tenant-isolation API test in fibrestats/test/api/isolation.test.ts"
Task: "Token-lifecycle test in fibrestats/test/api/tokens.test.ts"

# Then parallel independent implementation:
Task: "Session auth lib in fibrestats/lib/auth/session.ts"
Task: "Auth route handlers in fibrestats/app/api/auth/{signup,login,logout}/route.ts"
Task: "Auth pages in fibrestats/app/(auth)/{signup,login}/page.tsx"
Task: "Onboarding pages in fibrestats/app/(app)/{servers,tokens}/page.tsx"
```

### User Story 2

```bash
# Tests first:
Task: "Dashboards API test in fibrestats/test/api/dashboards.test.ts"
Task: "Provisioning validation test in fibrestats/test/infra/provisioning.test.ts"

# Then all five dashboard derivations in parallel (independent JSON files):
Task: "at-a-glance + uploads + downloads dashboards in deploy/grafana/provisioning/dashboards/"
Task: "storage-budget dashboard in deploy/grafana/provisioning/dashboards/storage-budget.json"
Task: "signing + connections dashboards in deploy/grafana/provisioning/dashboards/"
Task: "runtime dashboard in deploy/grafana/provisioning/dashboards/runtime.json"
Task: "host-disk dashboard in deploy/grafana/provisioning/dashboards/host-disk.json"
```

### User Story 3

```bash
# Tests first:
Task: "Alert-pack validation test in fibrestats/test/infra/alertpack.test.ts"
Task: "Channel delivery/health test in fibrestats/test/api/channels.test.ts"
Task: "Incident lifecycle test in fibrestats/test/api/incidents.test.ts"

# Then pages in parallel after APIs land:
Task: "Alert rules page in fibrestats/app/(app)/alerts/rules/page.tsx"
Task: "Channels + incidents pages in fibrestats/app/(app)/alerts/{channels,incidents}/page.tsx"
```

### User Story 4

```bash
# All three fixture-replay tests in parallel:
Task: "Golden contract test in fibrestats/services/ingest/test/golden_v10_4_0.test.ts"
Task: "Unknown-version test in fibrestats/services/ingest/test/unknown_version.test.ts"
Task: "Cross-release replay test in fibrestats/services/ingest/test/cross_release.test.ts"
```

### User Stories 5 & 6

```bash
# US5 packaging tracks in parallel:
Task: "Helm chart in deploy/helm/fibrestats/"
Task: "Self-host docs in deploy/docs/"

# US6 tests first, then page parallel with scheduling:
Task: "k≥5 suppression test in fibrestats/services/ingest/test/fleetagg_suppression.test.ts"
Task: "Fleet API test in fibrestats/test/api/fleet.test.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T011)
2. Complete Phase 2: Foundational (T012–T029) — **CRITICAL, blocks all stories**
3. Complete Phase 3: User Story 1 (T030–T044)
4. **STOP and VALIDATE**: run T044 — account → token → devnet fibre connected ≤ 1 min, version classified, rejection visibility
5. Demo: the SC-001 under-10-minutes onboarding flow

### Recommended MVP+ (joint P1 core)

US2 (dashboards) is joint-P1 with US1 per spec — ship Phases 3+4 together as the launchable MVP; an operator with flowing metrics but no dashboards gets half the day-one value.

### Incremental Delivery

1. Setup + Foundational → foundation ready (devnet metrics flowing into VM with tenant labels)
2. + US1 → onboarding demo (MVP)
3. + US2 → launchable MVP core (dashboards)
4. + US3 → alert pack (the differentiating value) → design-partner ready
5. + US4 → upgrade-safety proof → recruit mocha v10 cohort partners
6. + US5 → self-host trust unlock for security-conscious validators
7. + US6 → fleet benchmarks once ≥ 5 tenants participate
8. Each story adds value without breaking previous stories

### Parallel Team Strategy

1. Whole team: Setup + Foundational together
2. Then: Dev A → US1, Dev B → US2 (dashboards are pure provisioning + API, independent of US1 code)
3. Next wave: Dev A → US3, Dev B → US4 (fixture replay needs no UI), Dev C → US5 packaging
4. US6 last (needs ≥ 5-tenant realism); Polish gates the release

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [USn] label maps each task to its spec.md user story for traceability; Setup/Foundational/Polish tasks carry no story label by design
- Tests are mandated by the spec (FR-013/SC-005 golden fixtures, SC-008 isolation, SC-007 k≥5) — verify each fails before implementing
- Field constraints quoted verbatim from data-model.md and contracts are normative — implement exactly, do not re-decide
- Budget-utilization logic uses only the exported `budget_bytes` gauge — stake-share math is forbidden everywhere (dashboards, alerts, docs)
- The fibre binary cannot set headers: token-in-path auth is a hard input-contract fact, not a preference
- Commit after each task or logical group; stop at any checkpoint to validate a story independently
- Avoid: cross-story imports that break independence, dashboards/alerts referencing non-canonical series, any Celestia logo/mark asset
