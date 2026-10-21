# fibrestats — Idea Context

*What the sibling research workspace says about Celestia Fibre and why a monitoring/stats app is being considered. This document summarizes; it does not define requirements.*

---

## 1. What Fibre is

Fibre is Celestia's new, parallel, validator-operated data-availability (DA) protocol, formalized as **CIP-51** and shipped via the **v10 network upgrade** (CIP-52). It is the flagship of Celestia's Vision 2.0 ("every market onchain") pivot — announced January 2026, claiming ~1 Tb/s sustained throughput in lab conditions vs. the classic DA path's ~12 MiB/s ceiling.

Mechanically:

- A blob publisher (celestia-node Light/Bridge, `fibre.Submit` / `fibre.Upload` RPC) funds an **escrow account** on-chain, signs a **PaymentPromise**, erasure-codes the blob into shards, and uploads shards to deterministically-assigned validators.
- Validators run an off-chain **fibre server** (gRPC `:7980`, TLS 1.3 identity endorsed by the consensus key), register their `host:port` in the on-chain **x/valaddr** provider registry, validate promises, store shards, and sign.
- Once ≥2/3 voting power signs, the publisher settles on-chain with **MsgPayForFibre**: escrow is charged (fee formula `650,000 + 45,000 × ⌈blob_size / 256 KiB⌉` utia → `fee_collector`) and a small "system blob" anchors the commitment.
- Readers reconstruct via `fibre.Download`. Failure path: unsettled promises can be timeout-settled by anyone (`MsgPaymentPromiseTimeout`) — escrow still charged, no data anchored.

Key protocol facts that shape monitoring: blob bytes never touch the chain (long-term history requires block/tx indexing, not queries); there is **no slashing** in CIP-51 — availability is quorum/incentive-based, so the protocol relies on **external observation** to detect non-serving validators; signing a promise proves nothing about storing or serving data.

Timeline so far: v10 activated on the **mocha-5 testnet on 2026-09-24** (height 1,082,619). One week later: 68/82 validators registered (93.5% of stake), 550 settled publications, ~28,692 TIA in escrow, but only ~6 unique payers. **Mainnet v10 is unscheduled** — everything built on testnet now is a rehearsal for the mainnet launch-day market.

## 2. Monitoring/stats gaps the research identified

Key findings from the exploration and competitive gap analysis:

