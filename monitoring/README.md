# Fibre server monitoring — integrate with your existing Prometheus/Grafana stack

Standalone monitoring bundle for a single Celestia Fibre server operator (celestia-app v10+, verified against v10.4.0). The fibre binary has no Prometheus endpoint — it only pushes OTLP — so this bundle puts a small OTel Collector in front of your existing stack and adds a ready dashboard plus optional alert rules.

**Files in this bundle**

| File | What it is |
|---|---|
| `monitoring/otelcol/otelcol.yaml` | OTel Collector config — receives OTLP/HTTP on `:4318`, exposes Prometheus metrics on `:8889` |
| `monitoring/dashboards/fibre-operator.json` | Grafana dashboard (datasource input `DS_PROMETHEUS`, `$instance` filter variable) |
| `monitoring/prometheus/fibre_alerts.yml` | Optional Prometheus alert rules |

Metrics only. No keys, no blob payloads leave your host.

## Architecture

1. The fibre binary pushes metrics over OTLP/HTTP to the OTel Collector (`--otel-endpoint`, port `4318`).
2. The collector converts them and exposes a Prometheus scrape endpoint on port `8889`.
3. Your existing Prometheus scrapes `:8889` via a new `fibre` job and optionally evaluates `fibre_alerts.yml`.
4. Your existing Grafana renders the dashboard from Prometheus; alerts route through your existing Alertmanager.

```
┌──────────────┐                    ┌──────────────────┐                  ┌────────────────┐
│ fibre binary │   OTLP/HTTP push   │  OTel Collector  │   scrape :8889   │   Prometheus   │
│ fibre start  │ ─────────────────▶ │  (otelcol.yaml)  │ ◀─────────────── │   (existing)   │
│ --otel-      │  :4318 /v1/metrics │  in:  :4318      │   new job        │                │
│  endpoint    │                    │  out: :8889      │   "fibre"        │                │
└──────────────┘                    └──────────────────┘                  └───────┬────────┘
                                                                                  │
                                                            query                 │ rules: fibre_alerts.yml
                                                              ┌───────────────────┼──────────┐
                                                              ▼                   ▼          │
                                                      ┌──────────────┐   ┌───────────────┐   │
                                                      │   Grafana    │   │ Alertmanager  │ ◀─┘
                                                      │  dashboard   │   │  (existing)   │
                                                      │  (existing)  │   │  (optional)   │
                                                      └──────────────┘   └───────────────┘
```

## Prerequisites

- A running fibre server built from celestia-app **v10+** (metric surface verified against v10.4.0; other releases may expose a subset).
- **Docker** with the compose plugin on the host that will run the collector (can be the validator host itself).
- Existing **Prometheus** and **Grafana**, with permission to edit `prometheus.yml` and add dashboards.
- Network reachability: fibre host → collector `:4318`, Prometheus → collector `:8889`.

## Step 1 — Run the collector

Run from the directory that contains `monitoring/`.

Plain `docker run`:

```bash
docker run -d --name fibre-otelcol --restart unless-stopped \
  -p 4318:4318 \
  -p 8889:8889 \
  -v "$PWD/monitoring/otelcol/otelcol.yaml:/etc/otelcol-contrib/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:latest
```

Or paste this service into your own `docker-compose.yml`:

```yaml
services:
  fibre-otelcol:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: fibre-otelcol
    restart: unless-stopped
    volumes:
      - ./monitoring/otelcol/otelcol.yaml:/etc/otelcol-contrib/config.yaml:ro
    ports:
      - "4318:4318"   # OTLP/HTTP in  — the fibre binary pushes here
      - "8889:8889"   # Prometheus out — Prometheus scrapes here
```

```bash
docker compose up -d fibre-otelcol
```

Notes:

