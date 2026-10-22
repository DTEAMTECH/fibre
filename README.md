<p align="center">
  <img src="docs/assets/fibre-monitoring-hero.png" alt="Fibre Monitoring by DTEAMTECH — self-hosted Grafana monitoring for Celestia Fibre" width="100%" />
</p>

# Celestia Fibre Monitoring

**Your Fibre server, traffic and storage in Grafana — with optional network exploration from your own FibreStats instance.**

A self-hosted monitoring bundle by **DTEAMTECH**. Run it with Docker Compose, connect your existing Fibre service and explore its telemetry. Enable alerts when you want them; choose where notifications go.

[Quick start](#quick-start) · [Dashboard](#dashboard) · [FibreStats integration](#optional-fibrestats-integration) · [Optional alerts](#optional-alerts) · [Configuration](#configuration) · [Troubleshooting](#troubleshooting)

---

## Key features

| | What you can see |
|---|---|
| **Traffic** | Upload/download throughput, payload volume, request counts and latency percentiles |
| **Storage** | Occupancy versus budget, rejected uploads, backend errors and prune cycles |
| **Signing** | Payment-promise signing attempts, duration and failures |
| **Runtime** | Go memory and goroutines, alongside server telemetry presence |
| **Network view, optional** | Provider reachability, TCP/TLS latency, recorded endorsements and network reconstruction from your own FibreStats observer |
| **Exploration, optional** | Settlements, recorded paid bytes, unique blobs, namespaces, publishers and infrastructure changes inside Grafana |
| **Optional alerts** | Budget pressure, request failures, signing/storage errors and missing telemetry |
| **Local control** | Persistent storage, provisioned dashboards, pinned images and private service ports |

> [!NOTE]
> The quick start runs the local telemetry stack. Enable the [FibreStats integration](#optional-fibrestats-integration) to add measurements and chain records collected by an instance you operate. Place that observer on another machine/network for a useful outside view of your Fibre server.

## Quick start

### 1. Start the stack

You need Docker with the Compose plugin and a running Fibre server. These instructions assume Fibre runs directly on the same host as Docker. The metric contract is based on celestia-app **v10.4.0**; other releases need verification.

Clone the repository and enter the monitoring directory:

```bash
git clone https://github.com/DTEAMTECH/fibre-monitoring.git
cd fibre-monitoring/monitoring
docker compose up -d
```

Open **http://localhost:3000**. Sign in with `admin` / `admin` and change the password when prompted. The **Fibre Server — Operator** dashboard and **Prometheus** data source are already installed.

For a password of your choice before the first start, create `.env` in the current directory (`fibre-monitoring/monitoring/`):

```dotenv
GRAFANA_ADMIN_PASSWORD=your-long-password
```

Then start the stack. `.env` is ignored by Git. The password setting only initializes a new Grafana database; changing it later does not reset an existing account.

| Service | Local address | Purpose |
|---|---|---|
| Grafana | http://localhost:3000 | Dashboard |
| Prometheus | http://localhost:9091 | Queries, scrape status and optional alert rules |
| OTLP receiver | http://127.0.0.1:4318 | Fibre pushes telemetry here |
| Alertmanager, if enabled | http://localhost:9093 | Alert groups and silences |

Ports bind to `127.0.0.1`. The collector's metrics endpoint is available only inside the Compose network. Prometheus uses host port **9091** to avoid the application gRPC service commonly running on **9090** on Celestia validator hosts.

If this is a remote server, use an SSH tunnel from your laptop:

```bash
ssh -L 3000:127.0.0.1:3000 user@your-server
```

Then open http://localhost:3000 on your laptop.

### 2. Connect your existing Fibre service

Fibre has no Prometheus scrape endpoint. It pushes OTLP/HTTP to a collector. Add this flag to your existing start command, keeping your current home, node and signer configuration:

```bash
--otel-endpoint http://127.0.0.1:4318
```

Alternatively, set the environment variable used by your service:

```dotenv
FIBRE_OTEL_ENDPOINT=http://127.0.0.1:4318
```

For a systemd service named `fibre`, open an override:

```bash
sudo systemctl edit fibre
```

Add:

```ini
[Service]
Environment="FIBRE_OTEL_ENDPOINT=http://127.0.0.1:4318"
```

If the existing command already has `--otel-endpoint`, update that flag too. Apply the change during your normal maintenance window:

```bash
sudo systemctl daemon-reload
sudo systemctl restart fibre
```

Use the **base URL**, without `/v1/metrics`. Fibre appends its metrics and traces paths. This stack accepts traces and discards them; no trace database is needed.

Fibre normally exports every 60 seconds. Allow about two minutes for the first readings. Occupancy and budget should appear even on an idle server; request latency and byte counters need actual traffic.

**If Fibre is in a container:** `127.0.0.1` is that container, not the host. Connect it to the monitoring network and use `http://collector:4318`, or supply another private route. If Fibre runs on another machine, use a private network or tunnel and bind the collector to the required private interface; the default loopback mapping deliberately accepts only local traffic. No receiver authentication is configured.

### 3. Check that data arrives

```bash
docker compose ps
docker compose logs --tail 50 collector
```

Open http://localhost:9091/targets. The `fibre` job should be **UP**. This proves that Prometheus can scrape the collector; it does not by itself prove Fibre is sending data.

Run this query in Prometheus to check the server stream:

```promql
fibre_server_upload_shard_occupancy_bytes
```

Or query from the terminal:

```bash
curl --get http://127.0.0.1:9091/api/v1/query \
  --data-urlencode 'query=fibre_server_upload_shard_occupancy_bytes'
```

In Grafana, select your server in **Server instance**. The identity comes from Fibre's hostname (`service.instance.id`), so use unique hostnames if you send multiple servers into one collector.

An idle network is healthy. Empty request panels before the first request are expected. The collector removes cached series after two minutes without fresh telemetry; the availability alert adds another five-minute grace period. One missed export may therefore be tolerated, and detection is not immediate.

## Dashboard

![Fibre Grafana dashboard showing synthetic demonstration data](docs/assets/fibre-dashboard-synthetic.png)

*Actual Grafana UI with synthetic demonstration data. No live validator readings are shown.*

### Read the traffic panels correctly

- **Upload/download throughput:** shard-row payload bytes per second handled by this server.
- **Payload volume:** estimated counter increase across the selected time range.
- **Completed RPCs:** upload/download attempts, including failures; these are not unique blob counts.
- **Latency percentiles:** server handler timing, not internet ping or end-to-end client latency.
- **In-flight RPCs:** concurrent operations, not total network connections or configured connection capacity.

Successful duplicate uploads count again in upload bytes. Download bytes are recorded before returning the response and do not prove full client receipt. Neither counter includes transport overhead or measures interface bandwidth. The optional FibreStats dashboard adds network settlement and blob accounting; host bandwidth needs host metrics.

Storage occupancy includes limiter reservations. It is separate from physical disk usage. Signing latency measures Fibre payment-promise signing/waiting and has no universal 10 ms threshold. [Exact-tag instruments](https://github.com/celestiaorg/celestia-app/blob/v10.4.0/fibre/server_metrics.go), [upload byte accounting](https://github.com/celestiaorg/celestia-app/blob/v10.4.0/fibre/server_upload.go), [download byte accounting](https://github.com/celestiaorg/celestia-app/blob/v10.4.0/fibre/server_download.go).

## Optional FibreStats integration

Add the **Fibre Network — Explorer** dashboard alongside the operator dashboard. FibreStats runs the probes and blockchain reader; this stack reads its stored metrics and bounded JSON APIs. Grafana queries do not launch probes or paid uploads. The integration uses an instance you run, with no FibreStats account or token.

| View | What it explains |
|---|---|
| Sources and coverage | Scrape health, collection freshness, indexed boundaries and incomplete history |
| Providers | Registered endpoints, independent DNS/TCP/TLS/identity stages, latest measured latency and probe location |
| Endorsements | Recorded signature inclusions, known historical membership denominator and unresolved attribution |
| Network traffic | Settlements, recorded paid bytes and unique blobs over a fixed window |
| Exploration tables | Recent settlements, namespace/publisher breakdowns and infrastructure changes |
| Reconstruction | Network-level verified reconstruction outcomes and measurement coverage |

### 1. Run your own FibreStats observer

Deploy [FibreStats](https://github.com/zeroiwobro/fibrestats) on the monitoring/probe host using its [deployment guide](https://github.com/zeroiwobro/fibrestats/blob/main/docs/MONITORING-INTEGRATION.md). Its web service and collector need one shared **local persistent database**. Keep the collector running; the web service alone cannot refresh observations. Keep an existing database's `FIBRESTATS_VANTAGE` unchanged.

For a fresh source checkout, configure `.env` in the FibreStats directory, then start its Compose stack:

```dotenv
FIBRESTATS_NETWORKS=Mainnet,Mocha
FIBRESTATS_VANTAGE=probe-host-1
```

```bash
docker compose up -d --build
curl -fsS 'http://127.0.0.1:3000/api/integration/status?network=Mocha'
```

Configure chain/archive/index endpoints in FibreStats if needed. Historical endorsement attribution depends on historical membership and transaction availability; a working current endpoint does not guarantee complete history. Probe history starts when your observer runs.

### 2. Connect the monitoring stack

Run both Compose projects on the same monitoring/probe host. FibreStats creates the private `fibrestats_default` network. In **this repository's `monitoring/.env`**, add these settings alongside any existing password/notification configuration:

```dotenv
# FibreStats already uses host port 3000; Grafana gets its own port.
GRAFANA_PORT=3002
COMPOSE_FILE=compose.yaml:compose.fibrestats.yaml
FIBRESTATS_DOCKER_NETWORK=fibrestats_default
FIBRESTATS_API_URL=http://web:3000
```

From `fibre-monitoring/monitoring/`:

```bash
docker compose config --quiet
docker compose up -d
```

Open **http://localhost:3002** and choose **Fibre Network — Explorer** in the **Fibre** folder. The first start installs the pinned Grafana Infinity plugin for JSON tables; it needs access to Grafana's plugin registry. The normal telemetry mode does not install this plugin. Prometheus reads `/api/metrics` once a minute; Grafana reads the JSON API through its backend.

Both use FibreStats' **private `web:3000` service**. Its production Caddy deliberately returns 404 for `/api/metrics*` and `/api/integration*`. Do not set the API URL to the public website. If you use another private API address, update both `FIBRESTATS_API_URL` and [`monitoring/fibrestats/targets.yml`](monitoring/fibrestats/targets.yml); the target is `host:port`, without a scheme or path. Keep the API private and run any required tunnel independently.

The integration targets FibreStats [contract v1 at `7ca781b`](https://github.com/zeroiwobro/fibrestats/commit/7ca781bfb4bc28e2f55f866084135fab3c668792). Fibre OTLP stays on the local/private collector path described in the quick start.

### 3. Read the outside view correctly

- Select **Network**, **Chain window**, **Provider** and **Probe location** in the dashboard. Activity windows are `24h`, `7d` and `30d`; changing Grafana's time picker changes metric history, not the JSON API's fixed window.
- Check source freshness and coverage before interpreting totals. HTTP scrape success does not prove the collector or chain reader is healthy. Unknown data stays empty; a covered idle window can be zero.
- TCP connection and TLS handshake durations are individual saved measurements, not ICMP ping or latency histograms. Stale probe verdicts are unavailable rather than successful.
- Endorsement participation means recorded inclusions among settled promises with known historical membership. Clients can stop after quorum, so an absent endorsement is not proof of a missed duty or downtime. Unresolved memberships stay outside the provider denominator.
- Recorded paid bytes, settlements and unique blobs measure different things. These are finite-window gauges: do not apply `rate()` to them. Chain windows end at the indexed boundary; probe windows use observation time.
- Exploration tables show a bounded first page, with their row limits in the panel descriptions. Exact monetary values remain strings. Use FibreStats for deeper pagination and history; network reconstruction does not establish per-provider shard availability.

Private OTLP server identity and public provider identity are separate selectors. Match your server explicitly to `(network, provider)` using the provider's canonical consensus-key-derived ID; hostnames and monikers do not establish that mapping.

## Optional alerts

The normal stack evaluates **no alert rules**. To enable the rule pack and local Alertmanager:

```bash
docker compose -f compose.yaml -f compose.alerts.yaml up -d
```

For the FibreStats mode, change `COMPOSE_FILE` in `monitoring/.env` to include alerting, then use `docker compose up -d`:

```dotenv
COMPOSE_FILE=compose.yaml:compose.fibrestats.yaml:compose.alerts.yaml
```

Explicit `-f` flags override `COMPOSE_FILE`; include every enabled overlay when using them.

Alerts are visible at http://localhost:9091/alerts and http://localhost:9093. **No notifications are sent until you configure a receiver.**

| Check | Default trigger |
|---|---|
| Collector scrape | Down for 2 minutes |
| Fibre telemetry | All server occupancy gauges absent for 5 minutes, after cache expiration |
| Storage budget | Above 80% for 15 minutes; above 95% for 5 minutes |
| Budget rejections | Any increase in the last 15 minutes |
| Upload/download failures | More than 10% of at least 20 completed RPCs in 15 minutes, sustained for 5 minutes |
| Signing, pruning or backend errors | At least 3 errors in 15 minutes |
| Uploads without recent success | Successful uploads stale while that same server continues completing upload attempts |

These thresholds are starting points. Adjust [`monitoring/prometheus/fibre_alerts.yml`](monitoring/prometheus/fibre_alerts.yml) to your workload. No uploads, zero prune deletions, or a missing settlement endorsement alone are not failures.

The current missing-telemetry rule is a whole-stack check. With several servers, it does not detect one disappearing while another keeps reporting; that requires an explicit expected-server inventory.

With both FibreStats and alerts enabled, an additional pack checks API scrape failure and required source health. You can opt in to provider checks for fresh measured endpoint failure, unavailable current probe verdicts and TCP connection latency above **500 ms for 10 minutes**. Provider checks require successful scraping and fresh chain/observer/materializer sources. Tune [`monitoring/prometheus/fibrestats_alerts.yml`](monitoring/prometheus/fibrestats_alerts.yml) for your probe location. No endorsement-absence or idle-traffic alert is installed.

To watch your own provider, uncomment `labels` in [`monitoring/fibrestats/targets.yml`](monitoring/fibrestats/targets.yml) and set its network and canonical provider ID from the dashboard:

```yaml
- targets: [web:3000]
  labels:
    alert_network: Mocha
    alert_provider: "YOUR_LOWERCASE_40_CHARACTER_HEX_PROVIDER_ID"
```

Prometheus refreshes this target file automatically. All providers remain visible in Grafana; only the selected provider can produce endpoint/latency notifications. Without these labels, only collection/API checks from this pack run. These labels select alert scope; they do not map an OTLP hostname to a validator.

### Optional notifications

Alertmanager supports Telegram, Discord, webhooks and other receivers. Keep credentials in a local file:

```bash
cp alertmanager/alertmanager.yml alertmanager/alertmanager.local.yml
```

Edit that new file. For example, replace its receiver configuration with your Telegram bot and chat ID:

```yaml
route:
  receiver: telegram
  group_by: [alertname, service_instance_id, network, provider, vantage]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: telegram
    telegram_configs:
      - bot_token: 'YOUR_BOT_TOKEN'
        chat_id: -1001234567890
        send_resolved: true
```

Add this setting to `.env`, keeping any existing password settings:

```dotenv
ALERTMANAGER_CONFIG=./alertmanager/alertmanager.local.yml
```

The local receiver file is ignored by Git. Validate it before applying:

```bash
docker run --rm --entrypoint amtool \
  -v "$PWD/alertmanager/alertmanager.local.yml:/config.yml:ro" \
  prom/alertmanager:v0.33.0 check-config /config.yml

docker compose -f compose.yaml -f compose.alerts.yaml up -d --force-recreate alertmanager
```

Use the same enabled overlays whenever starting or updating this mode. For FibreStats, `docker compose exec ...` and `docker compose restart prometheus` use the `COMPOSE_FILE` settings above. In basic telemetry mode, validate and restart Prometheus after rule changes:

```bash
docker compose -f compose.yaml -f compose.alerts.yaml exec prometheus \
  promtool check rules /etc/prometheus/fibre_alerts.yml

docker compose -f compose.yaml -f compose.alerts.yaml restart prometheus
```

Notification delivery depends on your credentials and routing. Test it with controlled synthetic/devnet data; do not stop a live validator to test an alarm. See [Alertmanager receiver configuration](https://prometheus.io/docs/alerting/latest/configuration/) for other channels.

To return to dashboards without alerts, first remove `compose.alerts.yaml` from `COMPOSE_FILE` if set, retaining the other overlays you use. Then run:

```bash
docker compose up -d --remove-orphans
```

This removes the optional Alertmanager container and switches Prometheus back to its normal configuration. Stored metrics remain.

## Configuration

All commands below run from `fibre-monitoring/monitoring/`. Override these defaults in `.env`:

```dotenv
GRAFANA_PORT=3000
PROMETHEUS_PORT=9091
OTLP_PORT=4318
ALERTMANAGER_PORT=9093
PROMETHEUS_RETENTION=30d
```

If you change `OTLP_PORT`, update Fibre's endpoint too. Run the appropriate `docker compose ... up -d` command after changes.

Prometheus and Grafana use named Docker volumes. `docker compose down` stops the normal stack without deleting those volumes. When alerts are enabled, use `docker compose -f compose.yaml -f compose.alerts.yaml down` to stop the full stack. Avoid `down -v` unless you intentionally want to erase stored metrics, dashboards and accounts. Retention is time based; available disk space still needs monitoring.

Image versions are pinned in the Compose files. Test version upgrades with the configuration checks below before applying them.

<details>
<summary><strong>Use an existing Cosmos monitoring stack</strong></summary>


The dashboard JSON, collector config and alert rules are reusable independently. Run the collector where Fibre can reach it, add a `fibre` scrape job for its private `:8889` endpoint, and import [`monitoring/dashboards/fibre-operator.json`](monitoring/dashboards/fibre-operator.json) into your existing Grafana. Select your Prometheus data source. Only enable the rule file if you want these alerts.

If Prometheus is in a different Compose project, connect the services through a shared private network or explicitly publish the collector's metrics endpoint on an appropriate private interface. `localhost` inside a container refers to that container. Keep job name `fibre` for the bundled scrape-health rule. Your existing consensus monitoring can continue handling block signatures.

</details>

## Troubleshooting

| Symptom | What to check |
|---|---|
| Grafana is reachable, but panels are empty | Confirm the `fibre` target is UP, query the occupancy gauge in Prometheus, then check the selected data source and server. Allow two minutes. |
| Only request panels are empty | There may not have been any uploads/downloads. Periodic occupancy is a better connection check. |
| Fibre reports OTLP 404 errors | Remove `/v1/metrics` from the endpoint. Use the collector base URL. |
| The collector starts, but the occupancy query is empty | Confirm the endpoint on the running Fibre service; inspect Fibre's exporter logs and container/private-network routing. |
| A port is already allocated | Change that service's host port in `.env`; keep internal service ports unchanged. |
| Budget graph is empty or unbounded | Confirm the exported budget; a zero/non-positive budget disables enforcement and should not be interpreted as exhaustion. |
| Alerts fire but no messages arrive | The default receiver is local only. Check your receiver config, credentials, Alertmanager logs and silences. |
| Network/endorsement panels are missing | Enable the FibreStats overlay and open **Fibre Network — Explorer**. Local server OTLP does not supply outside-view data. |
| `fibrestats_default` network cannot be found | Start your FibreStats Compose project first; set `FIBRESTATS_DOCKER_NETWORK` if its network name differs. |
| FibreStats scrape or JSON tables return 404 | Use the private web service address. The public Caddy site intentionally blocks these APIs. Keep the scrape target and JSON base URL on the same instance. |
| JSON panels report a missing plugin | Check Grafana startup logs and access to the plugin registry; the optional mode installs pinned Infinity before provisioning. |
| A network panel says no data or unknown | Check the enabled network, source health, record age and history coverage. An empty provider selector needs a collected provider inventory first. |
| No endpoint alerts for a provider | Enable both overlays and configure `alert_network` and `alert_provider` in `fibrestats/targets.yml`. Check fresh observer/chain/materializer sources too. |

<details>
<summary><strong>Project files and validation commands</strong></summary>


| Path | Purpose |
|---|---|
| `compose.yaml` | Core local stack |
| `compose.alerts.yaml` | Optional alerting services/config |
| `compose.fibrestats.yaml` | Optional private FibreStats scraping and Grafana exploration |
| `otelcol/otelcol.yaml` | Receive OTLP, expose private Prometheus metrics, discard traces |
| `dashboards/fibre-operator.json` | Grafana operator and traffic dashboard |
| `grafana/provisioning/` | Automatic data source and dashboard setup |
| `prometheus/prometheus.yml` | Default scrapes, without alerts |
| `prometheus/prometheus-alerts.yml` | Scrapes, rules and Alertmanager routing |
| `prometheus/fibre_alerts.yml` | Tunable alert pack |
| `prometheus/tests.yml` | Synthetic rule regression cases |
| `prometheus/fibrestats_alerts.yml` | Optional source health and selected-provider alerts |
| `prometheus/fibrestats_tests.yml` | Synthetic outside-view rule regression cases |
| `fibrestats/targets.yml` | Private metrics target and opt-in provider alert identity |
| `fibrestats/dashboards/fibre-explorer.json` | Provider, traffic, endorsement and JSON exploration dashboard |
| `fibrestats/provisioning/` | Optional Infinity datasource and dashboard provisioning |
| `alertmanager/alertmanager.yml` | Local-only receiver default |

Paths in the table are relative to `monitoring/`. Run validation commands from that directory:

```bash
docker compose config --quiet
docker compose -f compose.yaml -f compose.alerts.yaml config --quiet
docker compose -f compose.yaml -f compose.fibrestats.yaml config --quiet
docker compose -f compose.yaml -f compose.fibrestats.yaml -f compose.alerts.yaml config --quiet
docker compose run --rm collector validate --config=/etc/otelcol-contrib/config.yaml

docker run --rm --entrypoint promtool \
  -v "$PWD/prometheus:/work:ro" -w /work \
  prom/prometheus:v3.5.4 check rules fibre_alerts.yml

docker run --rm --entrypoint promtool \
  -v "$PWD/prometheus:/work:ro" -w /work \
  prom/prometheus:v3.5.4 test rules tests.yml

docker run --rm --entrypoint promtool \
  -v "$PWD/prometheus:/work:ro" -w /work \
  prom/prometheus:v3.5.4 test rules fibrestats_tests.yml
```

Configuration and synthetic pipeline checks have passed. Real Fibre devnet/load testing and notification delivery with operator credentials remain to be verified. The [official Fibre monitoring guide](https://docs.celestia.org/operate/consensus-validators/fibre/metrics/) provides additional protocol context.

</details>

---

## Roadmap

Next additions focus on physical disk, I/O and interface throughput, explicit private-server/provider identity mappings, and deeper exploration pagination. Per-provider retained-shard verification requires separate evidence; network reconstruction alone does not establish that a particular server served its shard.

## Contributing

Changes and operator feedback are welcome through [issues](https://github.com/DTEAMTECH/fibre-monitoring/issues) and pull requests. Report the Fibre version, network, relevant configuration and reproducible symptoms; keep credentials and private information out of reports.
