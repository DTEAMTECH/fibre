# Contract: Alert Notification Channels

**Implements**: FR-018, FR-020, FR-021, US3 acceptance scenarios, SC-003 (95% delivered < 5 min), SC-010 (100% resolution notices).
**Transport owner**: Grafana Alerting contact points (research R10/R11); the service records every attempt as a `DeliveryRecord` and surfaces channel health in the web app.

## 1. Common alert payload (semantic contract for all channels)

```json
{
  "incident_id": "uuid",
  "kind": "firing | repeat | resolved | test",
  "rule": { "pack_key": "budget_utilization", "name": "Storage budget utilization", "severity": "warning" },
  "server": { "instance_id": "validator-eu-1", "reported_version": "v10.4.0", "network": "mainnet" },
  "fired_at": "2026-10-21T03:45:49Z",
  "resolved_at": null,
  "triggering_values": { "utilization": 0.83, "occupancy_bytes": 1.83e12, "budget_bytes": 2.2e12 },
  "summary": "Storage budget utilization 83% (threshold 80%) on validator-eu-1",
  "links": { "incident": "https://app.<domain>/incidents/<id>", "dashboard": "https://grafana.<domain>/d/..." }
}
```

Every payload explains **what fired and why** (US3); resolution payloads include the final observed values. Throttle: repeat ≤ 1 per `repeat_interval` (default 4 h) per incident per channel; resolution always sent (SC-010).

## 2. Channel types

### 2.1 Telegram

| Config field | Notes |
|---|---|
| `bot_token` | write-only, encrypted at rest; validated via `getMe` on save |
| `chat_id` | operator's chat/group id |
| `parse_mode` | default `MarkdownV2` (falls back to plain on entity errors) |

Delivery: Bot API `sendMessage`; message = summary + severity icon + triggering values + incident link. Failure handling per §3 (401/403 → immediate `failing`: revoked bot token or blocked bot).

### 2.2 Discord

| Config field | Notes |
|---|---|
| `webhook_url` | `https://discord.com/api/webhooks/...`; write-only, encrypted |

Delivery: webhook POST with one embed (title = summary, color by severity — brand palette: Indigo `#5640D1` firing, Cool Gray resolved; **no Celestia logos/marks** in avatar or embeds), fields = triggering values, footer = server + version.

### 2.3 Generic webhook

| Config field | Notes |
|---|---|
| `url` | https only (http allowed for self-host loopback) |
| `secret` | optional; enables `X-FibreStats-Signature: sha256=<hmac>` over the raw body |

Delivery: POST, `Content-Type: application/json`, body = common payload §1 verbatim. Response 2xx = success. Retries per §3 with `X-FibreStats-Attempt: n`.

### 2.4 PagerDuty

| Config field | Notes |
|---|---|
| `routing_key` | Events API v2 integration key; write-only, encrypted |
| `severity_map` | optional override of pack severity → PD severity (`critical→critical`, `warning→warning`, `info→info`) |

Delivery: Events API v2 `trigger` on firing, `resolve` on resolution with the same `dedup_key` (= `incident_id`); payload carries the common fields under `custom_details`.

## 3. Delivery, retries, channel health (FR-021)

- Attempts: up to 3 per notification, backoff 30 s / 2 min / 10 min (fits SC-003's 5-minute budget for the first two).
- After 3 failures: channel `health = failing`, `last_error` = sanitized transport error (never tokens/URLs with secrets); the tenant sees the failure in-app and on the incidents view (US3.3) — alerting gaps are never silent.
- Any subsequent success (including `POST /api/channels/{id}/test`) resets `health = ok`.
- `DeliveryRecord` kept per (incident, channel): kind, status, attempts, sanitized response, timestamp — queryable via `GET /api/incidents/{id}`.

## 4. Operator-facing guarantees

- Per-rule channel assignment (FR-018): an incident notifies exactly the channels on its rule's `channel_ids`.
- Disabled rules notify nothing (US3.2); agent-gated rules declare unavailability instead of firing vacuously (FR-022).
- Telemetry-silence notifications (rule 11) are visually distinct from threshold alerts ("telemetry stopped" ≠ "metric breached").
- Credentials: operators bring their own bots/webhooks/PD accounts (spec assumption); secrets are write-only, encrypted at rest, never rendered after save.
