# kanban-gh Design Spec

**Date:** 2026-03-21
**Status:** Approved

---

## Overview

`kanban-gh` is a standalone set of Claude Code skills that replicates the kanban AI pipeline but uses **GitHub Projects v2 as the data store and UI** instead of SQLite and the Vite web board. No local server. No database files. GitHub IS the board.

The AI pipeline logic (Planner → Critic → Builder → Shield → Inspector → Ranger) is identical to the existing `kanban` skills. The only difference is the backend: all reads and writes go through the GitHub Projects v2 GraphQL API and GitHub Issues via the `gh` CLI.

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

**Dependency:** `gh` CLI authenticated with `project` and `repo` scopes (`gh auth login --scopes project,repo`).

---

## Data Model

### GitHub Project Fields

`/kanban-gh-init` creates these fields on the target GitHub Project (skips any that already exist with the correct type and options):

| Field | Type | Values |
|-------|------|--------|
| `Status` | Single select | `Todo` · `Plan` · `Plan Review` · `Implement` · `Impl Review` · `Test` · `Done` |
| `Priority` | Single select | `low` · `medium` · `high` |
| `Level` | Single select | `L1` · `L2` · `L3` |
| `Tags` | Text | Comma-separated free text |

**Status values are title-cased strings** (e.g. `"Plan Review"`, not `"plan-review"` or `"plan_review"`). These exact strings are used in GraphQL mutations.

### Field Conflict Handling

If a field named `Status` already exists but is the wrong type (e.g. `Text` instead of `Single select`), or is a single select but is missing required options, the skill must:

1. Warn the user: `"Field 'Status' exists but has wrong type/options. Cannot auto-fix."`
2. Show the current field state vs. what is required.
3. Exit with a prompt to manually fix the field in GitHub Projects settings, then re-run init.

The skill does **not** attempt to mutate or delete existing fields — this avoids data loss on projects that already have items.

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
  "repo": "evilsquid888/my-project",
  "retries": 2
}
```

**`ownerType`** valid values: `"user"` or `"organization"`. Determined during init by querying `gh api /repos/{owner}/{repo}` and checking the `owner.type` field. This affects GraphQL query shape:
- `user` → `query { user(login: $owner) { projectV2(number: $number) { ... } } }`
- `organization` → `query { organization(login: $owner) { projectV2(number: $number) { ... } } }`

### No Web Board

There is no `start.sh`, no local server, and no Vite app. The GitHub Projects UI at `github.com/users/{owner}/projects/{number}` (user) or `github.com/orgs/{owner}/projects/{number}` (org) is the board.

### Images

Images are handled natively by GitHub — users drag-and-drop attachments in issue comments via the browser. Agent comments use descriptive text only. Images already in the repo can be embedded via `https://raw.githubusercontent.com/...` URLs.

---

## `kanban-gh-init` Flow

1. Accept optional project URL or number as argument. If none provided, run `gh project list --owner @me --format json` and prompt user to pick from the list.
2. Determine `ownerType` by fetching the repo owner type via `gh api`.
3. Fetch existing project fields via GraphQL (`projectV2.fields` nodes). Check each required field by name.
4. For each missing field: create it via GraphQL mutation. For each existing field: verify type and options match. On mismatch, warn and exit (see Field Conflict Handling above).
5. Ask which repo issues should be created in (default: output of `gh repo view --json nameWithOwner`).
6. Write `.claude/kanban-gh.json`.
7. Output:
```
✅ kanban-gh initialized.

  Project: github.com/users/evilsquid888/projects/2
  Repo:    evilsquid888/my-project
  Config:  .claude/kanban-gh.json

Add tasks with /kanban-gh add <title>
```

---

## `/kanban-gh` CRUD Commands

