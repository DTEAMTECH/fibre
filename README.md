# Fibre

Open tooling for [Celestia Fibre](https://celestia.org/) — the validator-operated data-availability protocol introduced in the Celestia v10 upgrade. Built and published by the **DTEAMTECH** organization.

This repository is the project umbrella: product context, research notes, brand/design tokens, and the spec-driven development workflow. Individual components live in their own repos as they mature.

## Showcase

### fibrestats — operator-side telemetry for Fibre servers

**Status:** in development (spec phase) · **Repo:** developed privately, to be published separately

Every existing ecosystem tool measures Fibre validators *from the outside* (reachability probes, on-chain signature counts). fibrestats is the *internal* telemetry layer: it ingests the fibre server's own push-based OTLP metrics (`fibre.server.*` — upload/download latency, rejections, occupancy vs. stake-derived storage budget, prune lag, signing latency) and gives operators per-tenant dashboards, an opinionated alert pack, and anonymized fleet benchmarks — so they see budget exhaustion, SLO breaches, and disk pressure *before* external observers flag them publicly.

- **Users:** Celestia validators / node operators running fibre servers, and infra providers hosting them.
- **Ingestion shapes:** zero-install direct OTLP push · agent path (Grafana Alloy / OTel Collector bundle with node-exporter + KMS latency) · fully self-hostable stack.
- **Trust rules:** metrics-only ingestion (no keys, no blob payloads), per-tenant isolation, k-anonymity ≥ 5 on any fleet aggregate, self-host always available.
- **Stack:** Next.js + Tailwind + shadcn/ui web app; OTLP receiver + versioned metric adapter; VictoriaMetrics + Grafana.

Screenshots and a public link will land here once the MVP is live.

## Operator monitoring

Standalone monitoring bundle for Fibre server operators — a Grafana dashboard, optional Prometheus alert rules, an OTel Collector config, and a step-by-step guide for plugging it into an existing Prometheus/Grafana stack (worked example: the [nodejumper monitoring-tool](https://github.com/nodejumper-org/monitoring-tool)). Metrics only, no keys, self-hosted.

→ [`monitoring/README.md`](monitoring/README.md)

## How we build

This project follows [Spec Kit](https://github.com/github/spec-kit) spec-driven development: specs before code, carried through plan → tasks → implementation. The project constitution lives in [`.specify/memory/constitution.md`](.specify/memory/constitution.md).

Start with [`AGENTS.md`](AGENTS.md) — the full product & protocol context (input contract, chain reference, architecture, MVP scope, kickoff tasks). Research background is in [`docs/`](docs/), and the design tokens (Celestia palette/typography — no Celestia logos are used anywhere) in [`docs/brand.md`](docs/brand.md).

## License

TBD
