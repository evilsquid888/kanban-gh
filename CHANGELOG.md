# Changelog

## v1.0.0 — 2026-03-21

Initial release of kanban-gh — AI-powered kanban pipeline for Claude Code, backed by GitHub Projects v2.

### Skills

- **kanban-gh-init** — Connect to a GitHub Project, create pipeline fields (Status, Priority, Level, Tags), write local config
- **kanban-gh** — Task CRUD: `add`, `list`, `move`, `edit`, `remove`, `stats`, `context`
- **kanban-gh-run** — Full AI pipeline orchestration with 6 agents (Planner, Critic, Builder, Shield, Inspector, Ranger)
- **kanban-gh-refine** — Requirements refinement through structured user interview
- **kanban-gh-explore** — Codebase exploration with direction report and phased task seeding

### Shared Files

- **shared/schema.md** — Canonical field names, status values, agent nicknames, config format
- **shared/graphql.md** — 8 GraphQL query/mutation templates for GitHub Projects v2 API
- **shared/pipeline.md** — Pipeline levels (L1/L2/L3), valid transitions, 6 agent prompt templates, scoring rubrics

### Pipeline Features

- 7-column pipeline: Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
- 3 pipeline levels: L1 Quick, L2 Standard, L3 Full
- 7 AI agents with structured scoring rubrics (Critic: 3 dimensions, Inspector: 7 dimensions)
- Configurable retry limit (`--retries N`, default 2) — pauses for human review even in `--auto` mode
- Board loop mode (`--loop`) — processes entire backlog sequentially
- ID resolution by issue number (`#12`) or partial title match

### Architecture

- Pure Claude Code skills — no local server, no database, no web board
- All operations via `gh` CLI and GitHub Projects v2 GraphQL API
- Agent outputs stored as signed issue comments
- Status and metadata tracked via GitHub Project custom fields
- GitHub Projects UI is the board

### Bug Fixes (from testing)

- Fixed `createField` mutation: added required `color` and `description` fields on single-select options
- Fixed `updateProjectV2Field` mutation: takes `fieldId` only, not `projectId`
- Added `updateField` operation for modifying existing field options (handles default Status field on new GitHub Projects)
- Fixed init flow to update existing Status field options instead of erroring (new GitHub Projects always have a default Status with Todo/In Progress/Done)
