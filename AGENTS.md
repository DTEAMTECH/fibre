# FibreStats — Agent Context & Handoff

> **Read this file first.** It contains everything needed to start building: product definition, protocol reference, architecture decisions, MVP scope, and a kickoff task list. It was prepared on 2026-10-21 from code-level research against celestia-app `v10.4.0` / `main` and live network state. Deeper reference docs are linked in §8 — read them when you need more detail.

## 0. Existing codebase — do not scaffold from scratch

This repo is the **fibre umbrella** (published at `github.com/DTEAMTECH/fibre`): README showcase, project docs, and the **Spec Kit** workflow (`.specify/`). The **fibrestats web app** lives in the **`fibrestats/` subdirectory** — a branded scaffold: Next.js 16 (App Router, TypeScript, Turbopack) + Tailwind CSS v4 + shadcn/ui, themed with official Celestia brand palette/typography tokens. The app code is **gitignored and intentionally not published in this repo** (it will get its own repo when it's ready); the umbrella README showcases it. **Read `README.md` first, follow the Spec Kit workflow already set up (spec → plan → tasks before code), and build the product's web surface in the existing app under `fibrestats/`.** Do not re-initialize the frontend, do not replace the stack, and respect the existing branding/theming work.

Brand boundary: palette and typography tokens only — **Celestia logos/marks are never used** in our apps or materials (owner decision, mirrored in `.specify/memory/constitution.md`).

Stack guidance: the web app is TypeScript/Next.js — that decision is made. The **telemetry ingestion path (OTLP receiver, versioned metric adapter) is a separate backend concern**; the recommendation is a small standalone Go service (the OTLP/OTel ecosystem and all Celestia code are Go-native), deployed alongside VictoriaMetrics + Grafana. Treat that backend choice as a recommendation to confirm with the owner in your plan — if the owner prefers a single-language codebase, a Node/TS ingest path (OTel Collector handles the protocol anyway; the adapter is just a mapping service) is acceptable.

---

## 1. What we are building (the one-paragraph version)

**FibreStats is an operator-side telemetry product for Celestia Fibre servers.** Celestia's v10 upgrade introduced Fibre: a parallel data-availability protocol where validators run a `fibre` server binary that stores and serves blob shards. Every existing ecosystem tool (Tensile, Cumulo, Valopers, ITRocket, NodeStake…) measures validators **from the outside** (reachability probes, on-chain signature counts). **Nobody ingests the fibre server's own internal OTLP metrics** — operators are currently told to wire up their own OpenTelemetry collector + Grafana by hand. We build the hosted (and self-hostable) service that does it for them: per-tenant dashboards, an opinionated alert pack, and anonymized fleet benchmarks.

- **Users:** Celestia validators / node operators running fibre servers (~92 active validators on mainnet; 57 registered a Fibre host at launch), plus infra providers hosting validators.
- **Purpose:** let operators see and keep their own Fibre server healthy — budget exhaustion, prune lag, signing-latency SLO breaches, connection starvation, disk pressure — before external observers flag them publicly.
- **Differentiation:** we are the *internal* telemetry layer; external-verdict data (Tensile/Cumulo) is consumed as complementary annotations, never re-implemented.

## 2. Timing context (why now)

- Fibre activated on **mocha-5 testnet 2026-09-24** (height 1,082,619) and on **mainnet 2026-10-20 03:45:49 UTC (height 15,116,659)**.
- Current binaries: **celestia-app v10.4.0** (mainnet), **celestia-node v0.34.3**. Mainnet params identical to testnet (see §4).
- Mainnet usage is smoke-test level (namespace `fibremain`, 2 blobs on day one). Real production load is expected later; mocha + local devnets are the development targets for load.
- Validator attention peaks *right now* (registration week). The metric surface is explicitly unstable — names changed between v10.2.0 and `main` already — so the **versioned metric adapter (§5) is a core feature, not an afterthought**.

## 3. How the fibre server exposes data (the input contract)

The `fibre` binary ships with celestia-app releases. Observability is **push-only OTLP/HTTP** — there is **no Prometheus scrape endpoint**:

- Enable with: `fibre start --otel-endpoint http://<collector>:4318` (env `FIBRE_OTEL_ENDPOINT`). The binary appends `/v1/traces` and `/v1/metrics` to the base URL. Disabled by default.
- Tracing: 10% parent-based sampling, W3C TraceContext, exemplars link metrics↔traces. Resource attrs: `service.name=fibre`, `service.version`, `service.instance.id`.
- Logging: `--log-level`, `--log-format json`. Profiling: `--pprof` (:6060), `--pyroscope-endpoint`.
- Official Grafana dashboards ship in-repo at `fibre/dashboards/fibre-dashboards.json` — use as the starting point.

**Exact server metric names (from `fibre/server_metrics.go`; histograms carry `success`, some `upload_size`/`shard_size` attrs):**

| Metric | Type | Meaning |
|---|---|---|
| `fibre.server.upload_shard.in_flight` | UpDownCounter | concurrent UploadShard RPCs |
| `fibre.server.upload_shard.duration` | Histogram (s) | upload latency |
| `fibre.server.upload_shard.bytes` | Counter (By) | shard bytes stored |
| `fibre.server.upload_shard.rejected` | Counter (`reason`, e.g. `budget_exceeded`) | rejected uploads |
| `fibre.server.upload_shard.dupe_hits` | Counter (`stage`) | duplicate uploads (new on `main`) |
| `fibre.server.upload_shard.last_success_timestamp` | Gauge (unix s) | watchdog — stale = server stopped accepting (new on `main`) |
| `fibre.server.upload_shard.occupancy_bytes` / `.budget_bytes` | ObservableGauge | disk usage vs stake-derived budget |
| `fibre.server.download_shard.{in_flight,duration,bytes}` | as above | serving path |
| `fibre.server.store.{put,get}.duration` | Histogram | storage engine |
| `fibre.server.backend.get.{duration,in_flight,bytes}` | Histogram/UpDown/Counter (`backend=local\|object`, `outcome=…`) | object-storage backend (new on `main`) |
| `fibre.server.sign.duration` | Histogram (s) | payment-promise signing |
| `fibre.server.prune.{entries,duration}` | Counter / Histogram | prune loop health |

Related node-side signals we also want (via the agent path, §5): host disk usage (node-exporter — the fibre server has **no disk gauge**), and consensus-node KMS latency `cometbft_privval_signing_latency_*` (SLO: median ≤ 10 ms; node warns when the median of the last 50 signatures exceeds it).

**Known operational facts that drive our alerts:**

- Storage budget per validator = `full_stake_storage_budget (2 TiB) × stake share`; over-budget uploads are rejected (`budget_exceeded`).
- Worst-case RAM ≈ `max_connections (16) × max_concurrent_streams (13) × ~132 MiB` ≈ 27 GiB at defaults; one upload occupies 16 connections → downloads starve unless raised.
- Shards prune every 1 min at `max(ExpiresAt, creation + shard_retention(4h))`; measured actual deletion lags ~1m45s past the deadline.
- Config: `$FIBRE_HOME/config/server_config.toml` (`app_grpc_address` 9090, `signer_grpc_address` 26669 — both plaintext, loopback only; `server_listen_address` public :7980; `min_upload_size` local admission policy — inconsistent settings across validators can fragment quorum).
- Client-facing (celestia-node) OTLP histograms: `fibre_upload_duration_seconds`, `fibre_submit_duration_seconds`, `fibre_escrow_*_duration_seconds` — future scope, not MVP.

## 4. Chain-side reference (for context/annotations, not core ingestion)

- Modules: `x/fibre` (escrow accounts, payment promises, settlement) and `x/valaddr` (validator consensus address → fibre `host:port` registry).
- Key txs: `MsgDepositToEscrow`, `MsgRequestWithdrawal`, `MsgPayForFibre` (PFF — settlement; one per tx, ≤200/block), `MsgPaymentPromiseTimeout`, gov `MsgUpdateFibreParams`; registry: `MsgSetFibreProviderInfo`.
- Key queries (REST): `/fibre/v1/params`, `/fibre/v1/escrow-account/{signer}`, `/fibre/v1/withdrawals/{signer}`, `/fibre/v1/is-payment-processed/{hash}`, `/valaddr/v1/all-bonded-fibre-providers`.
- Typed events (indexable): `celestia.fibre.v1.Event{DepositToEscrow, WithdrawFromEscrowRequest, WithdrawFromEscrowExecuted, PayForFibre, PaymentPromiseTimeout, UpdateFibreParams, ProcessedPaymentPruned}`; plain event `set_fibre_provider_info{validator_consensus_address, host}`.
- Params (gov): `shard_retention` 4h, `payment_promise_timeout` 1h, `payment_promise_height_window` 1000, `withdrawal_delay` 24h, `full_stake_storage_budget` 2 TiB.
- Fee formula: `(650,000 + 45,000 × ⌈blob_size / 262,144⌉)` utia → fee_collector. Payment formula marked TODO in code — treat as parametric.

## 5. Product architecture (decided)

Three ingestion shapes — support all; they map to operator trust postures:

1. **Direct OTLP push (zero-install, MVP):** operator adds `--otel-endpoint https://otel.<our-domain>` + per-tenant token. Metrics only, no keys, no inbound access.
2. **Agent path:** published Grafana Alloy / OTel Collector config that additionally scrapes node-exporter (disk) and the consensus node's privval latency, then pushes the bundle. Needed for the disk-vs-budget and KMS-SLO views.
3. **Self-host:** the whole stack as Helm chart / docker-compose; optional opt-in to publish only anonymized aggregates back.

Server-side components:

| Component | Choice (suggested) | Notes |
|---|---|---|
| OTLP receiver + auth | OTel Collector contrib + `bearertokenauth` | per-tenant tokens → `tenant_id` resource attr |
| **Versioned metric adapter** | custom Go service (or OTel transform processor) | maps dotted `fibre.server.*` names per celestia-app release tag to canonical series; detects unknown server versions; contract tests per release |
| Time-series storage | VictoriaMetrics (or Mimir) | multi-tenant; raw 30d + downsampled 13mo |
| Entity store | Postgres | tenants, validators, registry snapshots, alert state |
| Dashboards | hosted Grafana, per-tenant orgs | start from official `fibre-dashboards.json`, extend |
| Alerting | Grafana Alerting (or standalone) | routing: Telegram/Discord/webhook/PagerDuty per tenant |
| Fleet aggregation | nightly job | anonymized percentiles; **never publish cohorts < 5 tenants** |

## 6. MVP scope

**Alert pack v1 (the real product):** budget utilization > 80% · `budget_exceeded` rejections > 0 · occupancy growth-rate anomaly · prune lag · `sign.duration` p50 > 10 ms · `last_success_timestamp` stale during active windows · disk free < budget headroom (agent path) · connection saturation / download starvation · dupe-hit spike · server version drift vs network majority.

**Dashboards v1:** Fleet-at-a-glance (own validator only in MVP) · Uploads · Serving/downloads · Storage & budget · Signing · Connections · Host & disk (agent path).

**Explicitly out of scope for MVP:** external shard probing (Tensile/Cumulo do this), on-chain indexing product views (that's "Idea 2", a later sibling product), publisher/escrow UX, mobile apps.

**Trust rules (non-negotiable):** metrics-only ingestion (no keys, no blob payloads), per-tenant isolation, k-anonymity ≥ 5 for any fleet aggregate, documented data-handling page, self-host option always available.

## 7. Kickoff task list (suggested order)

1. **Do not scaffold** — the Next.js scaffold exists (see §0). Following the Spec Kit workflow, write the spec for the telemetry MVP first, then plan, then tasks; confirm the backend-ingest language choice (Go service vs Node/TS) with the owner in the plan.
2. Stand up the dev loop: local celestia-app devnet with Fibre — `scripts/single-node-fibre.sh` in the celestia-app repo (or `plsgiveup/fibre` `fibre-devnet` for multi-validator) — so `fibre.server.*` metrics flow without mainnet.
3. OTLP receiver + tenant tokens; point the devnet fibre server at it; verify every metric from §3 lands with expected names/labels; record a golden fixture (this becomes the adapter contract test).
4. Versioned adapter v1 for celestia-app v10.4.0 (and note drift vs `main`: `dupe_hits`, `last_success_timestamp`, `backend.*`, `min_upload_size` are newer than v10.2.0 docs).
5. VictoriaMetrics + Grafana; import official dashboard; build alert pack v1.
6. Dogfood on our own mocha/mainnet nodes; then recruit 3–5 validators from the mocha v10 cohort as design partners.

## 8. Reference material

**In the sibling workspace (readable on this machine):**

- `/Users/0x00d/Documents/Kimi/Workspaces/celestia/fibre-monitoring-exploration.md` — full code-level protocol reference (x/fibre + x/valaddr API surface, events, node RPC, ops details)
- `/Users/0x00d/Documents/Kimi/Workspaces/celestia/fibre-product-brief-telemetry-analytics.md` — product strategy, adoption plan, milestones, risks
- `/Users/0x00d/Documents/Kimi/Workspaces/celestia/fibre-apps-gap-analysis.md` — competitive landscape

**External (fetch when needed):**

- fibre server README (metrics contract): https://github.com/celestiaorg/celestia-app/blob/main/fibre/cmd/README.md
- Metrics docs: https://docs.celestia.org/operate/consensus-validators/fibre/metrics/
- CIP-51 (protocol spec): https://github.com/celestiaorg/CIPs/blob/main/cips/cip-051.md
- Releases: https://github.com/celestiaorg/celestia-app/releases
- Validator runbook / ops pain points: https://forum.celestia.org/t/fibre-from-the-validator-side-v10-lab-operator-runbook-readiness-checklist/2299
- Live endpoints: mainnet RPC `https://rpc.celestia.pops.one`, REST `https://api.celestia.pops.one`; mocha `https://public-endpoint.celestia-mocha.quiknode.pro`; reference data: Tensile `https://tensile.huginn.tech/api/v1/*`, Cumulo `https://fibre-api.cumulo.com.es`

**Rules of engagement:** verify protocol facts against code at the exact deployed tag (docs drift); treat metric names as unstable until mainnet settles; never fabricate network numbers — query live endpoints; keep everything buildable and testable against the local devnet first.
