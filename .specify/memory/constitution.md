# fibrestats Constitution

fibrestats is a stats/monitoring web app for Celestia's Fibre data-availability protocol (CIP-51, shipped in the v10 network upgrade). This constitution governs every artifact produced in this repository, from idea assessment through implementation.

## Core Principles

### I. Spec-Driven Workflow (NON-NEGOTIABLE)

Spec before code, always. No source file, component, endpoint, or schema may be written without a specification that preceded it through the Spec Kit pipeline: `speckit.specify` → `speckit.clarify` (when ambiguity exists) → `speckit.plan` → `speckit.tasks` → `speckit.implement`. While the project is in idea determination, the discovery track (`speckit.assess.intake` → `research` → `define` → `shape` → `decide`) runs first; only a `go` verdict from `speckit.assess.decide` may hand off to `speckit.specify`. Code that arrives without a spec is rejected at review, regardless of quality.

### II. Celestia Brand Fidelity

The UI must be visually indistinguishable from an official Celestia property. Use only the official brand tokens from https://celestia.org/brand/; never invent off-brand colors, fonts, or styling.

- Anchors: Frost `#FDFCFF`, Void `#0E1014`. Brand carriers: Indigo `#5640D1` (primary), Amethyst `#7C68F2` (secondary).
- Cool Gray: 50 `#EAECEE`, 200 `#C4C8CE`, 400 `#808890`, 600 `#4A5058`, 800 `#242830`
- Blue: 50 `#EAEDF0`, 200 `#A9C2D4`, 400 `#5595C3`, 600 `#32688F`, 800 `#254459`
- Purple: 50 `#E3E1F2`, 200 `#C8C3E5`, 400 `#7C68F2`, 600 `#4331A5`, 800 `#231860`
- Beige: 50 `#EDECEB`, 200 `#C6BDB7`, 400 `#9A897E`, 600 `#6C5E55`, 800 `#473E38`
- Typography: Nuber Next (`nuberNext`) and Nuber Next Wide (`nuberNextWide`) are the brand fonts — commercial, never redistribute font files; load only via a properly licensed mechanism or substitute a declared fallback during development. Roboto Mono is the code/data font. Small labels follow the site's uppercase + wide-tracking convention.
- Celestia logos and marks (symbol, wordmark, any colorway) are NEVER used in our apps or materials. fibrestats is an independent DTEAMTECH product — palette and typography conventions only. Do not download, commit, or hotlink Celestia logo assets.

Any new color, font, or visual motif not derivable from these tokens is a constitution violation.

### III. shadcn/ui + Tailwind Exclusively

All UI is built exclusively with shadcn/ui components styled via Tailwind CSS utilities. No alternative component libraries (MUI, Chakra, Ant, Bootstrap, etc.), no hand-rolled CSS frameworks, no CSS-in-JS runtimes. Custom one-off elements are composed from shadcn/ui primitives and Tailwind classes; brand tokens (Principle II) are wired in as Tailwind theme values, not scattered hex literals.

### IV. TypeScript Strict Mode

All application code is TypeScript with `strict: true` (including `noUncheckedIndexedAccess` where the tsconfig supports it). No `any` escapes without an inline justification comment; no `@ts-ignore`/`@ts-nocheck` in committed code; shared types are the single source of truth across client and server boundaries.

### V. pnpm Package Management

pnpm is the only package manager. All dependency operations use pnpm (`pnpm add`, `pnpm install`, `pnpm dlx`); never npm, yarn, or bun. Lockfile is `pnpm-lock.yaml`; no other lockfile may be committed.

### VI. Discovery Artifacts & Research Boundaries

Idea-stage decisions live exclusively in `.specify/assessments/<slug>/` (intake, research, problem, concept, decision artifacts produced by the assess extension). Research inputs live in the sibling workspace `/Users/0x00d/Documents/kimi/Workspaces/celestia` — that workspace is **reference only, never modified** from this project. When research is needed, read from the sibling workspace and cite it; conclusions and decisions are recorded in `.specify/assessments/`, not in the research source. A snapshot summary for context lives in `docs/idea-context.md` and is superseded by assessment artifacts as they are produced.

## Technology Constraints

- Language: TypeScript (strict) for all app code; shell scripts only inside `.specify/scripts/`.
- UI: shadcn/ui + Tailwind CSS, themed exclusively with the Celestia tokens above.
- Package manager: pnpm only.
- Fonts: brand fonts are commercial — never commit font binaries; Roboto Mono via its open license for code/data.
- Data integrity: monitoring figures must be traceable to chain records or declared observations; chain-record vs. observation separation is a product-level requirement inherited from the research inputs.

## Development Workflow & Quality Gates

- Idea determination: assess pipeline under `.specify/assessments/`; a documented `kill` is a successful outcome.
- Feature work: full SDD pipeline per feature; specs and plans live under `.specify/` per Spec Kit conventions.
- Reviews verify constitution compliance: spec-first evidence, brand-token usage, shadcn/ui + Tailwind only, strict TypeScript, pnpm-only changes.
- Complexity must be justified against the simplest design that satisfies the spec (YAGNI).

## Governance

This constitution supersedes all other practices and preferences. Amendments require a documented rationale, an explicit version bump, and a migration note for any in-flight specs affected. All PRs/reviews must verify compliance with the principles above; violations block merge. Use `.specify/memory/constitution.md` (this file) as the runtime guidance source for all agents working in this repository.

**Version**: 1.0.0 | **Ratified**: 2026-10-20 | **Last Amended**: 2026-10-20
