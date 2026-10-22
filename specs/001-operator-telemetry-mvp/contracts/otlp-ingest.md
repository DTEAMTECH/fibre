# Contract: OTLP Ingest Endpoint

**Audience**: the fibre server binary (unmodified) and the published agent (Alloy/OTel Collector).
**Implements**: FR-001…FR-005, FR-011, edge cases (token failures, clock skew, duplicates, floods, non-metric payloads).

## 1. Why auth is token-in-path

The fibre binary's only OTLP configuration is `--otel-endpoint` (env `FIBRE_OTEL_ENDPOINT`). It appends `/v1/metrics` and `/v1/traces` to the base URL, **preserving any path prefix**, and supports **no custom headers**. Therefore the per-tenant credential travels as a URL path segment — the only mechanism the unmodified binary supports.

## 2. Endpoint shape

| Purpose | URL | Method | Body |
|---|---|---|---|
| Metrics ingest | `https://otel.<domain>/t/{ingest_token}/v1/metrics` | POST | OTLP `ExportMetricsServiceRequest`, protobuf (`Content-Type: application/x-protobuf`) |
| Traces (accepted → discarded) | `https://otel.<domain>/t/{ingest_token}/v1/traces` | POST | OTLP `ExportTraceServiceRequest` |
| Agent path (Alloy/Collector push) | same `/v1/metrics` shape; bearer header `Authorization: Bearer fs_ingest_…` also accepted | POST | OTLP or Prometheus remote-write (agent config pins OTLP) |

- `{ingest_token}`: `fs_ingest_` + 32-byte URL-safe base64. Exact-match lookup by SHA-256 hash; unknown/revoked → rejection below.
- Self-host simple mode MAY instead use static bearer tokens configured on the collector (`bearertokenauth`); the contract is otherwise identical (research R2).

## 3. Request processing (edge, in order)

1. **Token extraction & validation** — path segment (or bearer header for agents). Missing/unknown/revoked → reject; log `RejectedPushEvent` (FR-004).
2. **Rate/cardinality guard** — per-tenant limits: request body ≤ 8 MiB; ≤ 5,000 active series per server; burst ≤ 30 requests/min sustained 2/min. Breach → `429` + quarantine flag to the tenant; other tenants unaffected (SC-004, flood edge case).
3. **Content enforcement (metrics-only, FR-005)** — only metrics payloads are decoded and stored. Traces: accepted, responded `200`, discarded. Anything else (logs, arbitrary blobs, non-OTLP bodies) → `415`/`400`, discarded, never stored or displayed.
4. **Tenant attribution (FR-003)** — inject resource attribute `tenant_id=<uuid>` and resolve `server_id` from `service.instance.id` (+`service.name=fibre` expected; other `service.name` values accepted under the agent path with `namespace` labeling). Version/classification computed from `service.version` (research R6).
5. **Timestamp policy (clock-skew edge case)** — points in `[now−7d, now+10m]` accepted as-is; future > +10 min clamped to receipt time and flagged; older than −7 d dropped and flagged. Both flags appear in the tenant's ingestion-status view; one bad clock cannot corrupt a timeline.
6. **Deduplication** — OTLP cumulative temporality makes retried deliveries idempotent for counters/up-downs; identical (series, timestamp) points are last-write-wins in storage. Duplicate pushes never double-count budgets or counters.
7. **Forward** — batch to the OTel Collector (localhost OTLP) → VictoriaMetrics with labels `{tenant_id, server_id, namespace, …original attributes}`.

## 4. Responses

| Code | Meaning | Body |
|---|---|---|
| 200 | accepted (incl. traces-accepted-then-discarded; partial success per OTLP spec) | `ExportMetricsServiceResponse` (empty or `partial_success`) |
| 400 | malformed OTLP/protobuf | `{ "error": "malformed_payload" }` |
| 401 | missing / unknown / revoked token | `{ "error": "unauthorized", "reason": "missing_token\|unknown_token\|revoked_token" }` |
| 413 | body too large | `{ "error": "payload_too_large" }` |
| 415 | non-protobuf content type | `{ "error": "unsupported_media_type" }` |
| 429 | rate/cardinality limit (per-tenant only) | `{ "error": "rate_limited", "retry_after_seconds": n }` |
| 5xx | edge/storage fault — clients SHOULD retry with backoff (the fibre binary retries via the OTel SDK) | OTLP error body |

Every 4xx/429 is recorded as a `RejectedPushEvent` (tenant-visible, 30-day retention) with the token prefix and reason.

## 5. What the operator sees (US1 acceptance)

- First accepted point → server `connected` within 1 minute, with reported `service.version` and support classification (`full` / `partial` / `unknown`).
- Revoked-token pushes → rejected from the next attempt (edge cache TTL ≤ 60 s); recent rejections listed in the tenant UI for debugging.
- Multiple servers on one token → one `MonitoredServer` per distinct `service.instance.id` (US1.4); duplicate instance identities → `conflict` state surfaced, never merged.

## 6. Non-goals (explicit)

- No pull/scrape of operator infrastructure, ever.
- No storage of traces, logs, blob payloads, or key material; no read access granted by ingest tokens.
- Chain-side queries are NOT part of this contract; annotations (RegistrySnapshot) use separate read-only REST calls (`/valaddr/v1/...`, `/fibre/v1/...`) documented in data-model.md.