- **The incumbent tools cover only the "observer" side.** Valopers (readable explorer-style dashboard: readiness card, settlement feed, endorsement rates, escrow aggregates), Tensile by Huginn (the evidentiary heavyweight — the only tool that actually downloads shards and verifies them against on-chain commitments, with signed daily exports and a formal methodology), and CelestiaResource (a registry listing added within days of activation) are all pull-only dashboards observing from outside. Registry views are already commoditized.
- **Push alerting is unclaimed (strongest gap).** No app notifies operators when their host becomes unreachable, endorsement rate collapses (live spread: 8%–99.6% across validators), TLS endorsement fails, params change, or they forget to register. Foundation Delegation Program criteria (run Fibre, upgrade within 24h) turn these into delegation risks operators would pay to be woken up for.
- **Operator-side telemetry is unclaimed.** The fibre server exports rich OTLP-push metrics (`fibre.server.*` — upload/download latency, rejections, occupancy vs. stake-derived budget, prune lag, signing latency) but has **no Prometheus scrape endpoint and no disk gauge**; metric names are explicitly unstable pre-mainnet, so any consumer needs a versioned adapter. Nothing helps a validator see itself.
- **Publisher/escrow tooling is unclaimed.** Only ~6–9 escrow accounts exist today, and no tool serves them: no balance forecasting, no visibility into celestia-node's auto-funding watermarks, no withdrawal countdowns, no per-payer fee analytics (the fee formula appears in no event — it must be recomputed).
- **Settlement-lifecycle tracking has a hole.** `fibre.Upload` broadcasts settlement in a background goroutine whose failure is only logged; nobody auto-submits timeout settlements (zero expired promises recorded likely means no timeout processor is running, not a perfect network). A public timeout keeper is both a service and a failure-statistics source.
- **The upgrade cockpit need is proven on testnet, sized for mainnet.** The 20-day mocha-5 rollout was tracked with improvised tooling; nobody offered a per-validator readiness checklist (binary version, gRPC enabled, :7980 reachable, `set-host` registered, first endorsement within 24h). Mainnet v10 has a hard-deadline "moment" when announced.
- **Long-term trend analytics are absent.** Everything shown is current-window; nobody draws capacity-utilization (2 TiB full-stake budget ≈ 146 MiB/s theoretical ingest), endorsement-concentration, fee-per-MiB, or payer-concentration trend lines — exactly the evidence the pending blob-economy governance debate will need.
- **Chain data must be indexed, not queried.** No on-chain queries exist for global escrow lists, per-height publications, or registry history; registry removals emit no events (poll-and-diff required). Celenium (the main explorer) has zero Fibre support as of the research date.
- **Trust standards are cheap and legible to the core team:** versioned methodology, per-figure block-height labelling, chain-record vs. observation separation, signed data exports (Tensile's vocabulary).
- **Risks flagged:** spec/code drift is real (docs lag shipped protos — verify against code at exact tags); shard-assignment constants are compiled in (version-pin verifiers); the 1 utia/gas pricing is a marked TODO; a read-path rate limiter is coming (probers must be polite); testnet demand is six payers — real users arrive only with mainnet v10.

**Positioning conclusion in the research:** don't build another observer dashboard (Tensile owns verification, Valopers owns explorer tables); be the **operational and economic layer** on top of chain records and existing evidence — alerting, operator telemetry, escrow/settlement watch, and the mainnet upgrade cockpit.

## 3. Research inputs

All sources live in the sibling workspace `/Users/0x00d/Documents/kimi/Workspaces/celestia` (read-only from this project):

| Source file | Absolute path | Description |
|---|---|---|
| `fibre-monitoring-exploration.md` | `/Users/0x00d/Documents/kimi/Workspaces/celestia/fibre-monitoring-exploration.md` | Code-level exploration (2026-09-21) of the x/fibre + x/valaddr on-chain API surface, node RPC, server metrics, and a monitorable-signal catalogue at celestia-app v10.2.0-mocha / celestia-node v0.34.2-mocha. |
| `fibre-apps-gap-analysis.md` | `/Users/0x00d/Documents/kimi/Workspaces/celestia/fibre-apps-gap-analysis.md` | Competitive analysis (2026-09-28) of the three live Fibre observability apps (Valopers, Tensile, CelestiaResource) with a capability matrix, six identified gaps, and recommended positioning/milestones. |
| `celestia-ecosystem-research-2026-09.md` | `/Users/0x00d/Documents/kimi/Workspaces/celestia/celestia-ecosystem-research-2026-09.md` | Broader ecosystem research (2026-09-16): Vision 2.0 strategy, upgrade history, tokenomics, adoption, developer pain points, and builder opportunity signals (Fibre tooling ranked highest). |
| `fibre_apps_charts.py` | `/Users/0x00d/Documents/kimi/Workspaces/celestia/fibre_apps_charts.py` | Matplotlib script generating the gap-analysis charts — a 15-capability × 3-app coverage matrix and a 20-day mocha-5 rollout timeline (outputs: `fibre_capability_matrix.png`, `fibre_rollout_timeline.png`). |

## 4. Status

Idea determination is **complete** (2026-10-21): the decided product is **fibrestats — operator-side telemetry for Celestia Fibre servers**, defined in the root `AGENTS.md` (product definition, input contract, architecture, MVP scope). This document remains as a historical snapshot (2026-10-20) of the research that fed the decision; the gap-analysis findings above are why the operator-telemetry positioning was chosen.
