# Feature Specification: Operator-Side Telemetry MVP for Celestia Fibre Servers

**Feature Branch**: `001-operator-telemetry-mvp`

**Created**: 2026-10-21

**Status**: Draft

**Input**: User description: "Operator-side telemetry MVP for Celestia Fibre servers: hosted + self-hostable ingestion of fibre.server.* OTLP metrics with per-tenant dashboards, opinionated alert pack, and anonymized fleet benchmarks"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Operator connects a fibre server and sees telemetry flowing (Priority: P1)

A Celestia validator operator creates an account, registers their fibre server (identified by its validator identity), receives a per-tenant ingest token, and adds a single startup flag plus the token to their fibre server configuration. Within minutes, the service confirms that metrics are arriving, that the server's software version was recognized, and that its metric stream is understood. No software installation on the operator's side is required beyond the flag change and a restart they fully control.

**Why this priority**: Nothing in the product delivers value until an operator's metrics are flowing and correctly attributed. This journey is deliberately zero-install to minimize friction during the current validator-registration window; it must work before any dashboard or alert matters.

**Independent Test**: Can be fully tested by creating an account, pointing a running fibre server at the service with the issued token, and confirming (a) data is accepted, (b) it is attributed to the right tenant and server, and (c) a "connected / receiving" status with server-version recognition is shown. Delivers standalone value: verified telemetry flow with zero inbound access to the operator's infrastructure.

**Acceptance Scenarios**:

1. **Given** a registered tenant with a valid ingest token, **When** the operator starts their fibre server pointing at the service with that token, **Then** the service accepts the pushed metrics, attributes them to that tenant and server, and shows the server as connected within one minute of first data arrival.
2. **Given** a tenant, **When** they request a new ingest token, **Then** a token is issued once for copying, and the tenant can later rotate or revoke it without involving support.
3. **Given** a server pushing with a revoked or unknown token, **When** data arrives, **Then** the service rejects it and the tenant sees no data from that source.
4. **Given** an infra provider tenant operating several validators, **When** they connect multiple fibre servers with the same tenant token, **Then** each server appears as a distinct monitored entity with its own identity and status.
5. **Given** a connected server, **When** the operator opens the connection status view, **Then** they see the server's reported software version and whether that version's metric contract is fully supported, partially supported, or unknown.

---

### User Story 2 - Operator monitors their server's health on per-tenant dashboards (Priority: P1)

An operator opens their private dashboards and sees the health of their own fibre server(s) at a glance: upload and download activity and latency, rejected uploads with reasons, storage occupancy against their stake-derived budget, prune-loop behavior, payment-promise signing latency, and connection pressure. All data shown is scoped strictly to their own tenant; they can select time ranges across retained history.

**Why this priority**: Dashboards are the day-to-day surface that turns raw telemetry into situational awareness — the operator sees budget exhaustion, prune lag, or signing-latency drift building up *before* external observers flag them publicly. Joint MVP core together with Story 1.

**Independent Test**: Can be fully tested by ingesting a known metric stream from a test server and verifying each dashboard panel renders the expected values for the correct tenant and time range, and that no other tenant's data is ever visible. Delivers standalone value: complete self-serve visibility into one's own fibre server.

**Acceptance Scenarios**:

1. **Given** a connected server emitting metrics, **When** the operator opens the at-a-glance dashboard, **Then** current upload/download activity, occupancy versus budget, signing latency, and connection state for their own server(s) are visible and update as new data arrives.
2. **Given** historical data within the retention window, **When** the operator selects a past time range, **Then** dashboards render that range (raw resolution inside the raw-retention window, downsampled resolution beyond it) without the operator needing to know which storage tier served it.
3. **Given** two tenants with connected servers, **When** tenant A views any dashboard, **Then** no metric, server name, or aggregate belonging to tenant B is ever displayed.
4. **Given** an operator using only direct push (no agent), **When** they open the host-and-disk dashboard, **Then** it clearly indicates that this view requires the optional agent path rather than showing empty or misleading panels.

---

### User Story 3 - Operator receives actionable alerts on their own channels (Priority: P2)

