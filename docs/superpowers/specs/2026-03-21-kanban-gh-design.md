# kanban-gh Design Spec

**Date:** 2026-03-21
**Status:** Approved

---

## Overview

`kanban-gh` is a standalone set of Claude Code skills that replicates the kanban AI pipeline but uses **GitHub Projects v2 as the data store and UI** instead of SQLite and the Vite web board. No local server. No database files. GitHub IS the board.

The AI pipeline logic (Planner → Critic → Builder → Shield → Inspector → Ranger) is identical to the existing `kanban` skills. The only difference is the backend: all reads and writes go through the GitHub Projects GraphQL API and GitHub Issues via the `gh` CLI.

**Target repo:** https://github.com/evilsquid888/kanban-gh

---

## Skill Set

| Skill | Command | Purpose |
|-------|---------|---------|
| `kanban-gh-init` | `/kanban-gh-init [url]` | Connect to a GitHub Project, create pipeline fields, write local config |
| `kanban-gh` | `/kanban-gh <subcommand>` | Task CRUD — add, list, move, edit, remove, stats, context |
| `kanban-gh-run` | `/kanban-gh-run <ID\|name> [--auto]` | Full AI pipeline orchestration |
| `kanban-gh-refine` | `/kanban-gh-refine <ID\|name>` | Requirements refinement interview |
| `kanban-gh-explore` | `/kanban-gh-explore [topic]` | Codebase exploration, seeds project with phased tasks |

**Dependency:** `gh` CLI authenticated with `project` and `repo` scopes (`gh auth login`).

---

## Data Model

### GitHub Project Fields

`/kanban-gh-init` creates these fields on the target GitHub Project (skips any that already exist):

| Field | Type | Values |
|-------|------|--------|
| `Status` | Single select | Todo · Plan · Plan Review · Implement · Impl Review · Test · Done |
| `Priority` | Single select | low · medium · high |
| `Level` | Single select | L1 · L2 · L3 |
| `Tags` | Text | Comma-separated free text |

### Issue Structure

Each task is a real GitHub **Issue** (not a Draft) in the configured repo, added as a project item:

- **Title** — task title
- **Body** — requirements/description (updated by `/kanban-gh-refine` and agents)
- **Comments** — all agent outputs, one comment per agent run, with signature header:

```
> **Planner** `opus` · 2026-03-21T10:00:00Z
```

Draft issues are not used — real issues are required so comments work.

### Local Config

`/kanban-gh-init` writes `.claude/kanban-gh.json` in the project root:

```json
{
  "project": "2",
  "owner": "evilsquid888",
  "ownerType": "user",
  "repo": "evilsquid888/my-project"
}
```

### No Web Board

There is no `start.sh`, no local server, and no Vite app. The GitHub Projects UI at `github.com/users/{owner}/projects/{number}` is the board.

### Images

Images are handled natively by GitHub — users drag-and-drop attachments in issue comments via the browser. Agent comments use descriptive text only. Images already in the repo can be embedded via `https://raw.githubusercontent.com/...` URLs.

---

## `kanban-gh-init` Flow

1. Accept optional project URL or number as argument. If none provided, run `gh project list --owner @me` and prompt user to pick.
2. Fetch existing fields via GraphQL. Skip creating fields that already exist.
3. Create missing pipeline fields: `Status` (7 options), `Priority`, `Level`, `Tags`.
4. Ask which repo issues should be created in (default: current git remote).
5. Write `.claude/kanban-gh.json`.
6. Output confirmation with project URL.

---

## `/kanban-gh` CRUD Commands

| Command | Behavior |
|---------|----------|
| `list` | Fetch all project items via GraphQL, render markdown table (ID, Status, Priority, Title) |
| `add <title>` | Create GitHub Issue, add to project, set Status=Todo. Prompt for priority, level, description, tags. |
| `move <ID\|name> <status>` | Update Status field via GraphQL mutation. Enforces valid pipeline transitions. |
| `edit <ID\|name>` | Update issue body or field values interactively |
| `remove <ID\|name>` | Remove item from project; optionally close the issue |
| `stats` | Item counts per Status column |
| `context` | Pipeline state summary — what's in each column, what's active |

**ID resolution:** Commands accept either a GitHub issue number (`#12`) or a partial title string. If a title matches multiple items, list matches and prompt to pick.

---

## Pipeline — `/kanban-gh-run`

Identical pipeline levels and agent sequence as the existing kanban skills:

| Level | Path |
|-------|------|
| L1 Quick | Todo → Implement → Done |
| L2 Standard | Todo → Plan → Implement → Impl Review → Done |
| L3 Full | Todo → Plan → Plan Review → Implement → Impl Review → Test → Done |

**Per-step execution:**

1. **Read** — fetch item from GitHub Project (field values) and issue (body + all comments) via GraphQL
2. **Run agent** — same prompt templates as existing kanban (Planner, Critic, Builder, Shield, Inspector, Ranger)
3. **Write** — append signed agent comment to the issue via `gh issue comment`
4. **Advance** — update Status field via GraphQL mutation to next pipeline column
5. **Pause** (default) or continue (--auto) at review steps

**Done:** Final comment with summary. Status set to Done.

---

## Shared Internals

Three shared reference files consumed by all skills:

| File | Contents |
|------|----------|
| `shared/graphql.md` | Reusable GraphQL query/mutation templates for reading items, updating fields, listing projects |
| `shared/pipeline.md` | Pipeline levels, valid status transitions, agent prompt templates, scoring rubrics |
| `shared/schema.md` | Field names, status values, config file format |

---

## Repo Structure

```
kanban-gh/
├── README.md
├── kanban-gh/
│   └── SKILL.md
├── kanban-gh-init/
│   └── SKILL.md
├── kanban-gh-run/
│   └── SKILL.md
├── kanban-gh-refine/
│   └── SKILL.md
├── kanban-gh-explore/
│   └── SKILL.md
└── shared/
    ├── graphql.md
    ├── pipeline.md
    └── schema.md
```

---

## Installation

```bash
git clone https://github.com/evilsquid888/kanban-gh.git
cp -R kanban-gh/kanban-gh         ~/.claude/skills/
cp -R kanban-gh/kanban-gh-init    ~/.claude/skills/
cp -R kanban-gh/kanban-gh-run     ~/.claude/skills/
cp -R kanban-gh/kanban-gh-refine  ~/.claude/skills/
cp -R kanban-gh/kanban-gh-explore ~/.claude/skills/
```

No `pnpm install`. No server. No `start.sh`.

---

## Out of Scope

- Bidirectional sync with the SQLite-based kanban system
- A local web board
- Agent-side image uploads (GitHub API does not support this)
- Webhook/hook-based auto-sync (no hooks in this design)
