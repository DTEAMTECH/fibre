# Contract: Versioned Metric Adapter — v10.4.0 Baseline

**Implements**: FR-008…FR-013, US4 acceptance scenarios, SC-005/SC-006.
**Ground truth**: `docs/protocol-verification-v10.4.0.md` §3 appendix (code-level, tag `v10.4.0`). Where AGENTS.md §3 conflicts, this contract follows the verification doc.

## 1. Manifest format

One YAML manifest per upstream release (`services/adapter/manifests/<release>.yaml`):

```yaml
release: "v10.4.0"              # exact service.version string matched (research R6)
support: full                   # full | partial
instruments:
  - name: fibre.server.upload_shard.duration
    kind: histogram             # counter | updown | histogram | gauge | observable_gauge
    unit: s
    canonical: fibre.server.upload_shard.duration   # rename target; equal for baseline
    attributes:
      success:   { type: bool, required: true }
      upload_size: { type: int, required: false, condition: "> 0" }
    buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
namespaces:
  fibre.server.: { policy: canonical }   # map + alert/dashboard scope
  fibre.client.: { policy: passthrough } # ingest, store, label namespace=client; no MVP alerts
  go.:           { policy: passthrough } # ingest, store, label namespace=runtime; runtime dashboard only
unknown:
  policy: record_and_flag       # store series untouched + UnknownSeriesRecord; never block
```

**Classification** (FR-010): `full` = exact `service.version` match against a `support: full` manifest; `partial` = match against a drifted/deprecated manifest or known version with unmapped additions; `unknown` = no manifest match. Missing/unparseable version → `unknown`.

## 2. v10.4.0 baseline — 18 instruments (normative)

| # | Metric | Kind | Unit | Attributes | Notes / canonical semantics |
|---|---|---|---|---|---|
| 1 | `fibre.server.upload_shard.in_flight` | updown | — | — | concurrent UploadShard RPCs; drives connection-saturation rule |
| 2 | `fibre.server.upload_shard.duration` | histogram | s | `success` (bool, always); `upload_size` (int, only when > 0) | buckets 0.001…5 (11 bounds) |
| 3 | `fibre.server.upload_shard.bytes` | counter | By | — | shard bytes received |
| 4 | `fibre.server.upload_shard.rejected` | counter | — | `reason` (string) | only value at tag: `budget_exceeded` — drives rejection rule; new enum values must pass through + flag |
| 5 | `fibre.server.upload_shard.last_success_timestamp` | observable_gauge | s | — | unix s of last successful upload; **absent before first success** → classify "no uploads yet", NOT stale (rule 6 guard) |
| 6 | `fibre.server.upload_shard.occupancy_bytes` | observable_gauge | By | — | limiter-tracked bytes = on-disk + in-flight reserved; **not a disk gauge** |
| 7 | `fibre.server.upload_shard.budget_bytes` | observable_gauge | By | — | current derived budget; **the only authoritative source for budget-utilization alerting — stake math is forbidden** (real budget = FullStakeStorageBudget × assignedRows/OriginalRows, ≈3× liveness-scaled, clamped) |
| 8 | `fibre.server.download_shard.in_flight` | updown | — | — | concurrent DownloadShard RPCs |
| 9 | `fibre.server.download_shard.duration` | histogram | s | `success`; `shard_size` (only when > 0) | buckets as upload duration |
| 10 | `fibre.server.download_shard.bytes` | counter | By | — | shard bytes sent |
| 11 | `fibre.server.store.put.duration` | histogram | s | `success` | buckets 0.001…1 (9 bounds) |
| 12 | `fibre.server.store.get.duration` | histogram | s | `success` | buckets as put |
| 13 | `fibre.server.backend.get.duration` | histogram | s | `backend` (`local`\|`object`), `outcome` (`success`\|`not_found`\|`timeout`\|`canceled`\|`throttled`\|`error`) | buckets 0.001…60 (adds 2.5, 5, 10, 30, 60); primary backend only |
| 14 | `fibre.server.backend.get.in_flight` | updown | — | `backend` | |
| 15 | `fibre.server.backend.get.bytes` | counter | By | `backend` | encoded bytes incl. partial failures/buffered reads |
| 16 | `fibre.server.sign.duration` | histogram | s | `success` | buckets 0.0001…1 (9 bounds); **p50 SLO 10 ms sits exactly on the 0.01 bucket edge** — rule descriptions must note the resolution caveat |
| 17 | `fibre.server.prune.entries` | counter | — | — | incremented only when `pruned > 0`; absence ≠ prune loop dead |
| 18 | `fibre.server.prune.duration` | histogram | s | `success` | buckets 0.001, 0.01, 0.05, 0.1, 0.5, 1, 5, 10 |