An operator enables the opinionated alert pack and routes notifications to the channels they actually watch (Telegram, Discord, a generic webhook, or PagerDuty). When their server starts rejecting uploads for budget exhaustion, when prune lag builds, when signing latency breaches its SLO, when the server silently stops accepting uploads, or when connections saturate, they receive a notification explaining what fired and why — and a resolution notice when it clears. They can disable individual rules or adjust thresholds to fit their setup.

**Why this priority**: The alert pack encodes the operational knowledge that makes this product more than graphs — it is the differentiating value. It depends on ingestion (Story 1) but not on dashboards (Story 2): a notification-only operator already gets the core protection.

**Independent Test**: Can be fully tested by replaying metric streams that deliberately breach each rule's condition and verifying that the right notification reaches a test channel, that resolution notices follow, and that disabled rules stay silent. Delivers standalone value: an operator is woken up for the failures that matter, with zero dashboard usage.

**Acceptance Scenarios**:

1. **Given** a connected server whose occupancy crosses 80% of its storage budget, **When** the alert pack is enabled with a Telegram route configured, **Then** the operator receives a budget-utilization alert naming the server and the current utilization, and a resolution notice after it drops back below threshold.
2. **Given** a rule the operator considers noise, **When** they disable that rule or raise its threshold, **Then** it no longer notifies them while the rest of the pack keeps working.
3. **Given** a configured notification channel that is failing, **When** an alert fires, **Then** the delivery failure is visible to the operator in the service so silent alerting gaps do not go unnoticed.
4. **Given** an alert that fired, **When** the operator reviews their alert history, **Then** they see when it fired, when it resolved, and the values that triggered it.
5. **Given** a previously-connected server that stops sending any data, **When** the silence persists beyond the expected interval, **Then** the operator is notified that telemetry itself has stopped (distinct from any single metric crossing a threshold).

---

### User Story 4 - Operator upgrades their fibre server without telemetry gaps (Priority: P2)

The fibre server's metric names and labels change between software releases. An operator upgrades their server binary to a new release; the service recognizes the new version's metric contract and keeps dashboards and alerts working unchanged — no re-onboarding, no config edits, no silent gaps. If the service does not yet know the new version, the operator is explicitly told so (with their data still safely ingested for recognized series), instead of panels quietly going empty.

**Why this priority**: The metric surface is explicitly unstable across releases; version-drift handling is a core product feature, not an afterthought. Without it, every upstream upgrade silently breaks the very trust the product is built on.

**Independent Test**: Can be fully tested by replaying golden metric fixtures from two different server releases against the same tenant and verifying that canonical dashboards and alerts produce identical semantics for both, plus replaying a fixture from an unknown future version and verifying the operator-visible "unsupported version" state. Delivers standalone value: upgrade-safe monitoring.

**Acceptance Scenarios**:

