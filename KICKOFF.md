# Kickoff prompt — paste this to the agent working in this workspace

```
Read AGENTS.md in this workspace root first — it is the full project context for
FibreStats (operator-side telemetry for Celestia Fibre servers). Also read
README.md: the Next.js + Spec Kit scaffold already exists, so do not
re-scaffold. Follow the Spec Kit workflow: produce the spec for the telemetry
MVP (AGENTS.md §5–§6), then the implementation plan — including your call on
the backend ingest language (Go service vs Node/TS, see §0) — and wait for my
approval before writing code. The dev-loop target is the local Fibre devnet
(§7 item 2) so fibre.server.* metrics flow without mainnet traffic. If anything
in AGENTS.md conflicts with the actual celestia-app code, trust the code and
flag the conflict to me.
```