| Command | Behavior |
|---------|----------|
| `list` | Fetch all project items via GraphQL, render markdown table (Issue#, Status, Priority, Level, Title) |
| `add <title>` | Create GitHub Issue, add to project, set Status=Todo. Prompt for priority, level, description, tags. |
| `move <ID\|name> <status>` | Update Status field via GraphQL mutation. Enforces valid pipeline transitions (see below). |
| `edit <ID\|name>` | Update issue body or field values interactively |
| `remove <ID\|name>` | Remove item from project; optionally close the issue |
| `stats` | Item counts per Status column |
| `context` | Pipeline state summary — what's in each column, what's active |

### ID Resolution

Commands accept:
- **Issue number**: `12` or `#12` — the `#` prefix is optional
- **Partial title**: case-insensitive substring match against item titles

If a title string matches multiple items: list all matches with their issue numbers and prompt the user to pick one.

In `--auto` mode (non-interactive): if multiple matches exist, the command **fails with an error** — never silently picks one.

### Valid Pipeline Transitions

Transitions are enforced by the `move` command and `kanban-gh-run`. Invalid moves return an error listing the allowed next statuses.

**L1 Quick:**
```
Todo → Implement → Done
```

**L2 Standard:**
```
Todo → Plan → Implement → Impl Review → Done
Plan → Todo (rejected plan, back to requirements)
Impl Review → Implement (rejected impl, back to build)
```

**L3 Full:**
```
Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
Plan → Todo
Plan Review → Plan (changes requested)
Impl Review → Implement
Test → Implement (test failures, back to build)
```

**Cross-level manual moves:** A user may manually call `move` to any valid next status regardless of level. The Level field determines which transitions the pipeline enforces automatically; a human operator can override via `move` with a confirmation prompt.

Tasks cannot skip directly to `Done` from any status other than `Implement` (L1), `Impl Review` (L2), or `Test` (L3).

---

## Pipeline — `/kanban-gh-run`

Identical pipeline levels and agent sequence as the existing kanban skills.

**Per-step execution:**

1. **Read** — fetch item from GitHub Project (field values) and issue (body + all comments) via GraphQL
2. **Run agent** — prompt templates from `shared/pipeline.md` (same as existing kanban: Planner, Critic, Builder, Shield, Inspector, Ranger)
3. **Write** — append signed agent comment to the issue: `gh issue comment {number} --repo {repo} --body "{content}"`
4. **Advance** — update Status field via GraphQL mutation to next pipeline column
5. **Pause or continue** per `--auto` flag (see below)

**Pause behavior:**

Pause steps are: `Plan Review` and `Impl Review`. At these steps, the pipeline prints the review comment and waits:
```
[kanban-gh] Waiting for your review. Options:
  approve  — continue to next stage
  reject   — send back with comments
  abort    — stop the pipeline here
```
In `--auto` mode, these pauses are skipped and the pipeline continues automatically.

`Test` is not a pause step — the Ranger agent runs autonomously and advances to `Done` or sends back to `Implement` based on results.

If the pipeline is aborted mid-run, the item's Status stays at whatever column it was last advanced to.

**Retry limit:**

Configurable via `--retries N` flag (default: `2`). The limit can also be set permanently in `.claude/kanban-gh.json` as `"retries": 3`. Flag takes precedence over config.

Each agent is allowed a maximum of `N` rejections per task before the pipeline pauses for human review — even in `--auto` mode. On the Nth rejection of the same agent (e.g. Builder rejected twice with default), the pipeline:
1. Posts a comment: `> ⚠️ [kanban-gh] Builder has been rejected 2 times. Pausing for human review.`
2. Leaves the item at its current Status column.
3. Exits and waits for the user to intervene (edit requirements or manually advance).

The rejection count resets to 0 each time an agent successfully passes its review step.

**Board loop mode (`--loop`):**

`/kanban-gh-run --loop` processes the entire backlog continuously: it picks the next `Todo` item, runs its full pipeline, then picks the next, until all `Todo` items are exhausted or a retry-limit pause occurs. Integrates with [Ralph Loop](https://github.com/cyanluna/cyanluna.skills) if available — if Ralph Loop is active in the session, `--loop` defers to it for scheduling. Without Ralph Loop, `--loop` runs sequentially.

Usage:
```
/kanban-gh-run --loop          # process all Todo items
/kanban-gh-run --loop --auto   # process all, skip review pauses (retry limit still applies)
```

**Done:** Final comment: `> ✅ Pipeline complete. All done-when criteria met.` Status set to `Done`.

---

## `/kanban-gh-refine` Flow

A structured requirements refinement interview that rewrites the issue body with a complete, actionable specification.

1. Fetch the issue title and current body.
2. Ask the user 3–5 targeted questions one at a time (using `AskUserQuestion` — the Claude Code native interactive prompt tool) to clarify: goal, scope, acceptance criteria, edge cases, constraints.
3. Rewrite the issue body with a structured spec:
   ```
   ## Goal
   ## Scope
   ## Acceptance Criteria
   ## Edge Cases
   ## Out of Scope
   ```
4. Update the issue body via `gh issue edit {number} --repo {repo} --body "{new_body}"`.
5. Confirm: `✅ Requirements updated for #12.`

The interview terminates after the 5th question or when the user indicates they have nothing more to add.

---

## `/kanban-gh-explore` Flow

Explores the codebase when you have a vague idea but don't know how to implement it. Seeds the GitHub Project with phased tasks.

1. Accept optional topic argument. If none, ask a clarifying question to understand the goal.
2. Validate context: check that the topic includes *why* and *where* (not just *what*). If missing, ask.
3. Launch an Explore subagent with a structured prompt covering: codebase structure, relevant code, pain points, constraints.
4. Produce a structured direction report:
   - Current state
   - Key findings
   - 2–3 directions with pros/cons
   - Recommendation
5. Present directions to the user; ask them to pick one (or "Cancel — save report only").
6. Create the anchor issue for the full exploration report (tagged `[Explore]`), add to project. Record its issue number.
7. On pick: create 3–7 phased issues in the repo, add each to the project with Status=Todo, Level=L3, tagged `phase:1`, `phase:2`, etc. Each issue body links back to the anchor report issue number from step 6.
8. Output summary of created issues.

**Cancel path:** If the user picks Cancel, only the anchor report issue is created — no implementation tasks.

---

## Required GraphQL Operations

The `shared/graphql.md` file must provide templates for these operations:

| Operation | Purpose |
|-----------|---------|
| `getProjectFields` | List all fields on a project (to detect existing fields) |
| `createField` | Create a single-select or text field on a project |
| `getProjectItems` | Fetch all items with field values (for `list`, `context`, `stats`) |
| `getProjectItem` | Fetch a single item by issue number (for `move`, `run`, etc.) |
| `updateFieldValue` | Set a single-select field value on an item (advance pipeline status) |
| `addIssueToProject` | Add an existing issue as a project item |
| `removeItemFromProject` | Remove a project item by itemId (`deleteProjectV2Item` mutation) |

All operations require: `projectId` (node ID of the project), `fieldId` (node ID of the field), `itemId` (node ID of the project item), `optionId` (node ID of the select option).

Node IDs are fetched at command runtime and not cached between skill invocations.

---

## Shared Files

Three shared reference files consumed by all skills, installed to `~/.claude/skills/shared/`:

| File | Contents |
|------|----------|
| `shared/graphql.md` | GraphQL query/mutation templates for all required operations (see above) |
| `shared/pipeline.md` | Pipeline levels, valid status transitions, agent prompt templates, Critic and Inspector scoring rubrics |
| `shared/schema.md` | Canonical field names, exact status option strings, config file format |

Skills reference shared files as `~/.claude/skills/shared/graphql.md` etc.

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
cp -R kanban-gh/shared            ~/.claude/skills/
```

No `pnpm install`. No server. No `start.sh`.

---

## Out of Scope

- Bidirectional sync with the SQLite-based kanban system
- A local web board
- Agent-side image uploads (GitHub API does not support this)
- Webhook/hook-based auto-sync
- Caching of GraphQL node IDs between invocations