1. **Given** a server reporting a fully supported version, **When** the operator upgrades to another supported version, **Then** the same dashboards and alert rules continue to work on the canonical series with no operator action and no gap in interpretation.
2. **Given** a server reporting a version the service has never seen, **When** its data arrives, **Then** recognized series are still ingested and shown, the server is flagged as running an unsupported version, and the operator is notified of the drift.
3. **Given** a known version whose release renamed or added metrics relative to its predecessor, **When** data arrives under both old and new names (e.g., during a rolling upgrade across an infra provider's fleet), **Then** both map to the same canonical series and are presented as one continuous history where semantically identical.
4. **Given** the operator's server version drifts from the version the network majority runs, **When** the version-drift alert rule is enabled, **Then** the operator is notified they are behind (or ahead of) the fleet majority.

---

### User Story 5 - Operator self-hosts the entire stack (Priority: P3)

A sovereignty-focused operator (or an infra provider with a strict no-external-telemetry policy) deploys the complete service in their own infrastructure using the published packaged deployment and documentation, gaining the same dashboards and alert pack for their own servers without any data leaving their environment. Optionally, they may explicitly opt in to contribute anonymized aggregates back to the hosted fleet benchmark; the default is off.

**Why this priority**: Self-hosting is a non-negotiable trust rule and an adoption unlock for the most security-conscious validators, but the hosted path serves the majority first; self-host parity can trail the hosted MVP slightly without blocking core value.

**Independent Test**: Can be fully tested by deploying the packaged stack in a clean environment, connecting a test server, and verifying dashboards and alerts work end-to-end with zero outbound dependency on the hosted service. Delivers standalone value: full product under the operator's exclusive control.

**Acceptance Scenarios**:

1. **Given** the published self-host package and documentation, **When** an operator with standard container infrastructure follows the installation guide, **Then** they have a working deployment able to ingest from their servers, render dashboards, and deliver alerts, without any account on the hosted service.
2. **Given** a self-host installation with default settings, **When** it runs, **Then** no metric data or aggregates leave the operator's environment.
3. **Given** a self-host operator who explicitly opts in to fleet contribution, **When** the opt-in is enabled, **Then** only anonymized aggregates (never raw series) are published, and the operator can see exactly what is shared.
4. **Given** a self-host installation, **When** the operator inspects its documentation, **Then** they find how to apply the same data-retention policy (raw 30 days, downsampled 13 months) and how to upgrade when new server-version contracts are released.

---

### User Story 6 - Operator compares their server against anonymized fleet benchmarks (Priority: P3)

An operator opens the fleet benchmark view to answer "am I normal?": how do my upload latencies, rejection rates, occupancy growth, and signing latencies compare against anonymized percentiles computed across all participating tenants. No benchmark is ever shown when the contributing cohort is smaller than five tenants, and no individual tenant is ever identifiable from an aggregate.

**Why this priority**: Benchmarks are the compounding network-effect feature, but they only become meaningful once enough tenants participate; they depend on Stories 1–2 data and on trust rules being demonstrably enforced. Valuable, not launch-blocking.

**Independent Test**: Can be fully tested by seeding multiple tenants' data, running the aggregation, and verifying percentiles render, that cohorts below five tenants are suppressed entirely, and that no query can attribute an aggregate contribution to a specific tenant. Delivers standalone value: context no single operator can obtain alone.

**Acceptance Scenarios**:

1. **Given** five or more tenants contributing data, **When** the nightly aggregation runs, **Then** anonymized percentile benchmarks for the key health metrics are published and visible to tenants.
2. **Given** fewer than five contributing tenants for a metric cohort, **When** benchmarks are requested, **Then** that cohort's benchmark is suppressed with an explanation rather than a number.
3. **Given** a tenant viewing a benchmark, **When** they compare against their own server, **Then** their own values are shown alongside the anonymized fleet percentile bands without exposing any other tenant's identity or individual values.
4. **Given** a self-host tenant that has not opted in, **When** aggregates are computed, **Then** their data contributes nothing to any published benchmark.

---

### Edge Cases

- **Invalid, expired, or revoked token**: push attempts are rejected; repeated failures must not affect other tenants' ingestion, and the tenant can see recent rejected-push attempts to debug misconfiguration.
- **Server identity collisions**: two fibre servers reporting the same instance identity (e.g., a cloned config) must be surfaced to the tenant as a conflict to resolve, not silently merged into one series.
- **Clock skew at the source**: data points with timestamps significantly in the future or far past are handled predictably (accepted within a documented tolerance, flagged otherwise) so one misconfigured clock cannot corrupt a tenant's timeline.
- **Duplicate delivery**: retried or duplicated pushes from the same server are deduplicated so counters and budgets are not double-counted.
- **Metric flood / cardinality explosion**: a misbehaving or compromised source sending abnormal volumes or label cardinality is rate-limited or quarantined per tenant, protecting other tenants and flagging the cause to the affected tenant.
- **Unknown server version**: ingestion of recognized series continues; unknown series are recorded and flagged, never crash or block the pipeline, and the operator sees a clear "unsupported version" state.
- **Cohort below anonymity threshold**: any fleet aggregate computed over fewer than five distinct tenants is suppressed in every view and export, without leaking that the cohort exists at a specific small size.
- **Silent server**: a previously-reporting server that stops pushing triggers a telemetry-silence notification; dashboards mark the data as stale rather than showing the last value as if current.
- **Alert channel misconfiguration**: undeliverable notifications (bad webhook, revoked bot token) are surfaced as a channel health problem to the operator.
- **Retention boundary**: when raw data ages past 30 days, queries transparently fall back to downsampled resolution for the older range; no error, no fabricated resolution.
- **Non-metric payloads**: anything other than metrics (traces, logs, blob payloads, key material) received on the ingest endpoint is discarded and never stored or displayed.
- **Tenant offboarding**: on account deletion, all tenant data — raw series, downsampled series, alert history, tokens — is purged within a documented window, and the tenant's past contributions to already-published anonymized aggregates are not reversible (documented plainly on the data-handling page).

## Requirements *(mandatory)*

### Functional Requirements

**Ingestion & tenancy**

- **FR-001**: The service MUST accept metrics pushed by the fibre server binary over its native push protocol (OTLP/HTTP — the only export mechanism the binary offers), authenticated by a per-tenant ingest token.
- **FR-002**: Tenants MUST be able to issue, rotate, and revoke ingest tokens self-service; revocation MUST take effect on subsequent push attempts.
- **FR-003**: Every accepted data point MUST be attributed to exactly one tenant and one monitored fibre server, using the server's self-reported identity (service version, instance identity) and the tenant implied by the token.
- **FR-004**: The service MUST reject pushes with missing, unknown, or revoked tokens, and MUST make recent rejection events visible to the tenant for debugging.
- **FR-005**: Ingestion MUST be metrics-only: the service MUST NOT accept, store, or display blob payloads, key material, traces, or logs received on the ingest path; non-metric payloads are discarded.
- **FR-006**: A single tenant MUST be able to monitor multiple fibre servers (infra-provider posture), each appearing as a distinct entity with its own status, dashboards, and alert scope.
- **FR-007**: All data access MUST be scoped to the requesting tenant; no tenant may read, infer, or enumerate another tenant's servers, series, alert history, or configuration.

**Versioned metric adaptation**

- **FR-008**: The service MUST maintain a canonical internal metric model for fibre-server health and MUST map each supported server release's metric names, types, and labels onto that model, so dashboards and alerts are written once against canonical series.
- **FR-009**: Version support MUST cover at minimum the server release deployed on mainnet at spec time (celestia-app v10.4.0 metric contract) and MUST be extensible per release without operator action.
- **FR-010**: The service MUST detect the server's software version from its self-reported attributes and MUST classify it as fully supported, partially supported, or unknown — surfacing that state prominently to the operator.
- **FR-011**: Unrecognized metric names or labels MUST NOT interrupt ingestion of recognized ones; unrecognized data MUST be recorded and flagged for contract-drift review.
- **FR-012**: When a new server release is onboarded, existing tenants MUST NOT need to change configuration, tokens, or dashboards for their monitoring to keep working.
- **FR-013**: The correctness of each supported version's mapping MUST be verifiable against a recorded golden fixture of that release's actual metric output (contract test per release).

**Dashboards**

- **FR-014**: Each tenant MUST have private dashboards covering, at minimum: (a) at-a-glance health of their own server(s); (b) uploads (activity, latency, rejected uploads by reason, duplicate-upload hits); (c) serving/downloads (activity, latency, bytes); (d) storage and budget (occupancy versus stake-derived budget, prune-loop activity and lag); (e) payment-promise signing latency; (f) connections (concurrency and saturation pressure); (g) host and disk — available only via the agent path and clearly marked as such otherwise.
- **FR-015**: Dashboards MUST render only the requesting tenant's data and MUST mark stale data (no recent points) visually distinct from live data.
- **FR-016**: Tenants MUST be able to select arbitrary time ranges across the full retention window, with raw resolution inside the raw-retention period and downsampled resolution beyond it, transparent to the user.

**Alerting**

- **FR-017**: The service MUST ship an opinionated alert pack, enabled per tenant, containing at minimum these rules: (1) storage budget utilization above 80%; (2) any upload rejected for budget exhaustion; (3) anomalous occupancy growth rate; (4) prune lag beyond expected deletion deadlines; (5) median payment-promise signing latency above 10 ms; (6) stale last-successful-upload timestamp during active windows; (7) free disk space below required budget headroom (agent path only); (8) connection saturation / download starvation risk; (9) duplicate-upload hit spike; (10) server version drift from the network majority; (11) telemetry silence from a previously-reporting server.
- **FR-018**: Tenants MUST be able to route alert notifications to at least Telegram, Discord, a generic webhook, and PagerDuty, with per-rule channel assignment.
- **FR-019**: Tenants MUST be able to enable or disable individual rules and to override rule thresholds; opinionated defaults MUST ship for every rule so the pack works with zero tuning.
- **FR-020**: Alerts MUST have a visible lifecycle: firing notification, repeat-notification throttling, resolution notification, and a queryable incident history including triggering values.
- **FR-021**: Notification delivery failures (invalid channel configuration, rejected deliveries) MUST be surfaced to the tenant as channel health status.
- **FR-022**: Rules depending on agent-path data MUST be clearly identified as unavailable until the agent path is active for that server.

**Agent path**

- **FR-023**: The service MUST publish a documented, ready-to-use agent configuration that an operator runs alongside their infrastructure to additionally collect host disk usage and consensus-node remote-signing latency, forwarding everything under the tenant's token.
- **FR-024**: The agent path MUST be optional: direct push alone MUST yield a fully functional product minus the explicitly agent-dependent views and rules.

**Self-host**

- **FR-025**: The complete stack MUST be deployable by an operator in their own environment via a published, documented package, providing dashboard and alert-pack functionality for their own servers with no dependency on the hosted service.
- **FR-026**: Self-host installations MUST NOT transmit any data externally by default; an explicit opt-in MAY enable publishing anonymized aggregates only (never raw series) to the hosted fleet benchmark, with the shared content inspectable by the operator.

**Fleet benchmarks**

- **FR-027**: The service MUST compute anonymized cross-tenant fleet aggregates (percentile bands for key health metrics) on a regular schedule, at least nightly.
- **FR-028**: No fleet aggregate computed over fewer than five distinct tenants MUST ever be published, displayed, or exported; suppressed cohorts MUST show an explanation instead of a value.
- **FR-029**: Tenants MUST be able to compare their own server's metrics against published fleet percentile bands without any other tenant's identity or individual values being exposed.

**Retention & data lifecycle**

- **FR-030**: Raw-resolution metric data MUST be retained for 30 days; downsampled data MUST be retained for 13 months; enforcement MUST be automatic for hosted and self-host deployments alike.
- **FR-031**: On tenant-initiated account deletion, all tenant data (raw and downsampled series, alert incidents, tokens, configuration) MUST be purged within a documented window not exceeding 30 days.

**Trust & transparency**

- **FR-032**: The service MUST publish a data-handling page documenting what is collected (metrics only), what is never collected (keys, blob payloads), retention, isolation, deletion, and fleet-aggregation anonymity rules.
- **FR-033**: The self-host option MUST remain available for every release of the service so that no operator is ever forced into hosted ingestion.

### Key Entities *(include if feature involves data)*

- **Tenant**: An operator organization using the service (a validator, an infra provider, or a self-host installation contributing opt-in aggregates). Owns tokens, servers, alert configuration, and notification channels. Relationships: has many ingest tokens, monitored servers, alert rules, and incidents.
- **Ingest Token**: A revocable credential authenticating metric pushes for one tenant. Key attributes: issued-at, last-used-at, revoked status. Never grants read access.
- **Monitored Fibre Server (Validator)**: A single fibre server instance reporting to a tenant. Key attributes: validator identity, self-reported software version, instance identity, support classification (supported / partial / unknown), connection status, last-seen time. Relationships: belongs to one tenant; source of many metric series; target of alert rules.
- **Metric Series**: A canonical health metric for one server over time (e.g., upload latency, occupancy vs. budget, signing latency), mapped from the version-specific names the server emits. Key attributes: canonical name, labels, resolution tier (raw vs. downsampled), retention class.
- **Alert Rule**: A named condition from the opinionated pack (or a tenant-tuned variant) evaluated against a server's series. Key attributes: condition, threshold(s), enabled state, severity, agent-path dependency, assigned notification channels.
- **Alert Incident**: One firing of a rule for one server. Key attributes: fired-at, resolved-at, triggering values, delivery records per channel, lifecycle state.
- **Fleet Aggregate**: An anonymized cross-tenant percentile band for one canonical metric, computed on a schedule. Key attributes: metric, percentile values, contributing-tenant count, computed-at. Publishable only when the contributing count is at least five.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can go from account creation to first visible metric from their own server in under 10 minutes, changing only one startup flag plus a token on their side; at least 90% of design-partner operators complete onboarding on the first attempt without assistance.
- **SC-002**: Dashboard freshness: at least 95% of data points emitted by a connected server are visible in that tenant's dashboards within 60 seconds of emission.
- **SC-003**: Alert latency: at least 95% of alert notifications are delivered to the configured channel within 5 minutes of the triggering condition being emitted by the server.
- **SC-004**: Hosted ingestion availability is at least 99.9% per calendar month, and no single tenant's misbehavior measurably degrades ingestion for other tenants.
- **SC-005**: Monitoring figures are faithful: 100% of canonical series for a supported server release are verifiable against that release's recorded golden fixture, and support for a new upstream server release is available within 7 days of its publication.
- **SC-006**: 100% of unknown-version and contract-drift events are surfaced to the affected operator within 24 hours of first occurrence; zero silent telemetry gaps after a server upgrade.
- **SC-007**: Anonymity: 100% of published fleet aggregates are computed over at least 5 distinct tenants; zero aggregates below that threshold are ever displayed or exported.
- **SC-008**: Isolation: zero cross-tenant data exposure in security review and penetration testing; 100% of read paths are tenant-scoped.
- **SC-009**: Retention compliance: 100% of raw data points older than 30 days are no longer queryable at raw resolution, and downsampled history remains queryable for at least 13 months.
- **SC-010**: Alert quality: during the design-partner phase (3–5 operators), at least 80% of delivered alert notifications are judged actionable by recipients, and 100% of resolved incidents produce a resolution notification.

## Assumptions

- **Users and environment**: Operators already run (or are about to run) a fibre server on a v10+ network and can modify its startup flags and restart it at will. The initial target cohort is the ~92 active mainnet validators, with mocha testnet and local devnets as development targets.
- **Account model**: A tenant corresponds to one operator organization; standard email-based account authentication is used for the hosted service (reasonable default; SSO is post-MVP). One tenant may monitor many servers.
- **Ingestion contract**: The fibre binary's native push protocol (OTLP/HTTP) is the only direct ingestion wire format, and the binary is push-only; the service never reaches into operator infrastructure. Trace payloads the binary may also emit are discarded at ingestion (metrics-only MVP).
- **Baseline version**: The known-good metric contract at spec time is celestia-app v10.4.0; metric names are treated as unstable across releases, and drift items known today (duplicate-upload hits, last-success timestamp, object-storage backend metrics) are handled by the versioned adaptation requirements rather than hard-coded assumptions.
- **Alert defaults**: Every pack rule ships with an opinionated default threshold derived from documented operational facts (e.g., 80% budget utilization, 10 ms median signing latency); tenants can tune or disable, but no tuning is required for value.
- **External context**: External-verdict data (public reachability probes, on-chain signature counts from third-party tools) is out of scope for MVP ingestion; chain-side data may later appear as annotations only.
- **Explicit MVP exclusions** (scope boundary): external shard probing (existing third-party tools cover it); on-chain indexing and publisher/escrow product views (a later sibling product); client-side (blob publisher) metrics; trace and log ingestion; mobile applications; SSO and enterprise role-based access control; automated remediation (the service detects and notifies — the operator acts).
- **Hosting posture**: The hosted service is the default consumption mode; a single deployment region is acceptable for MVP and will be stated on the data-handling page. Pricing/packaging is deliberately unspecified here.
- **Self-host parity**: Self-host deployments get the same dashboards, alert pack, and retention policy; hosted-only capabilities are limited to fleet-benchmark publication/consumption.
- **Notification channels**: Operators bring their own Telegram/Discord/PagerDuty accounts or webhook endpoints; the service is responsible only for delivery, delivery-status visibility, and safe credential handling.
- **Fleet benchmarks**: Benchmark views remain hidden until at least 5 tenants contribute; historical aggregates already published are not retracted when a tenant leaves (documented on the data-handling page).