- Pin a released collector tag instead of `latest` for production.
- If the fibre binary and Prometheus both reach the collector via localhost, bind to loopback only: `"127.0.0.1:4318:4318"` and `"127.0.0.1:8889:8889"`.
- The OTLP receiver has no authentication. Do not expose `:4318` on a public interface without firewall rules — anyone who can reach it can write metrics into your pipeline.

## Step 2 — Point the fibre binary at the collector

```bash
fibre start --otel-endpoint http://<collector-host>:4318
```

or via environment variable:

```bash
export FIBRE_OTEL_ENDPOINT=http://<collector-host>:4318
fibre start
```

systemd unit:

```ini
# sudo systemctl edit fibre   (or your unit name)
[Service]
Environment="FIBRE_OTEL_ENDPOINT=http://127.0.0.1:4318"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart fibre
```

Gotchas:

- **Pass the base URL only.** The binary appends `/v1/metrics` (and `/v1/traces`) itself. `http://host:4318/v1/metrics` would push to `/v1/metrics/v1/metrics` and get 404s.
- Same host: use `http://127.0.0.1:4318`. Nothing on `:4318` needs to be reachable from the internet — firewall it to the fibre host.
- The setting takes effect at process start; restart the fibre server after changing it.
- The binary also pushes traces to `/v1/traces`. The bundled collector config deliberately drains them via a `nop` exporter, so pushes never error and traces are simply not stored — no action needed (a commented example shows how to wire Tempo instead if you ever want them).
- With OTLP enabled, the binary additionally emits `go.*` runtime series. Expected and harmless — the dashboard keys on `fibre_server_*`.

## Step 3 — Scrape the collector from Prometheus

### Generic Prometheus

Add to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "fibre"
    scrape_interval: 15s
    static_configs:
      - targets: ["<collector-host>:8889"]   # e.g. "127.0.0.1:8889" if Prometheus runs on the collector host
```

Any unique job name works; the examples here use `fibre`. Apply with `curl -X POST http://<prometheus>:9090/-/reload` (if `--web.enable-lifecycle` is on) or restart Prometheus.

### nodejumper monitoring-tool — exact flow

For the [nodejumper monitoring-tool](https://github.com/nodejumper-org/monitoring-tool) stack (assumed cloned at `~/monitoring-tool`). That stack keeps its Prometheus config in `prometheus/prometheus.yml` (created from `prometheus.yml.example`), auto-loads any dashboard JSON dropped into `grafana/provisioning/dashboards/` (via `grafana/provisioning/dashboards/config.yml`, 30 s scan interval), and keeps alert rule files in `prometheus/*.yml` alongside `cosmos_alerts.yml` / `server_alerts.yml`, with notifications routed by `alertmanager/config.yml`.

1. **Copy the files in:**

   ```bash
   cd ~/monitoring-tool
   BUNDLE=/path/to/this/repo/monitoring

   mkdir -p otelcol
   cp "$BUNDLE/otelcol/otelcol.yaml"          otelcol/otelcol.yaml
   cp "$BUNDLE/prometheus/fibre_alerts.yml"   prometheus/fibre_alerts.yml
   cp "$BUNDLE/dashboards/fibre-operator.json" grafana/provisioning/dashboards/fibre-operator.json
   ```

2. **Edit `docker-compose.yml`** — add the collector service (anywhere under `services:`):

   ```yaml
     fibre-otelcol:
       container_name: fibre_otelcol
       image: otel/opentelemetry-collector-contrib:latest
       restart: unless-stopped
       volumes:
         - ./otelcol/otelcol.yaml:/etc/otelcol-contrib/config.yaml:ro
       ports:
         - "4318:4318"        # fibre binary pushes OTLP here
       networks:
         - monitoring
   ```

   and add one line to the existing `prometheus:` service's `volumes:`:

   ```yaml
         - ./prometheus/fibre_alerts.yml:/etc/prometheus/fibre_alerts.yml
   ```

   The stack mounts each rule file individually — without this mount the file does not exist inside the Prometheus container. Prometheus reaches the collector over the internal `monitoring` network by service name, so `:8889` does not need to be published.

3. **Edit `prometheus/prometheus.yml`** — register the rule file and add the scrape job:

   ```yaml
   rule_files:
     - server_alerts.yml
     - cosmos_alerts.yml
     - fibre_alerts.yml          # add this line

   scrape_configs:
     # ... existing jobs ...
     - job_name: "fibre"         # add this block
       static_configs:
       - targets: ["fibre-otelcol:8889"]
   ```

   > **Note:** do not set a static `instance:` label here — one collector can front several fibre servers. The dashboard's `instance` dropdown lists your fibre servers by hostname automatically (the collector converts the binary's `service.instance.id` resource attribute into a `service_instance_id` label on every series).

