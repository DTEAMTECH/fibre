# Protocol fact verification — AGENTS.md §3/§4 vs celestia-app `v10.4.0`

- **Date:** 2026-10-21
- **Scope:** fact-check of AGENTS.md §3 (fibre server OTLP input contract) and §4 (chain-side x/fibre + x/valaddr reference) against celestia-app source at tag `v10.4.0` (mainnet version). Main-branch drift checked where AGENTS.md claims "new on `main`".
- **Method:** full source tree of `celestiaorg/celestia-app` at `v10.4.0` fetched and grepped locally; every claim below is backed by a quoted line from the tagged source. Main-branch files fetched ad hoc for drift comparison.
- **Result:** 53 facts checked — **46 confirmed · 5 DIFFERS · 1 not found at tag · 1 not verifiable from code** (empirical claim). Details and required fixes below.

---

## 1. Status table

Status legend: **confirmed** = claim matches source at `v10.4.0` · **DIFFERS** = claim is wrong or imprecise at `v10.4.0` · **not found at tag** = referenced artifact absent · **n/a (empirical)** = cannot be verified from source.

### §3 — OTLP / observability enablement (`fibre/cmd`)

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 1 | Push-only OTLP/HTTP; **no Prometheus scrape endpoint** | confirmed | `fibre/cmd/metrics.go:13,36` — only exporter wired is `otlp/otlpmetric/otlpmetrichttp`; repo-wide grep for `prometheus` under `fibre/` hits only a Pyroscope label-naming comment |
| 2 | `--otel-endpoint` flag, env `FIBRE_OTEL_ENDPOINT` | confirmed | `fibre/cmd/tracing.go:21-22` — `flagOTelEndpoint = "otel-endpoint"`, `envOTelEndpoint = "FIBRE_OTEL_ENDPOINT"`; README:242-244 |
| 3 | Binary appends `/v1/traces` and `/v1/metrics` to the base URL | confirmed | `fibre/cmd/tracing.go:49` — `url.JoinPath(endpoint, "v1/traces")`; `fibre/cmd/metrics.go:32` — `url.JoinPath(endpoint, "v1/metrics")`; README:250 (proxy prefix preserved) |
| 4 | OTLP disabled by default | confirmed | `fibre/cmd/tracing.go:23` — `defaultOTelEndpoint = ""`; `tracing.go:45-47` / `metrics.go:28-30` — empty endpoint ⇒ no-op shutdown; README:244 *(disabled)* |
| 5 | 10% parent-based trace sampling | confirmed | `fibre/cmd/tracing.go:73` — `sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1)))`; README:254 |
| 6 | W3C TraceContext | confirmed | `fibre/cmd/tracing.go:80-83` — composite propagator `TraceContext{}` + `Baggage{}` (Baggage is an extra beyond AGENTS.md); README:256 |
| 7 | Resource attrs `service.name=fibre`, `service.version`, `service.instance.id` | confirmed | `fibre/cmd/metrics.go:42-46` and `tracing.go:59-63` — `semconv.ServiceName("fibre")`, `ServiceVersion(version)`, `ServiceInstanceID(hostname)` (instance id = hostname); README:258 |
| 8 | `--log-level`, `--log-format json` | confirmed | `fibre/cmd/log.go:12-17` — flags `log-level`/`log-format`, envs `FIBRE_LOG_LEVEL`/`FIBRE_LOG_FORMAT`; README:229-232 (defaults `info`/`text`, values `debug,info,warn,error` / `text,json`) |
| 9 | `--pprof` (:6060) | confirmed | `fibre/cmd/profiling.go:19-20` — `flagPProf = "pprof"`, `defaultPProfAddress = "localhost:6060"`; README:310-315 |
| 10 | `--pyroscope-endpoint` | confirmed | README:336-340 (`FIBRE_PYROSCOPE_ENDPOINT` + basic-auth envs); `fibre/cmd/tracing.go:77` wraps provider with `otelpyroscope` for span-ID profile labels |
| 11 | Metrics exported via periodic reader; exemplars link metrics↔traces | confirmed | `fibre/cmd/metrics.go:52` — `sdkmetric.NewPeriodicReader(exp)`; README:260. Extra: `metrics.go:57` also starts Go runtime instrumentation (`runtime.Start`) — expect `go.*` runtime series alongside `fibre.*` when OTLP is on |

