# Contract: Self-Host Packaging (docker-compose + Helm)

**Implements**: FR-025, FR-026, FR-030, FR-033, US5 acceptance scenarios.
**Principle**: hosted and self-host run the **same component set** (parity by construction); hosted-only extras are multi-tenant Grafana org management and fleet-benchmark publication.

## 1. Components (both packages)

| Service | Image | Role |
|---|---|---|
| `adapter` | `fibrestats/adapter:<version>` | OTLP edge + versioned mapping + status + fleet aggregation (`serve` / `aggregate` / `migrate` commands) |
| `otelcol` | `otel/opentelemetry-collector-contrib:<pinned>` | pipeline: receive from adapter → batch → VictoriaMetrics; `bearertokenauth` for static-token simple mode |
| `victoriametrics` | `victoria-metrics/victoria-metrics:<pinned>` | series storage; `-retentionPeriod=30d` + downsampling rules to 13 mo (FR-030) |
| `postgres` | `postgres:16` | entity store |
| `grafana` | `grafana/grafana:<pinned>` | dashboards + alert pack, provisioned from `deploy/grafana/provisioning/` |
| `web` (optional) | `fibrestats/web:<version>` | Next.js control plane; self-host may omit it and drive tokens/rules via config files |
| `alloy` (optional, on operator hosts) | `grafana/alloy:<pinned>` | agent path (FR-023): node-exporter + `cometbft_privval_signing_latency_*` scrape → push under tenant token |

## 2. docker-compose contract

- One `docker-compose.yml` + `.env` (documented keys below); `docker compose up -d` yields a working stack (US5.1).
- Volumes: `vm-data`, `pg-data`, `grafana-data` (named volumes; retention survives restarts/upgrades).
- Ports (defaults, all overridable): adapter `:4318` (operator-facing OTLP), grafana `:3001`, web `:3000`; VM/Postgres internal-only.
- Env surface (`FIBRESTATS_*`):

| Key | Default | Meaning |
|---|---|---|
| `FIBRESTATS_DOMAIN` | `localhost` | base domain for links in notifications |
| `FIBRESTATS_ADMIN_EMAIL` | — | bootstrap owner account |
| `FIBRESTATS_RETENTION_RAW_DAYS` | `30` | VM raw retention |
| `FIBRESTATS_RETENTION_DOWNSAMPLED_MONTHS` | `13` | VM downsampled retention |
| `FIBRESTATS_FLEET_OPT_IN` | `false` | publish anonymized aggregates to the hosted benchmark (§4) |
| `FIBRESTATS_FLEET_ENDPOINT` | `https://fleet.<hosted-domain>/v1/aggregates` | where opt-in aggregates go |
| `FIBRESTATS_STATIC_TOKENS` | — | comma list for collector `bearertokenauth` simple mode (small installs without the web app) |
| `FIBRESTATS_ALERT_REPEAT_INTERVAL` | `4h` | pack-wide default throttle |

- Provisioned content: datasources (VM), the seven FR-014 dashboard groups (derived from upstream `observability/docker/grafana/dashboards/fibre.json` + `fibre-runtime.json`, re-pointed at canonical series, brand-palette themed, no Celestia marks), the 11-rule alert pack with research-R10 defaults, contact-point templates for the four channel types.
- Docs shipped with the package: install guide, agent-path setup, retention configuration, upgrade procedure (US5.4).

## 3. Helm chart contract (`deploy/helm/fibrestats`)

- One Deployment/StatefulSet per component of §1; `values.yaml` mirrors every compose env key 1:1 (same names); PVCs for the three stateful services; Ingress for adapter OTLP + Grafana + web with TLS notes.
- `helm upgrade` runs `adapter migrate` as a pre-upgrade hook (Postgres schema); new metric manifests activate without any tenant action (FR-012).
- Resource requests ship conservative defaults sized for ~10 servers; values documented for scaling to infra-provider fleets.

## 4. Data-leaving guarantee & opt-in contribution (FR-026, US5.2/5.3)

- **Default: nothing leaves the environment.** No telemetry about the self-host installation itself, no metric data, no aggregates. Verified by inspecting the compose network policy (only the operator's fibre servers and agent hosts reach the adapter) and the absence of any enabled outbound job.
- **Explicit opt-in** (`FIBRESTATS_FLEET_OPT_IN=true`): the nightly aggregation computes the same k≥5-suppressed percentile bands as hosted (research R13) and POSTs **only** the anonymized payload to `FIBRESTATS_FLEET_ENDPOINT`:

```json
{
  "installation_id": "uuid (random, regeneratable; not tied to identity)",
  "period_date": "2026-10-21",
  "aggregates": [
    { "canonical_name": "fibre.server.sign.duration", "cohort_key": "mainnet", "p10": 0.002, "p50": 0.004, "p90": 0.011, "contributing_server_count": 3 }
  ]
}
```

- Payload contains no raw series, no labels, no tenant/server identifiers. The operator can inspect exactly what would be shared before enabling: `GET http://localhost:4318/../internal/fleet-contribution/preview` (documented; renders the same JSON the publisher would send).
- Opt-in can be withdrawn at any time; past published aggregates are not retractable (data-handling page, FR-032).

## 5. Upgrade & version-support contract

- New adapter releases ship new manifests + golden-fixture contract tests; self-host operators upgrade by pulling images (`docker compose pull && up -d` or `helm upgrade`) — dashboards, alerts, tokens, and history keep working (US5.4, FR-012).
- Manifest releases are also published standalone so operators can verify adapter support for a new celestia-app release before upgrading their fibre server (SC-005: ≤ 7 days after upstream publication).

## 6. Verification hooks (used by quickstart.md)

- `GET /healthz` on adapter, Grafana, web — compose healthchecks gate `depends_on`.
- `adapter migrate` idempotent; safe to re-run.
- First-boot seed: pack rules + MetricSeriesCatalog loaded from manifests; Grafana org "selfhost" created.