4. **Apply:**

   ```bash
   sudo docker compose up -d                            # creates fibre-otelcol, recreates prometheus (new mount)
   sudo docker compose restart prometheus grafana
   ```

   (Prometheus also reloads on `curl -X POST localhost:9090/-/reload` — the stack enables `--web.enable-lifecycle`.)

5. **Point the fibre binary** (Step 2): `http://127.0.0.1:4318` if fibre runs on this host; `http://<this-host-ip>:4318` if it runs elsewhere — firewall `:4318` to that host.

The dashboard appears in Grafana within ~30 s of the file drop; alerts route to the Telegram receiver in `alertmanager/config.yml` (set `chat_id` / `bot_token` there if you haven't already).

## Step 4 — Add the dashboard

**Option A — UI import (any Grafana):**

1. Dashboards → New → Import → upload `monitoring/dashboards/fibre-operator.json`.
2. When prompted for `DS_PROMETHEUS`, pick your Prometheus datasource → Import.
3. At the top of the dashboard, set the `instance` dropdown to your fibre server.

**Option B — provisioning drop-in:**

Copy the JSON into a directory watched by a Grafana file provider (in the nodejumper stack: `grafana/provisioning/dashboards/` — covered above). The dashboard's datasource variable falls back to your default Prometheus datasource; if panels bind to the wrong one, pick the correct datasource from the dashboard's datasource dropdown.

## Step 5 (optional) — Enable the alerts

The dashboard works without alerts. To enable `monitoring/prometheus/fibre_alerts.yml`:

1. Copy the file where your Prometheus reads rule files (nodejumper flow already did this).
2. Reference it from `prometheus.yml`:

   ```yaml
   rule_files:
     # ... existing rule files ...
     - fibre_alerts.yml    # path relative to prometheus.yml, or absolute
   ```

3. Validate and reload:

   ```bash
   promtool check rules fibre_alerts.yml
   # inside the nodejumper container instead:
   docker exec prometheus promtool check rules /etc/prometheus/fibre_alerts.yml

   curl -X POST http://<prometheus>:9090/-/reload   # or restart
   ```

**Notifications** go through your existing Alertmanager. In the nodejumper stack all alerts land in the single `telegram` receiver — fill `chat_id` / `bot_token` in `alertmanager/config.yml` and `sudo docker compose restart alertmanager`.

**Tuning:** thresholds and `for:` windows are plain YAML in `fibre_alerts.yml` — edit, re-validate with `promtool`, reload. **Disable everything:** remove the `rule_files` entry and reload. **Disable one alert:** delete or comment out that rule block. **Temporary silence:** use your Alertmanager (UI or `amtool`).

## Verify it works

- [ ] **Collector healthy:** `docker logs fibre-otelcol --tail 50` shows a clean startup, no config errors.
- [ ] **Metrics exposed:** `curl -s http://<collector-host>:8889/metrics | grep '^fibre_server'` returns series (allow ~1 minute after the fibre restart). `fibre_server_upload_shard_occupancy_bytes` / `fibre_server_upload_shard_budget_bytes` are always present while the server runs.
- [ ] **Prometheus target UP:** `http://<prometheus>:9090/targets` → job `fibre` = UP.
- [ ] **Series in Prometheus:** query `fibre_server_upload_shard_occupancy_bytes` — instant result. After the first upload, `fibre_server_upload_shard_duration_seconds_count` appears.
- [ ] **Dashboard:** panels populate; the `instance` dropdown lists your server.
- [ ] **Alerts loaded** (if enabled): Prometheus → Status → Rules / Alerts lists the fibre rules.
- [ ] **End-to-end alert test** (if enabled): stop the fibre server (`sudo systemctl stop fibre`). After the rule's `for:` window, `FibreServerInstanceStoppedReporting` (critical) goes FIRING in Prometheus → Alerts and lands in your receiver; with uploads still flowing elsewhere, a stale `fibre_server_upload_shard_last_success_timestamp_seconds` also trips `FibreUploadSuccessStale` (warning). Start the server again and watch them resolve.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `curl :8889/metrics` shows no `fibre_server_*` series | fibre isn't pushing — endpoint not set or unreachable | Confirm the flag/env on the *running* process (`systemctl cat fibre`); check fibre logs for OTLP export errors; test reachability from the fibre host: `nc -zv <collector-host> 4318`; check `docker logs fibre-otelcol` |
| fibre log shows push failures with **404** | endpoint URL includes a path | The binary appends `/v1/metrics` itself — set the endpoint to the base URL only (`http://host:4318`). If a reverse proxy fronts the collector, it must forward `/v1/metrics` (and `/v1/traces`) |
| Prometheus target `fibre` is DOWN | Prometheus can't reach `:8889` | Port not published or wrong host. Inside docker networks target the **service name** (`fibre-otelcol:8889`) — `localhost` inside the Prometheus container is the container itself |
| Duplicate/merged series, `job` label confusing | job name `fibre` already used by another scrape config | Job names land in the `job` label on every series — keep them unique; rename one job |
| Dashboard panels show "No data" | wrong datasource or empty `instance` filter | Re-check the `DS_PROMETHEUS` pick at import / the dashboard datasource dropdown; confirm series exist in Prometheus first (checklist above) |
| `last_success_timestamp` / upload-latency panels empty | no successful upload yet | Expected — that gauge is not emitted before the first successful upload, and histogram series appear only after the first observation |
| Collector container crash-loops | invalid `otelcol.yaml` | `docker logs fibre-otelcol` for the parse error; re-copy the file; validate: `docker run --rm -v "$PWD/monitoring/otelcol/otelcol.yaml:/etc/otelcol-contrib/config.yaml:ro" otel/opentelemetry-collector-contrib validate` |
| Alerts fire but nothing notifies | Alertmanager routing | nodejumper: `chat_id` / `bot_token` must be set in `alertmanager/config.yml`; check Alertmanager container logs |

## Metric naming in Prometheus

The collector converts OTLP names to Prometheus conventions: dots become underscores and units become suffixes. Examples (celestia-app v10.4.0 surface):

| OTLP instrument | Prometheus series |
|---|---|
| `fibre.server.upload_shard.duration` (histogram, s) | `fibre_server_upload_shard_duration_seconds_{bucket,sum,count}` |
| `fibre.server.upload_shard.bytes` (counter, By) | `fibre_server_upload_shard_bytes_total` |
| `fibre.server.upload_shard.in_flight` (up/down) | `fibre_server_upload_shard_in_flight` |
| `fibre.server.upload_shard.occupancy_bytes` / `.budget_bytes` (gauges) | `fibre_server_upload_shard_{occupancy,budget}_bytes` |
| `fibre.server.upload_shard.rejected` (counter, `reason`) | `fibre_server_upload_shard_rejected_total` |
| `fibre.server.sign.duration` (histogram, s) | `fibre_server_sign_duration_seconds_{bucket,sum,count}` |
