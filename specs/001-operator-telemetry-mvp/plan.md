# Implementation Plan: Operator-Side Telemetry MVP for Celestia Fibre Servers

**Branch**: `001-operator-telemetry-mvp` | **Date**: 2026-10-21 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-operator-telemetry-mvp/spec.md`

## Summary

FibreStats MVP ingests the fibre server's **push-only OTLP/HTTP metric stream** (the binary offers no Prometheus scrape endpoint — verified at v10.4.0), attributes it to per-tenant ingest tokens, adapts the versioned `fibre.server.*` metric contract (baseline: celestia-app **v10.4.0**, 18 instruments) onto canonical series, and serves per-tenant Grafana dashboards, an opinionated 11-rule alert pack, and anonymized fleet benchmarks suppressed below a 5-tenant cohort. Stack per AGENTS.md §5: OTel Collector contrib → VictoriaMetrics (30d raw + 13mo downsampled), Postgres entity store, hosted Grafana per-tenant orgs, Grafana Alerting, nightly fleet-aggregation job. **Single-language TypeScript** (owner decision 2026-10-21): the existing Next.js 16 / TS-strict / pnpm app is the control plane (accounts, tokens, server status, alert config, benchmark views), and a **Node.js 22 LTS + TypeScript-strict ingest service** (`fibrestats/services/ingest` in the same pnpm workspace) terminates the OTLP edge — token auth, tenant attribution, version classification, versioned metric mapping, drift flagging, and fleet aggregation. Delivered hosted and self-host (docker-compose + Helm; fleet contribution opt-in, default off).

## Technical Context

**Language/Version**: TypeScript 5 (strict, `noUncheckedIndexedAccess`) as the single language across the whole product — Next.js 16.4 / React 19 web app (existing scaffold — do not re-init) and the ingest service on **Node.js 22 LTS**; YAML/JSON for collector, manifest, and Grafana provisioning

**Primary Dependencies**: Fastify 5 (ingest HTTP edge); `@opentelemetry/otlp-transformer` + protobufjs / `opentelemetry-proto` (OTLP/HTTP protobuf decode/encode — the TS OTLP tooling evaluated in research.md R1); Drizzle ORM (shared strict-typed Postgres data layer, workspace package); vitest (unit + golden-fixture contract tests); OTel Collector contrib (batching, VM exporter); VictoriaMetrics (single-node MVP); Grafana (dashboards + Alerting); Grafana Alloy (published agent-path config); Postgres 16; shadcn/ui + Tailwind CSS v4 (web UI, existing)

**Storage**: VictoriaMetrics — all metric series, tenant-discriminated by a `tenant_id` label injected at the edge; raw retention 30 days, VM downsampling rules to 13 months. Postgres — tenants, users, ingest tokens, monitored servers, alert rules/incidents, notification channels, delivery records, fleet aggregates, registry snapshots, rejected-push log

**Testing**: TypeScript contract tests (vitest) per supported server release replaying a recorded golden OTLP fixture and asserting full baseline mapping (FR-013); unit/integration tests for edge auth, mapping, classification, aggregation k≥5 suppression; `pnpm lint` + `tsc --noEmit` across the workspace; end-to-end validation per `quickstart.md` against the local celestia-app devnet (`scripts/single-node-fibre.sh`)

**Target Platform**: Linux x86_64 containers — docker-compose (self-host all-in-one) and Helm chart (k8s); hosted deployment single region for MVP; web app via Next.js (Turbopack); ingest service as a Node 22 LTS container built from the same pnpm workspace

**Project Type**: Multi-component web service — control-plane web app + ingest/adapter service + off-the-shelf observability infra (one TypeScript pnpm workspace for all custom code)

**Performance Goals**: SC-002 ≥95% of emitted points visible in tenant dashboards within 60 s; SC-003 ≥95% of alert notifications delivered within 5 min of the triggering condition; SC-004 hosted ingestion availability ≥99.9%/calendar month with per-tenant misbehavior isolation (rate-limit/quarantine)

**Constraints**:
- **Metrics-only ingestion** (FR-005): the binary also POSTs `/v1/traces` — traces are accepted and discarded; logs, blob payloads, key material are never accepted, stored, or displayed.
- **Token-in-path authentication**: the fibre binary has **no header/auth configuration** — it only appends `/v1/metrics` and `/v1/traces` to the `--otel-endpoint` base URL (path prefix preserved, verified at v10.4.0). Per-tenant credentials therefore travel as a URL path segment: `--otel-endpoint https://otel.<domain>/t/<ingest-token>`. This is a hard input-contract fact, not a design preference.
- **Budget math**: storage-budget alerting MUST use the exported `fibre.server.upload_shard.budget_bytes` gauge — never stake-share arithmetic (real budget = `FullStakeStorageBudget × assignedRows/OriginalRows`, ≈3× liveness-scaled and clamped; stake-only estimates under-read ~3×).
- Every read path tenant-scoped (FR-007); fleet aggregates never published for cohorts < 5 tenants (FR-028); retention 30d raw / 13mo downsampled enforced identically in hosted and self-host (FR-030).
- Adapter baseline for v10.4.0 MUST include `upload_shard.last_success_timestamp` and `backend.get.{duration,in_flight,bytes}` (they ship in v10.4.0); only `upload_shard.dupe_hits` and the `store.backend` info gauge are main-only drift.
- The OTLP stream carries three namespaces from one endpoint: `fibre.server.*`, `fibre.client.*`, `go.*` runtime — the adapter contract defines handling for all three.
- Celestia palette/typography tokens only; Celestia logos/marks never used anywhere (UI, Grafana theming, docs).

