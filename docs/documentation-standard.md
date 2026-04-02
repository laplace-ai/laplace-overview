# Documentation Standard

How documentation is organized, created, updated, and consumed across Laplace Log repos.

## Principles

Based on Anthropic's official Claude Code best practices, 04/2026 (1 and 2) and internal preferences (3 and 4):

1. **Invest in context.** CLAUDE.md and supporting docs are the project's persistent memory. Docs should be modular (one concern per file) and current (stale docs actively mislead the agent). The structure separates docs that load every session from docs that load on-demand, so the agent starts each session with the right baseline without overloading the context window.

2. **Invest in verification.** Tests and expected outputs let the agent check its own work. A spec backed by tests is a contract; a spec backed only by prose is a suggestion. Plans and specs should point to executable verification whenever possible.

3. **Docs are the primary interface.** We read and verify the system through its documentation, not the code directly. Docs must be complete, readable, and trustworthy for humans first — optimized for agents second.

4. **Enable async collaboration.** Knowledge trapped in one developer's conversation context is invisible to everyone else. Shared docs (reference, specs, guides, decisions) are the agreed-upon truth. Individual docs (plans, ideas) are a log of each developer's thinking — accessible to others for context without contaminating the shared baseline.

## Directory structure

```
docs/
├── reference/              # Imported in CLAUDE.md — always in context
│   ├── architecture.md
│   ├── conventions.md
│   └── design-system.md
├── specs/                  # On-demand — loaded when relevant to the task
│   ├── scenario-simulator.md
│   ├── margin-monitor.md
│   └── route-map.md
├── guides/                 # On-demand — how-to procedures
│   ├── local-setup.md
│   └── deploy.md
├── plans/                  # Temporal — active execution documents
│   ├── 2026-04-01_multi-tenant.md
│   └── 2026-04-15_agent-yaml.md
├── decisions/              # Temporal — ADRs, write-once
│   ├── 2026-03-20_polars-over-pandas.md
│   └── 2026-04-01_state-not-worldstate.md
└── ideas/                  # Temporal — exploration, freeform
    └── 2026-03-15_margin-monitor-feasibility.md
```

| Directory   | Context        | Lifecycle          | Prefix        | Purpose                              |
|-------------|----------------|--------------------|---------------|--------------------------------------|
| README.md   | always         | living             | —             | Repo entry point (GitHub)            |
| reference/  | always         | living             | —             | Repo-wide knowledge                  |
| specs/      | on-demand      | living             | —             | Subsystem descriptions               |
| guides/     | on-demand      | living             | —             | How-to procedures                    |
| plans/      | on-demand      | temporal           | `YYYY-MM-DD_` | Execution documents                  |
| decisions/  | on-demand      | temporal           | `YYYY-MM-DD_` | Why we chose X over Y                |
| ideas/      | on-demand      | temporal           | `YYYY-MM-DD_` | Exploration before commitment        |

## reference/ vs specs/

The distinction is **loading strategy**, not content type. Both are living docs that describe what the system is and how it must behave.

**reference/** — Repo-wide knowledge. **Imported in CLAUDE.md, always in context.**
Applies to the entire codebase regardless of which subsystem you're working on. Architecture, conventions, design system.

**specs/** — Subsystem-specific knowledge. **Loaded on-demand when pointed to.**
Applies to one part of the system and is only relevant when working on that part. Scenario simulator, margin monitor, auth system, chart system.

The test: "Does this apply to the whole repo or to a specific subsystem?" Whole repo → reference. Specific subsystem → specs.

## Document types

### `README.md` — Repo entry point

Lives at the repo root, outside `docs/`. Rendered by GitHub when opening the repo. **Imported in CLAUDE.md** — contains quick start commands, structure overview, and pointers to docs that the agent benefits from every session. Should include at minimum:

- **Quick start** — how to install and run locally
- **Documentation** — pointer to `docs/`
- **Structure** — high-level folder overview

### `reference/` — Always-loaded baseline

What the system is, right now, across the whole repo.

- **Updates:** in the same PR that changes the described behavior
- **Claude Code:** imported via `@docs/reference/<file>.md` in CLAUDE.md
- **Stale = dangerous:** if a reference doc contradicts the code, the code wins. Fix the doc immediately.

### `specs/` — On-demand system descriptions

What a subsystem or feature is, does, and must respect. Contracts, invariants, inputs/outputs, edge cases, scope boundaries.

- **Updates:** when requirements change or after a plan that affects the spec is completed
- **Claude Code:** loaded on-demand (`@docs/specs/<file>.md` when relevant)
- **Naming:** `<subsystem-or-feature>.md` (no date prefix — living documents)

When possible, point to executable verification: test files, CI checks, or commands that confirm the spec holds.

### `guides/` — How-to procedures

Step-by-step instructions for humans and agents. Local setup, deploy, run tests, debug common issues.

- **Updates:** when the procedure changes
- **Claude Code:** loaded on-demand
- **Naming:** `<verb-or-topic>.md`

### `plans/` — Execution documents

Temporal docs that describe HOW and WHEN to implement a change. A plan references which specs it affects and is tied to a specific execution window. Plans are separate from specs because one plan can affect multiple specs.

- **Updates:** during execution if the approach changes; mark as `done` when complete
- **Claude Code:** loaded on-demand, only during active execution
- **Naming:** `YYYY-MM-DD_<slug>.md`

Header template:

```markdown
# Plan: <title>
- **Date:** YYYY-MM-DD
- **Status:** draft | in-progress | done
- **Affects:** <spec filenames, comma-separated>

## Goal
<one paragraph>

## Approach
<numbered steps>

## Verification
<how to confirm it worked — tests, checks, expected outputs>
```

When a plan reaches `done`, update the affected specs and reference docs in the same PR.

### `decisions/` — Why we chose X over Y

Architecture Decision Records. Write-once, never updated (append a superseding decision instead).

- **Claude Code:** rarely — only when asked "why do we do X this way?"
- **Naming:** `YYYY-MM-DD_<slug>.md`

Header template:

```markdown
# Decision: <title>
- **Date:** YYYY-MM-DD
- **Status:** accepted | superseded by <link>

## Context
<what prompted this decision>

## Options considered
<brief list>

## Decision
<what we chose and why>
```

### `ideas/` — Exploration before commitment

Rough notes, feasibility analysis, research dumps. Low-ceremony, high-freedom. May graduate to a plan, be abandoned, or sit indefinitely.

- **Claude Code:** on-demand, when asked
- **Naming:** `YYYY-MM-DD_<slug>.md`

No required template.

## Rules for Claude Code

1. **Before creating a new doc**, check if an existing one covers the topic. Prefer updating over creating.
2. **Reference and specs must be readable by humans.** Write complete, clear docs — not shorthand optimized for token savings.
3. **When a plan reaches `done`**, update the affected specs and reference docs in the same PR.
4. **Use header metadata** (Status, Affects, Date) to understand relationships between docs.
5. **Specs describe the WHAT, plans describe the HOW.** Don't mix execution steps into specs or requirements into plans.
6. **Prefer concrete statements over narrative prose.** "API returns 404 when cargo not found" over "The API should handle the case where a cargo doesn't exist gracefully."
7. **Point to verification.** When a spec or plan can reference a test file or CI check, include the path or command.