### §3 — server metric table (`fibre/server_metrics.go`)

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 12 | `fibre.server.upload_shard.in_flight` UpDownCounter | confirmed | `fibre/server_metrics.go:66` — `Int64UpDownCounter("fibre.server.upload_shard.in_flight")` |
| 13 | `fibre.server.upload_shard.duration` Histogram (s), attrs `success` (+`upload_size`) | confirmed | `server_metrics.go:73-77` (unit `s`, explicit buckets 0.001…5); `:238-241` — `success` bool always, `upload_size` int64 only when > 0 |
| 14 | `fibre.server.upload_shard.bytes` Counter (By) | confirmed | `server_metrics.go:82-85` (unit `By`) |
| 15 | `fibre.server.upload_shard.rejected` Counter (`reason`, e.g. `budget_exceeded`) | confirmed | `server_metrics.go:90` ("rejected by the storage limiter, by reason"); recorded once at `fibre/server_upload.go:106` — `attribute.String("reason", "budget_exceeded")` is the **only** reason value at v10.4.0 |
| 16 | `fibre.server.upload_shard.dupe_hits` Counter (`stage`) — "(new on `main`)" | confirmed | Absent at v10.4.0 (zero matches repo-wide). Present on `main`: `server_metrics.go:107` — description "…skipped storing because the shard was already stored, by detection stage"; attr `stage` (`main fibre/server_upload.go:206-207`) |
| 17 | `fibre.server.upload_shard.last_success_timestamp` Gauge (unix s) — "(new on `main`)" | **DIFFERS** — already shipped in v10.4.0 | `server_metrics.go:97-108` — `Int64ObservableGauge("fibre.server.upload_shard.last_success_timestamp")`, unit `s`, not reported before first success (`:101` `if ts > 0`); documented in v10.4.0 README:287; covered by `fibre/server_upload_metrics_test.go:64` |
| 18 | `fibre.server.upload_shard.occupancy_bytes` / `.budget_bytes` ObservableGauge — "disk usage vs stake-derived budget" | confirmed (with semantic precision) | `server_metrics.go:110-130` — both `Int64ObservableGauge`, unit `By`. `occupancy_bytes` = "Shard bytes tracked by the storage limiter: **on-disk plus in-flight reserved**" (`:111`) — a limiter counter, not a disk gauge (consistent with AGENTS.md's separate "no disk gauge" note); `budget_bytes` = "Current per-node storage budget of the limiter" |
| 19 | `fibre.server.download_shard.{in_flight,duration,bytes}` (duration attrs `success`, `shard_size`) | confirmed | `server_metrics.go:133/:140-143/:149-152`; attrs at `:253-256` (`shard_size` only when > 0) |
| 20 | `fibre.server.store.{put,get}.duration` Histogram | confirmed | `server_metrics.go:158-161/:167-170` (unit `s`, buckets 0.001…1); attr `success` via `observeStoreOp` `:262-264` |
| 21 | `fibre.server.backend.get.{duration,in_flight,bytes}` (`backend=local\|object`, `outcome=…`) — "(new on `main`)" | **DIFFERS** — already shipped in v10.4.0 | `server_metrics.go:176-196` (duration unit `s`, buckets 0.001…60); attrs `:271-277` `backend` + `outcome`; outcome enum `:20-26` = `success\|not_found\|timeout\|canceled\|throttled\|error`; `backend=local\|object` documented in v10.4.0 README:300; covered by `fibre/store_metrics_test.go:43-53` |
| 22 | `fibre.server.sign.duration` Histogram (s) | confirmed | `server_metrics.go:199-202` (unit `s`, buckets 0.0001…1); attr `success` (`:329-331`) |
| 23 | `fibre.server.prune.{entries,duration}` Counter / Histogram | confirmed | `server_metrics.go:209-211/:216-219` (duration unit `s`, attr `success`, `:334-338`) |

Note: the v10.4.0 `fibre/cmd/README.md` server-metrics table (lines 284-298) **omits** `upload_shard.rejected`, `occupancy_bytes`, `budget_bytes` — the code (and ADR-029) is ground truth; treat the README table as incomplete.

### §3 — operational facts

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 24 | Storage budget per validator = `full_stake_storage_budget (2 TiB) × stake share` | **DIFFERS (simplification)** — budget is stake-proportional but scaled by the liveness inverse and clamped | `fibre/server.go:217-218` — "`FullStakeStorageBudget * assignedRows / OriginalRows`"; `deriveBudget` `:279-290`; `fibre/validator/set.go:108-113` — `AssignedRows = ceil(OriginalRows × VotingPower × lt.Denominator / (TotalVotingPower × lt.Numerator))`, clamped to `[MinRowsPerValidator, OriginalRows]`; with `LivenessThreshold = 1/3` (`fibre/protocol_params.go:66`) the multiplier is ≈ 3× raw stake share |
| 25 | Over-budget uploads rejected (`budget_exceeded`) | confirmed | `fibre/server_upload.go:106` — rejection + `rejected{reason="budget_exceeded"}` metric; reject carries a `RetryInfo` hint of one prune interval (`fibre/client_server_budget_test.go:63-64`) |
| 26 | Worst-case RAM ≈ `max_connections (16) × max_concurrent_streams (13) × ~132 MiB` ≈ 27 GiB | confirmed | `fibre/internal/grpc/server.go:29-32` — `DefaultMaxConnections = 16`, `DefaultMaxConcurrentStreams = 13`, comment `:20` "~132 MiB"; README:124-130 — formula and "suit a 32 GiB validator (≈ 27 GiB)" |
| 27 | One upload occupies 16 connections → downloads starve unless raised | confirmed | README:132 — "An upload uses 16 signers, so it fills all 16 connection slots and blocks concurrent downloads. Raise `max_connections` above 16 to keep slots free for downloads." |
| 28 | Shards prune every 1 min at `max(ExpiresAt, creation + shard_retention(4h))` | confirmed | `fibre/server_prune.go:9` — `pruneInterval = time.Minute`; `fibre/server_upload.go:244-250` — `shardPruneAt`: `floor := creationTimestamp.Add(retention); if expiresAt.After(floor) { return expiresAt }; return floor`; same contract exposed chain-side in `proto/celestia/fibre/v1/query.proto:98-99` |
| 29 | Measured actual deletion lags ~1m45s past the deadline | n/a (empirical) | Not derivable from source; keep as an ops observation, cite source when publishing |
| 30 | Config `$FIBRE_HOME/config/server_config.toml`; `app_grpc_address` 9090, `signer_grpc_address` 26669; `server_listen_address` public :7980 | confirmed | `fibre/server_config.go:27` (`server_config.toml`), `:44-46` (`config/` under home, default `~/.celestia-fibre`); defaults `:127-129` — `127.0.0.1:9090`, `0.0.0.0:7980`, `127.0.0.1:26669`; README:13,89 |
| 31 | "both [app/signer gRPC] plaintext, loopback only" | **DIFFERS (imprecise)** — app link is plaintext-only *always*; signer link is plaintext on loopback **or mTLS for remote** | README:9,218 — "The app link (`--app-grpc-address`) is not TLS-protected" (no TLS option exists); `fibre/server_config.go:232-255` — signer dials plaintext only to loopback IP literals, otherwise requires `signer_grpc_{ca,cert,key}_file` mTLS (or `signer_grpc_allow_insecure`); README:148-176 |
| 32 | `min_upload_size` local admission policy (inconsistent settings can fragment quorum) | confirmed (exists at v10.4.0) | `fibre/server_config.go:67` — toml `min_upload_size`, default from `p.Rows * p.MinRowSize` = 262144 (README:93 — "defaults to `262144` (256 KiB)"); README:99 — "local admission policy … incompatible minimums can prevent clients from collecting enough signatures" |
| 33 | Official Grafana dashboards ship in-repo at `fibre/dashboards/fibre-dashboards.json` | **not found at tag** (also 404 on `main`) | No `fibre/dashboards/` in the v10.4.0 tree (git tree API + tarball checked); raw URL 404 at both `v10.4.0` and `main`. The v10.4.0 README:306 links that path — an upstream broken link. Actual dashboards at v10.4.0: `observability/docker/grafana/dashboards/fibre.json` ("Fibre Network", uid `fibre-network`, 16 panels) and `observability/docker/grafana/dashboards/fibre-runtime.json` ("Fibre Go Runtime", uid `fibre-go-runtime`, 12 panels); copies also under `experiments/fibre-*/payload/observability/...` |

### §4 — x/fibre params (`x/fibre/types/params.go`)

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 34 | `shard_retention` 4h | confirmed | `x/fibre/types/params.go:28` — `DefaultShardRetention = 4 * time.Hour` (bounds: min 10m `:73`, max 7d `:75`) |
| 35 | `payment_promise_timeout` 1h | confirmed | `params.go:23` — `DefaultPaymentPromiseTimeout = 1 * time.Hour` (bounds: min 10m `:67`, max 12h `:70`) |
| 36 | `payment_promise_height_window` 1000 | confirmed | `params.go:25` — `DefaultPaymentPromiseHeightWindow uint64 = 1000` |
| 37 | `withdrawal_delay` 24h | confirmed | `params.go:21` — `DefaultWithdrawalDelay = 24 * time.Hour` (bounds: min = MaxPaymentPromiseTimeout + 10m `:56`, max 7d `:61`) |
| 38 | `full_stake_storage_budget` 2 TiB | confirmed | `params.go:31` — `DefaultFullStakeStorageBudget uint64 = 2 << 40 // 2 TiB (~146 MiB/s full-stake)` |
| 39 | Params are gov-updatable | confirmed | `proto/celestia/fibre/v1/tx.proto:51,106-114` — `UpdateFibreParams`, signer = `authority` (gov); five fields in `proto/celestia/fibre/v1/params.proto:12-28` match the five keys `params.go:14-18` |

### §4 — fee/payment formula

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 40 | Fee = `(650,000 + 45,000 × ⌈blob_size / 262,144⌉)` utia | confirmed | `x/fibre/types/gas.go:20-26` — `EstimateGasForPayForFibre`: `chunks := (blobSize + 262144 - 1) / 262144` (ceil); `return 650_000 + 45_000 * chunks`; constants `pkg/appconsts/fibre_gas_consts.go:9-15` — `PFBFibreGasFixedCost=650_000`, `PFBFibreGasPerChunk=45_000`, `PFBFibreChunkSize=262_144`; `gas.go:38-40` `PaymentAmount` = 1 utia/gas in `BondDenom` |
| 41 | Payment goes to `fee_collector` | confirmed | `x/fibre/keeper/msg_server.go:328` — `SendCoinsFromModuleToModule(ctx, types.ModuleName, authtypes.FeeCollectorName, …)` ("distributed to validators and delegators like a regular data-availability fee", `:322-323`); test `msg_server_test.go:711,736` |
| 42 | "Payment formula marked TODO in code — treat as parametric" | confirmed (refined) | The gas formula itself is declared "the single source of truth" (`gas.go:18-19`); the TODOs question only the **1 gas = 1 utia** conversion: `x/fibre/keeper/keeper.go:446` — "TODO: This assumes 1 gas = 1 utia but the minimum gas price could be different."; `x/fibre/keeper/msg_server.go:336` — "TODO: this assumes 1 utia per gas which may not be correct." |

### §4 — messages, queries, events

| # | AGENTS.md claim | Status | Evidence at v10.4.0 |
|---|---|---|---|
| 43 | `MsgDepositToEscrow` | confirmed | `proto/celestia/fibre/v1/tx.proto:19-24,55-61` (REST POST `/fibre/v1/deposit-to-escrow`) |
| 44 | `MsgRequestWithdrawal` | confirmed | `tx.proto:27-32,67-73` (POST `/fibre/v1/request-withdrawal`) |
| 45 | `MsgPayForFibre` — settlement, one per tx, ≤200/block | confirmed | `tx.proto:35-40,79-87` (POST `/fibre/v1/pay-for-fibre`); `pkg/appconsts/fibre_consts.go:6-7` — `MaxPayForFibreMessages = 200` ("maximum number of PayForFibre messages that a block can contain") |
| 46 | `MsgPaymentPromiseTimeout` | confirmed | `tx.proto:43-48,93-99` (POST `/fibre/v1/payment-promise-timeout`; signer "can be anyone") |
| 47 | gov `MsgUpdateFibreParams` | confirmed | `tx.proto:51,106-114` (authority = governance account) |
| 48 | `MsgSetFibreProviderInfo` | confirmed | `proto/celestia/valaddr/v1/tx.proto:15-21,24-30` (POST `/valaddr/v1/set-fibre-provider-info`; signer = validator operator address) |
| 49 | Query `/fibre/v1/params` | confirmed | `proto/celestia/fibre/v1/query.proto:17-19` |
| 50 | Query `/fibre/v1/escrow-account/{signer}` | confirmed | `query.proto:22-24` |
| 51 | Query `/fibre/v1/withdrawals/{signer}` | confirmed | `query.proto:27-29` (paginated) |
| 52 | Query `/fibre/v1/is-payment-processed/{hash}` | **DIFFERS (cosmetic)** — path param is `{promise_hash}` | `query.proto:33-35` — `get = "/fibre/v1/is-payment-processed/{promise_hash}"`, request field `bytes promise_hash = 1` |
| 53 | Query `/valaddr/v1/all-bonded-fibre-providers` | confirmed | `proto/celestia/valaddr/v1/query.proto:19-21` (bonded-only semantics `:16-18`) |
| 54 | Typed events `celestia.fibre.v1.Event{DepositToEscrow, WithdrawFromEscrowRequest, WithdrawFromEscrowExecuted, PayForFibre, PaymentPromiseTimeout, UpdateFibreParams, ProcessedPaymentPruned}` | confirmed | All 7 defined in `proto/celestia/fibre/v1/event.proto:13-58`; emitted via `EmitTypedEvent` — `x/fibre/keeper/msg_server.go:65,116,175,260,286` and `x/fibre/keeper/abci.go:118,163` |
| 55 | Plain event `set_fibre_provider_info{validator_consensus_address, host}` | confirmed | `x/valaddr/types/msg.go:13,16` — `EventTypeSetFibreProviderInfo = "set_fibre_provider_info"`, `AttributeKeyValidatorAddress = "validator_consensus_address"`; emitted plain via `EmitEvent` at `x/valaddr/keeper/msg_server.go:58-62`. (A typed `celestia.valaddr.v1.EventSetFibreProviderInfo` exists in proto but is **not** emitted at v10.4.0.) |

**Extras present at v10.4.0 but not listed in AGENTS.md §4** (additive, not conflicts): query `POST /fibre/v1/validate-payment-promise` (`query.proto:37-43`, returns `is_valid`, `expiration_time`, `shard_retention` — this is what the fibre server polls); query `GET /valaddr/v1/fibre-provider-info/{validator_consensus_address}` (`valaddr/v1/query.proto:12-14`); REST tx gateways for all five msgs (paths above); `MaxPromiseClockSkew = 10 * time.Minute` and derived `PaymentPromiseRetentionWindow = withdrawal_delay + 10m` (`x/fibre/types/params.go:38,103-105`); server-side `--unlimited-budget` kill switch (`fibre/server_config.go:106`).

---

## 2. Conflicts to fix in AGENTS.md or flag in plan

Ordered by impact on the spec/plan:

1. **§3 metric table mislabels two v10.4.0 metrics as "new on `main`".** `fibre.server.upload_shard.last_success_timestamp` and the whole `fibre.server.backend.get.*` family **ship in v10.4.0** (mainnet today) — `fibre/server_metrics.go:97-108,176-196`, documented in the v10.4.0 README and covered by tagged tests. Only `dupe_hits` is genuinely main-only (plus a new `fibre.server.store.backend` info-gauge on main that AGENTS.md doesn't mention). Fix: move `last_success_timestamp` and `backend.get.*` into the v10.4.0 baseline; keep `dupe_hits` + `store.backend` in the "main drift" bucket.
2. **§3 dashboard path is wrong (and upstream's own README link is broken).** `fibre/dashboards/fibre-dashboards.json` does not exist at `v10.4.0` **or** on `main` (404 both). Real dashboards at the tag: `observability/docker/grafana/dashboards/fibre.json` + `fibre-runtime.json`. Fix AGENTS.md §3 and §7-task-5 wording; the plan's "import official dashboard" step should use those two files.
3. **§3 "storage budget = 2 TiB × stake share" is a simplification.** Actual: `FullStakeStorageBudget × assignedRows/OriginalRows`, where `assignedRows = ceil(OriginalRows × votingPower × 3 / totalVotingPower)` clamped to `[MinRowsPerValidator, OriginalRows]` — i.e. ≈ **3× raw stake share** at `LivenessThreshold = 1/3`, quantized and clamped (`fibre/server.go:279-290`, `fibre/validator/set.go:108-113`). Matters for the "budget utilization" alert: predicted budget from stake alone will under-read by ~3×. The reliable source is the exported `fibre.server.upload_shard.budget_bytes` gauge itself.
4. **§3 "both plaintext, loopback only" is imprecise.** The app gRPC link has **no TLS support at all**; the signer link is plaintext-on-loopback **or mutual TLS for remote** (`signer_grpc_{ca,cert,key}_file`), with `signer_grpc_allow_insecure` as an explicit unsafe override. Fix for the data-handling/security page and the agent-path deployment docs.
5. **§4 minor: `/fibre/v1/is-payment-processed/{hash}` → `{promise_hash}`.** Cosmetic path-param rename for the chain-reference table.
6. **§4 "payment formula marked TODO" — refine, don't remove.** The TODOs (`x/fibre/keeper/keeper.go:446`, `msg_server.go:336`) question only the 1 gas = 1 utia conversion; the `650,000 + 45,000×⌈size/262,144⌉` gas formula itself is declared the single source of truth shared by keeper and client. Keep "treat the utia conversion as parametric", drop any implication the size formula is unsettled.
7. **§3 "~1m45s deletion lag" is not code-verifiable.** Keep as an empirical ops note only; do not bake it into alert thresholds as a constant (the code constant is the 1-minute prune interval).

Additive gaps worth folding into the spec/plan (not conflicts): `POST /fibre/v1/validate-payment-promise` and `GET /valaddr/v1/fibre-provider-info/{validator_consensus_address}` queries; REST tx gateways; the fibre binary also exports Go runtime metrics (`go.*`) when OTLP is enabled, and client-mode `fibre.client.*` series from the same binary; rejected-uploads carry a `RetryInfo` hint of one prune interval.

---

## 3. Appendix — metric surface at `v10.4.0` (adapter contract baseline)

Exact instruments registered by `fibre/server_metrics.go` at tag `v10.4.0` (file `newServerMetrics`, lines 59-226; attribute wiring lines 228-338). All names verbatim; units from `metric.WithUnit`; bucket boundaries from `WithExplicitBucketBoundaries`.

| Metric | Instrument | Unit | Attributes | Notes |
|---|---|---|---|---|
| `fibre.server.upload_shard.in_flight` | Int64 UpDownCounter | — | — | concurrent UploadShard RPCs |
| `fibre.server.upload_shard.duration` | Float64 Histogram | `s` | `success` (bool); `upload_size` (int64, only when > 0) | buckets 0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5 |
| `fibre.server.upload_shard.bytes` | Int64 Counter | `By` | — | shard bytes received |
| `fibre.server.upload_shard.rejected` | Int64 Counter | — | `reason` (string) | only value at tag: `budget_exceeded` (`fibre/server_upload.go:106`) |
| `fibre.server.upload_shard.last_success_timestamp` | Int64 ObservableGauge | `s` | — | unix time of last successful upload; **not emitted before first success** |
| `fibre.server.upload_shard.occupancy_bytes` | Int64 ObservableGauge | `By` | — | limiter-tracked bytes = on-disk + in-flight reserved (not a disk gauge) |
| `fibre.server.upload_shard.budget_bytes` | Int64 ObservableGauge | `By` | — | current per-node derived budget |
| `fibre.server.download_shard.in_flight` | Int64 UpDownCounter | — | — | concurrent DownloadShard RPCs |
| `fibre.server.download_shard.duration` | Float64 Histogram | `s` | `success` (bool); `shard_size` (int64, only when > 0) | buckets same as upload duration |
| `fibre.server.download_shard.bytes` | Int64 Counter | `By` | — | shard bytes sent |
| `fibre.server.store.put.duration` | Float64 Histogram | `s` | `success` (bool) | buckets 0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1 |
| `fibre.server.store.get.duration` | Float64 Histogram | `s` | `success` (bool) | buckets as store.put |
| `fibre.server.backend.get.duration` | Float64 Histogram | `s` | `backend` (`local`\|`object`), `outcome` (`success`\|`not_found`\|`timeout`\|`canceled`\|`throttled`\|`error`) | buckets 0.001…60 (adds 2.5, 5, 10, 30, 60); primary backend only |
| `fibre.server.backend.get.in_flight` | Int64 UpDownCounter | — | `backend` | |
| `fibre.server.backend.get.bytes` | Int64 Counter | `By` | `backend` | encoded bytes incl. partial failures/buffered reads |
| `fibre.server.sign.duration` | Float64 Histogram | `s` | `success` (bool) | buckets 0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1 — p50 SLO 10 ms sits on the 0.01 bucket edge |
| `fibre.server.prune.entries` | Int64 Counter | — | — | incremented only when `pruned > 0` |
| `fibre.server.prune.duration` | Float64 Histogram | `s` | `success` (bool) | buckets 0.001, 0.01, 0.05, 0.1, 0.5, 1, 5, 10 |

Also exported when OTLP is enabled (`fibre/cmd/metrics.go:57`): Go runtime series from `go.opentelemetry.io/contrib/instrumentation/runtime` (`go.*`), and — from the same binary in client mode — the `fibre.client.*` family documented in the v10.4.0 README:262-278 (e.g. `fibre.client.upload.duration`, `fibre.client.download_from.rpc_latency`). The adapter should expect all three namespaces from one endpoint.

OTLP framing at v10.4.0: OTLP/HTTP protobuf push to `<base>/v1/metrics` (periodic reader) and `<base>/v1/traces`; resource attributes `service.name="fibre"`, `service.version=<release>`, `service.instance.id=<hostname>` on **both** signals; exemplars on histogram datapoints; 10% parent-based trace sampling.

### Drift on `main` (post-v10.4.0, for adapter versioning)

| Metric | Instrument | Attributes | Evidence on `main` |
|---|---|---|---|
| `fibre.server.upload_shard.dupe_hits` | Int64 Counter | `stage` (string) | `main:fibre/server_metrics.go:107` ("skipped storing because the shard was already stored, by detection stage"); recorded `main:fibre/server_upload.go:206-207` |
| `fibre.server.store.backend` | Int64 ObservableGauge | `backend` (string), `object_configured` (bool); constant value 1 | `main:fibre/server_metrics.go:246-258` ("Configured primary shard storage backend") |

Everything else in the baseline table is unchanged on `main` as of 2026-10-21.

---

## 4. Reproduction notes

- Tree listing: `https://api.github.com/repos/celestiaorg/celestia-app/git/trees/v10.4.0?recursive=1` (1526 paths, not truncated).
- Source: release tarball `https://github.com/celestiaorg/celestia-app/archive/refs/tags/v10.4.0.tar.gz` + individual `raw.githubusercontent.com/celestiaorg/celestia-app/{v10.4.0,main}/<path>` fetches.
- All line numbers refer to the tagged files; quotes are verbatim.