Notes: instruments 5 and 13–15 **ship in v10.4.0** (AGENTS.md mislabeled them "new on main"; corrected per verification doc). The v10.4.0 upstream README metric table omits #4, #6, #7 — code is ground truth.

## 3. Same-endpoint namespaces (must all be handled)

| Namespace | Examples | Policy |
|---|---|---|
| `fibre.server.*` | table above | canonical mapping; dashboards + alerts + fleet aggregates |
| `fibre.client.*` | `fibre.client.upload.duration`, `fibre.client.download_from.rpc_latency` | ingest + store (`namespace=client`); out of MVP alert scope (spec exclusion) |
| `go.*` | runtime series from `runtime.Start` (enabled whenever OTLP is on) | ingest + store (`namespace=runtime`); feeds runtime dashboard only |
| anything else | future/unknown names | store untouched; `UnknownSeriesRecord`; drift-flagged (FR-011) |

## 4. Main-drift manifest (pre-staged, not yet "supported release")

| Metric | Kind | Attributes | Status |
|---|---|---|---|
| `fibre.server.upload_shard.dupe_hits` | counter | `stage` (string) | present on `main` only; `main-drift.yaml` maps it so series are understood on arrival; alert rule 9 activates only for versions exposing it |
| `fibre.server.store.backend` | observable_gauge (const 1) | `backend`, `object_configured` (bool) | present on `main` only; info gauge, no MVP alerts |

All other baseline instruments unchanged on `main` as of 2026-10-21.

## 5. Golden-fixture contract test (FR-013, SC-005)

- **Fixture**: one recorded OTLP `ExportMetricsServiceRequest` (protobuf) per release, captured from a real binary on the local devnet (see quickstart.md step 6), stored at `fibrestats/services/ingest/test/fixtures/golden/<release>.otlp.pb`.
- **Test assertions** (per release, `pnpm --filter @fibrestats/ingest exec vitest run golden_<release>` — vitest replaying the fixture through `@opentelemetry/otlp-transformer` decoding):
  1. Every instrument in the manifest's canonical namespace appears in the fixture and maps to its declared canonical name/kind/unit.
  2. Every `fibre.server.*` series in the fixture is covered by the manifest (coverage = 100%; SC-005) — failures name the uncovered series.
  3. Attribute schemas match (names, types, enum values, conditionality).
  4. Unknown-namespace series produce `UnknownSeriesRecord` entries, not errors.
  5. Replay of two different releases' fixtures against one tenant yields identical canonical-series semantics (US4.1/US4.3).
- A fixture replay from a fabricated "unknown future version" must yield classification `unknown` with recognized series still ingested (US4.2).

## 6. Onboarding a new release (FR-012, SC-005)

1. Record golden fixture on devnet running the new tag.
2. Diff fixture vs existing manifests → generate/update `<new>.yaml` (renames map old → same canonical names for continuous history).
3. Add contract test; mark prior manifest `partial` if drifted.
4. Ship — existing tenants change nothing; their servers reclassify automatically; version-drift alert notifies operators ahead/behind majority.
Target: support available ≤ 7 days after upstream publication (SC-005).

## 7. What the adapter writes downstream

- **VictoriaMetrics**: all series with `{tenant_id, server_id, namespace, …original attributes}`; canonical names for mapped server series.
- **Postgres**: MonitoredServer status/version/classification/last_seen; UnknownSeriesRecords; heartbeat marker series for telemetry-silence detection.
- **Never**: traces, logs, payloads, keys (FR-005); never stake-derived budget computations (gauge #7 only).