**Scale/Scope**: Initial cohort ~92 active mainnet validators (57 registered Fibre hosts at launch) plus mocha testnet and devnets. Per server: 18 `fibre.server.*` instruments + `fibre.client.*` + `go.*` series at the OTLP periodic-reader cadence → low hundreds of active series per server; MVP sizing ≤100 connected hosted servers. 11-rule alert pack; 7 dashboard groups (one agent-path-gated); 4 notification channel types.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design — see bottom.*

| Principle | Verdict | Evidence |
|---|---|---|
| I. Spec-Driven Workflow (NON-NEGOTIABLE) | **PASS** | `specs/001-operator-telemetry-mvp/spec.md` predates this plan; pipeline specify → plan followed; this phase produces design artifacts only — no source code. |
| II. Celestia Brand Fidelity | **PASS** | Plan introduces no new colors, fonts, or motifs; web UI stays on the existing licensed-token theme in the scaffold; Celestia logos/marks are never used — the ban extends to Grafana theming, alert templates, and docs. |
| III. shadcn/ui + Tailwind Exclusively | **PASS** | All new tenant-facing UI is built in the existing scaffold with shadcn/ui primitives + Tailwind. Grafana dashboards are JSON provisioning for an off-the-shelf observability product (AGENTS.md §5 mandated stack), not application UI. |
| IV. TypeScript Strict Mode | **PASS** | Single-language product: web app **and** ingest service are TypeScript strict in one pnpm workspace. *Owner resolved the backend-language question on 2026-10-21 in favor of Node.js/TypeScript (AGENTS.md §0's acceptable single-language path) — no exception required.* |
| V. pnpm Package Management | **PASS** | pnpm-only across the entire workspace (scaffold already `pnpm@9.15.0`); the ingest service is a workspace package (`@fibrestats/ingest`) — one lockfile, `pnpm-lock.yaml`. |
| VI. Discovery Artifacts & Research Boundaries | **PASS** | Sibling celestia workspace treated read-only (read for reference, never modified); all decisions and artifacts recorded under `specs/001-operator-telemetry-mvp/`. |

**Initial gate result: PASS — zero exceptions.**

## Project Structure

### Documentation (this feature)

```text
specs/001-operator-telemetry-mvp/
├── plan.md              # This file (/skill:speckit-plan output)
├── research.md          # Phase 0 output — decisions, rationale, alternatives
├── data-model.md        # Phase 1 output — entities, fields, validation, state transitions
├── quickstart.md        # Phase 1 output — end-to-end devnet validation guide
├── contracts/           # Phase 1 output — external interface contracts
│   ├── otlp-ingest.md
│   ├── adapter-mapping-v10.4.0.md
│   ├── web-api.md
│   ├── alert-channels.md
│   └── self-host-packaging.md
├── checklists/          # (existing) requirements checklist
└── tasks.md             # Phase 2 output (/skill:speckit-tasks — NOT created by this command)
```

### Source Code (repository root)

```text
fibrestats/                    # pnpm workspace root (existing Next.js app, gitignored by umbrella
│                              #  policy) — ALL custom code lives here, TypeScript strict
├── app/                       # App Router pages + route handlers implementing contracts/web-api.md
├── components/                # shadcn/ui-based components (existing conventions)
├── lib/                       # session auth, domain logic
├── packages/
│   └── db/                    # shared strict-typed data layer (Drizzle ORM) — used by the app
│                              #  route handlers and the ingest service alike
├── services/
│   └── ingest/                # NEW Node 22 / TS service (@fibrestats/ingest): OTLP edge
│       │                      #  (token-in-path auth → tenant_id), versioned metric adapter,
│       │                      #  drift detector, status writer, fleet-aggregation CLI
│       ├── src/
│       │   ├── edge/          #   Fastify OTLP/HTTP termination, token auth, tenant_id injection,
│       │   │                  #   metrics-only enforcement, rate-limit/quarantine, skew handling
│       │   ├── mapping/       #   version manifests → canonical series; unknown-series capture
│       │   ├── status/        #   server status/version/classification writer (Postgres)
│       │   └── fleetagg/      #   nightly cross-tenant percentile job with k≥5 suppression
│       ├── manifests/         #   per-release metric manifests (v10.4.0.yaml, main-drift.yaml …)
│       └── test/fixtures/golden/ #   recorded OTLP ExportMetricsServiceRequest fixtures per release
│                              #   (FR-013 contract tests replay these via vitest)
├── package.json               # existing Next.js app package (workspace root)
└── pnpm-workspace.yaml        # workspace: app + packages/* + services/* (one pnpm-lock.yaml)

deploy/
├── docker-compose.yml         # self-host all-in-one: ingest, otelcol, victoriametrics,
│                              #  postgres, grafana, web
├── otelcol/config.yaml        # collector pipeline: receive from ingest service → batch → VM
├── grafana/provisioning/      # datasources, per-tenant dashboard JSON (derived from the official
│                              #  observability/docker/grafana/dashboards/fibre.json + fibre-runtime.json),
│                              #  alert-pack rules, contact-point templates
├── agent/alloy/               # published agent-path config: node-exporter + cometbft privval
│                              #  (cometbft_privval_signing_latency_*) scrape → push under tenant token
└── helm/fibrestats/           # k8s packaging of the same components (values mirror compose env)
```

**Structure Decision**: Single-language layout — every line of custom code is TypeScript in the existing `fibrestats/` directory, now a pnpm workspace: the Next.js app (untouched scaffold conventions) plus `services/ingest` (the OTLP edge + versioned adapter) plus `packages/db` (shared data layer). `deploy/` at repo root holds infra/packaging config unchanged (collector, VM, Grafana, Alloy, compose, Helm). Hosted and self-host run the *same* component set (parity by construction); hosted-only extras are multi-tenant Grafana org management and fleet-benchmark publication. Grafana remains the metric-visualization surface for MVP (per-tenant orgs, provisioned dashboards); the Next.js app owns everything else tenants touch and links into Grafana.

## Phase 0/1 Outputs (generated)

- `research.md` — 15 decisions (language, edge auth, adapter architecture, storage, dashboards, alerting, version classification, namespace policy, tokens, agent path, fleet aggregation, packaging, dev loop, web-app data path, retention) with rationale and alternatives.
- `data-model.md` — 13 entities with fields, relationships, validation rules from FRs, and state transitions.
- `contracts/` — five contracts: OTLP ingest, v10.4.0 adapter mapping (golden-fixture contract), web query/management API, alert notification channels (Telegram/Discord/webhook/PagerDuty), self-host packaging (Helm/docker-compose).
- `quickstart.md` — runnable end-to-end devnet validation: devnet → local stack → token → fibre `--otel-endpoint` → per-metric landing checklist → golden-fixture capture → alert trigger → dashboard.

## Constitution Check — Post-Design Re-evaluation

| Principle | Verdict | Post-design evidence |
|---|---|---|
| I. Spec-Driven | **PASS** | All artifacts derive from spec.md FRs/entities; every contract cites its FRs; no code produced. |
| II. Brand Fidelity | **PASS** | No visual assets introduced; contracts and quickstart contain no logos/marks; dashboard derivation uses upstream JSON as data, re-themed with brand palette only. |
| III. shadcn/ui + Tailwind | **PASS** | Web-app surfaces in contracts/web-api.md are implemented in the existing scaffold; no alternative UI library anywhere in the design. |
| IV. TypeScript Strict | **PASS** | Single-language design: edge, mapping, drift detection, fleet worker, and web app are all TypeScript strict in one workspace. No exceptions. |
| V. pnpm | **PASS** | All dependency operations across all packages via pnpm; single `pnpm-lock.yaml`. |
| VI. Research Boundaries | **PASS** | Sibling workspace untouched; verification doc (`docs/protocol-verification-v10.4.0.md`) treated as ground truth and incorporated (corrections a–e in Constraints and contracts). |

**Post-design gate result: PASS — zero exceptions.** (The one exception documented in an earlier revision of this plan — a Go backend vs Principle IV — was withdrawn after the owner's 2026-10-21 decision for Node.js/TypeScript.)

## Complexity Tracking

> No violations require justification. The owner resolved the backend-ingest language question on 2026-10-21 (Node.js/TypeScript); the previously documented Principle IV exception is withdrawn, and this section is intentionally empty.
