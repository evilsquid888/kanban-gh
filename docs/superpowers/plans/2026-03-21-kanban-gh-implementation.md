# kanban-gh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone set of Claude Code skills that replaces the SQLite + Vite kanban system with GitHub Projects v2 as the data store and UI.

**Architecture:** Pure Claude Code skills communicating with GitHub via `gh` CLI and GraphQL. No local server, no database, no web board. Agent outputs are stored as issue comments; status and metadata are tracked via GitHub Project custom fields.

**Tech Stack:** Claude Code skills (Markdown), `gh` CLI, GitHub Projects v2 GraphQL API, GitHub Issues

**Spec:** `docs/superpowers/specs/2026-03-21-kanban-gh-design.md`

---

## File Map

| File | Responsibility |
|------|---------------|
| `shared/schema.md` | Canonical field names, status strings, config format, agent nicknames |
| `shared/graphql.md` | All GraphQL query/mutation templates with placeholder variables |
| `shared/pipeline.md` | Pipeline levels, valid transitions, agent prompt templates, scoring rubrics |
| `kanban-gh-init/SKILL.md` | Project init: connect to GitHub Project, create fields, write config |
| `kanban-gh/SKILL.md` | CRUD commands: add, list, move, edit, remove, stats, context |
| `kanban-gh-run/SKILL.md` | Pipeline orchestration: dispatch agents, manage transitions |
| `kanban-gh-refine/SKILL.md` | Requirements refinement interview |
| `kanban-gh-explore/SKILL.md` | Codebase exploration and task seeding |
| `README.md` | Installation, usage, architecture overview |

---

### Task 1: Shared Schema (`shared/schema.md`)

Foundation file — all other files reference this for canonical names.

**Files:**
- Create: `shared/schema.md`

- [ ] **Step 1: Create `shared/schema.md`**

Write the complete schema reference containing:

1. **Config file format** — `.claude/kanban-gh.json` with all fields: `project`, `owner`, `ownerType` (user|organization), `repo`, `retries` (default 2)
2. **GitHub Project field definitions** — exact field names, types, and option values:
   - `Status` (Single select): `Todo`, `Plan`, `Plan Review`, `Implement`, `Impl Review`, `Test`, `Done`
   - `Priority` (Single select): `low`, `medium`, `high`
   - `Level` (Single select): `L1`, `L2`, `L3`
   - `Tags` (Text): comma-separated
3. **Agent nicknames table** — same as existing kanban:

| Nickname | Role | Model | Writes to (issue comment section) |
|----------|------|-------|-----------------------------------|
| `Planner` | Plan Agent | `opus` | Plan + Decision Log + Done When |
| `Critic` | Plan Review | `sonnet` | Review verdict + scores |
| `Builder` | Worker | `opus` | Implementation notes |
| `Shield` | TDD Tester | `sonnet` | Test notes (appended to Builder's comment thread) |
| `Inspector` | Code Review | `sonnet` | Review verdict + scores |
| `Ranger` | Test Runner | `sonnet` | Test results |

4. **Signature header rule** — every agent comment starts with:
```
> **[Nickname]** `[model]` · [ISO 8601 timestamp]Z
```

5. **Config loading snippet**:
```bash
CONFIG=$(cat .claude/kanban-gh.json 2>/dev/null)
PROJECT=$(echo "$CONFIG" | jq -r '.project')
OWNER=$(echo "$CONFIG" | jq -r '.owner')
OWNER_TYPE=$(echo "$CONFIG" | jq -r '.ownerType')
REPO=$(echo "$CONFIG" | jq -r '.repo')
RETRIES=$(echo "$CONFIG" | jq -r '.retries // 2')
```

- [ ] **Step 2: Verify references**

Check that all status strings are title-cased, agent nicknames match the spec, and config fields match the spec's JSON example.

- [ ] **Step 3: Commit**

```bash
git add shared/schema.md
git commit -m "feat: add shared schema reference"
```

---

### Task 2: Shared GraphQL (`shared/graphql.md`)

All GraphQL operations needed by the skills. Templates use `$VARIABLES` for runtime substitution.

**Files:**
- Create: `shared/graphql.md`

- [ ] **Step 1: Create `shared/graphql.md`**

Write templates for all 7 operations from the spec:

1. **`getProject`** — Resolve project node ID from owner + number. Must handle both `user` and `organization` owner types:
```graphql
# For user-owned projects:
query {
  user(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}

# For org-owned projects:
query {
  organization(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}
```

2. **`getProjectFields`** — List all fields with their types and options:
```graphql
query {
  node(id: "$PROJECT_ID") {
    ... on ProjectV2 {
      fields(first: 50) {
        nodes {
          ... on ProjectV2Field { id name dataType }
          ... on ProjectV2SingleSelectField {
            id name dataType
            options { id name }
          }
        }
      }
    }
  }
}
```

3. **`createField`** — Create a single-select field with options:
```graphql
mutation {
  createProjectV2Field(input: {
    projectId: "$PROJECT_ID"
    dataType: SINGLE_SELECT
    name: "$FIELD_NAME"
    singleSelectOptions: [
      {name: "$OPTION_1", color: BLUE},
      ...
    ]
  }) {
    projectV2Field { id }
  }
}
```
Also include a text field variant.

4. **`getProjectItems`** — Fetch all items with field values:
```graphql
query {
  node(id: "$PROJECT_ID") {
    ... on ProjectV2 {
      items(first: 100) {
        nodes {
          id
          content {
            ... on Issue {
              number
              title
              body
              url
            }
          }
          fieldValues(first: 20) {
            nodes {
              ... on ProjectV2ItemFieldSingleSelectValue {
                field { ... on ProjectV2SingleSelectField { name } }
                name
              }
              ... on ProjectV2ItemFieldTextValue {
                field { ... on ProjectV2Field { name } }
                text
              }
            }
          }
        }
      }
    }
  }
}
```

5. **`updateFieldValue`** — Set a single-select field value:
```graphql
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "$PROJECT_ID"
    itemId: "$ITEM_ID"
    fieldId: "$FIELD_ID"
    value: { singleSelectOptionId: "$OPTION_ID" }
  }) {
    projectV2Item { id }
  }
}
```

6. **`addIssueToProject`** — Add an issue to a project:
```graphql
mutation {
  addProjectV2ItemById(input: {
    projectId: "$PROJECT_ID"
    contentId: "$ISSUE_NODE_ID"
  }) {
    item { id }
  }
}
```

7. **`removeItemFromProject`** — Remove an item:
```graphql
mutation {
  deleteProjectV2Item(input: {
    projectId: "$PROJECT_ID"
    itemId: "$ITEM_ID"
  }) {
    deletedItemId
  }
}
```

Include a **Usage** section showing how to call each via `gh api graphql`:
```bash
gh api graphql -f query='<QUERY>' -f owner="$OWNER" -F number=$PROJECT_NUMBER
```

- [ ] **Step 2: Verify all 7 operations from spec are covered**

Cross-check against the Required GraphQL Operations table in the spec.

- [ ] **Step 3: Commit**

```bash
git add shared/graphql.md
git commit -m "feat: add shared GraphQL query/mutation templates"
```

---

### Task 3: Shared Pipeline (`shared/pipeline.md`)

Pipeline definitions, transition rules, and agent prompt templates adapted for GitHub Issues (comments instead of DB fields).

**Files:**
- Create: `shared/pipeline.md`

- [ ] **Step 1: Write pipeline levels and transitions**

Identical to existing kanban `shared.md` but referencing GitHub Project Status field values (title-cased):

```
L1 Quick:  Todo → Implement → Done
L2 Standard: Todo → Plan → Implement → Impl Review → Done
             Plan → Todo | Impl Review → Implement
L3 Full:   Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
           Plan → Todo | Plan Review → Plan | Impl Review → Implement | Test → Implement
```

- [ ] **Step 2: Write agent context flow**

Map how agents read/write via GitHub Issues instead of DB fields:

| Agent | Reads | Writes | How |
|-------|-------|--------|-----|
| `Planner` | Issue body (description) | Issue comment (plan + decision log + done_when) | `gh issue comment` |
| `Critic` | Issue body + Planner comment | Issue comment (review verdict) | `gh issue comment` |
| `Builder` | Issue body + Planner comment + Critic comment | Issue comment (impl notes) | `gh issue comment` |
| `Shield` | Issue body + Builder comment | Issue comment (test notes) | `gh issue comment` |
| `Inspector` | Issue body + Planner comment + Builder comment + Shield comment | Issue comment (review verdict) | `gh issue comment` |
| `Ranger` | Builder comment + Shield comment | Issue comment (test results) | `gh issue comment` |

**Reading previous agent comments:** Agents must fetch all comments via `gh issue view <NUMBER> --repo <REPO> --json comments` and identify relevant ones by signature header (e.g., find the comment starting with `> **Planner**` to read the plan).

- [ ] **Step 3: Write agent prompt templates**

Adapt all 6 templates from the existing kanban (`kanban/templates/*.md`) to use GitHub instead of localhost API:

**Key changes for ALL templates:**
- Replace `curl` API calls with `gh issue comment` for writing
- Replace `curl GET /api/task` with `gh issue view --json` for reading
- Replace `curl PATCH` status updates with `gh api graphql` mutations
- Agent reads previous outputs by parsing issue comments (find by signature header)
- Include the `shared/graphql.md` reference for status update mutation

**For each template, include:**
1. Identity block (nickname, model, task reference)
2. What to read (issue body + which previous agent comments)
3. How to read (`gh issue view <NUMBER> --repo <REPO> --json body,comments`)
4. Output format (same markdown structure as existing templates)
5. How to write (`gh issue comment <NUMBER> --repo <REPO> --body "<content>"`)
6. How to update Status field (GraphQL mutation from `shared/graphql.md`)
7. Guidelines and scoring rubrics (identical to existing)

**Planner template changes:**
- Reads: issue body
- Writes: issue comment with Plan + Decision Log + Done When sections
- Advances: Status → `Plan Review` (L3) or `Implement` (L2, skip review)

**Critic template changes:**
- Reads: issue body + Planner's comment (find by `> **Planner**` header)
- Writes: issue comment with scoring table + verdict
- Advances: Status → `Implement` (approved) or `Plan` (rejected)
- Scoring rubrics: identical 3-dimension rubric (Clarity, Done-When Quality, Reversibility)

**Builder template changes:**
- Reads: issue body + Planner's comment + Critic's comment (if exists)
- Writes: issue comment with impl notes, files modified, done_when verification
- Does NOT update status (same as existing — Shield runs next)

**Shield template changes:**
- Reads: issue body + Builder's comment
- Writes: issue comment with test notes (new comment, not appending to Builder's)
- Does NOT update status

**Inspector template changes:**
- Reads: issue body + Planner's comment + Builder's comment + Shield's comment
- Writes: issue comment with 7-dimension scoring + verdict
- Advances: Status → `Test` (approved) or `Implement` (rejected)
- Scoring rubrics: identical 7-dimension rubric

**Ranger template changes:**
- Reads: Builder's comment + Shield's comment
- Writes: issue comment with lint/build/test results
- Advances: Status → `Done` (pass) or `Implement` (fail)

- [ ] **Step 4: Add retry limit handling**

Include a section on how the orchestrator tracks rejection counts and pauses at the configurable limit (default 2, from `--retries` flag or config).

- [ ] **Step 5: Verify completeness**

Cross-check that every agent from the spec is covered, scoring rubrics match, and the signature format is consistent.

- [ ] **Step 6: Commit**

```bash
git add shared/pipeline.md
git commit -m "feat: add shared pipeline definitions and agent templates"
```

---

### Task 4: `kanban-gh-init` Skill

**Files:**
- Create: `kanban-gh-init/SKILL.md`

- [ ] **Step 1: Write SKILL.md with frontmatter**

```yaml
---
name: kanban-gh-init
description: "Connect to a GitHub Project, create pipeline fields, and write local config. Usage: /kanban-gh-init [project-url-or-number]"
license: MIT
---
```

- [ ] **Step 2: Write the full init procedure**

Following the spec's init flow (steps 1–7):

1. **Parse argument** — accept URL (`https://github.com/users/X/projects/N`), number, or none
2. **List projects if needed** — `gh project list --owner @me --format json` → AskUserQuestion to pick
3. **Determine ownerType** — `gh api /repos/{owner}/{repo}` → check `owner.type`
4. **Resolve project node ID** — GraphQL query from `shared/graphql.md` (`getProject`)
5. **Check existing fields** — GraphQL query (`getProjectFields`)
6. **Create missing fields** — GraphQL mutations (`createField`). Handle conflicts per spec (warn + exit if wrong type)
7. **Ask target repo** — default to `gh repo view --json nameWithOwner`
8. **Write `.claude/kanban-gh.json`** — using Write tool
9. **Output confirmation**

Include reference: `> Shared context: read ~/.claude/skills/shared/schema.md and ~/.claude/skills/shared/graphql.md`

- [ ] **Step 3: Add existing config detection**

If `.claude/kanban-gh.json` already exists, show current config and ask overwrite/keep.

- [ ] **Step 4: Add gh auth check**

At the start, verify `gh auth status` succeeds and has required scopes.

- [ ] **Step 5: Commit**

```bash
git add kanban-gh-init/SKILL.md
git commit -m "feat: add kanban-gh-init skill"
```

---

### Task 5: `kanban-gh` CRUD Skill

**Files:**
- Create: `kanban-gh/SKILL.md`

- [ ] **Step 1: Write SKILL.md with frontmatter**

```yaml
---
name: kanban-gh
description: "Manage tasks in GitHub Projects. Supports add, list, move, edit, remove, stats, context. Uses GitHub Projects v2 as the data store. Run /kanban-gh-init first."
license: MIT
---
```

- [ ] **Step 2: Write shared context reference and config loading**

Reference `shared/schema.md`, `shared/graphql.md`. Include config loading snippet.

- [ ] **Step 3: Write all 7 commands**

For each command, write the complete procedure using `gh` CLI:

**`/kanban-gh list`:**
- Fetch all items via GraphQL (`getProjectItems`)
- Parse field values from response
- Render markdown table: `| # | Status | Priority | Level | Title |`

**`/kanban-gh add <title>`:**
- AskUserQuestion for priority, level, description, tags
- Create issue: `gh issue create --repo $REPO --title "$TITLE" --body "$DESCRIPTION"`
- Add to project: GraphQL `addIssueToProject`
- Set field values: GraphQL `updateFieldValue` for Status=Todo, Priority, Level, Tags

**`/kanban-gh move <ID|name> <status>`:**
- Resolve ID (see ID resolution section)
- Validate transition against pipeline level (reference `shared/pipeline.md`)
- Update Status field via GraphQL `updateFieldValue`

**`/kanban-gh edit <ID|name>`:**
- Fetch current values
- AskUserQuestion for which fields to change
- Update issue body: `gh issue edit <NUMBER> --repo $REPO --body "$NEW_BODY"`
- Update project fields via GraphQL

**`/kanban-gh remove <ID|name>`:**
- Remove from project: GraphQL `removeItemFromProject`
- Ask whether to also close the issue
- If yes: `gh issue close <NUMBER> --repo $REPO`

**`/kanban-gh stats`:**
- Fetch all items, count by Status field value
- Render summary table

**`/kanban-gh context`:**
- Fetch all items, group by Status
- Output pipeline state summary (what's in each column, what's active)

- [ ] **Step 4: Write ID resolution section**

- Issue number: `12` or `#12` (strip `#` prefix)
- Partial title: case-insensitive substring match via `getProjectItems` + filter
- Multiple matches: list them, AskUserQuestion to pick
- `--auto` mode: multiple matches → error

- [ ] **Step 5: Commit**

```bash
git add kanban-gh/SKILL.md
git commit -m "feat: add kanban-gh CRUD skill"
```

---

### Task 6: `kanban-gh-run` Pipeline Orchestration Skill

**Files:**
- Create: `kanban-gh-run/SKILL.md`

- [ ] **Step 1: Write SKILL.md with frontmatter**

```yaml
---
name: kanban-gh-run
description: "Run the AI pipeline for GitHub Projects kanban tasks. Dispatches agents (Planner, Critic, Builder, Shield, Inspector, Ranger) using GitHub Issues for I/O. Usage: /kanban-gh-run <ID|name> [--auto] [--retries N] [--loop]"
license: MIT
---
```

- [ ] **Step 2: Write the orchestration loop**

Adapt from existing `kanban-run/SKILL.md` but replace all API calls with GitHub equivalents.

**Commands:**
- `/kanban-gh-run <ID|name>` — full pipeline, pause at reviews
- `/kanban-gh-run <ID|name> --auto` — fully automatic
- `/kanban-gh-run step <ID|name>` — single step then exit
- `/kanban-gh-run --loop [--auto]` — process all Todo items
- `/kanban-gh-run review <ID|name>` — trigger code review

**Orchestration loop (level-aware):**

Same L1/L2/L3 flows as existing kanban-run. For each agent step:

```
① Read task: gh issue view <NUMBER> --repo $REPO --json title,body,comments
   Parse field values from GitHub Project (GraphQL getProjectItems filtered by issue number)

② Read agent template from ~/.claude/skills/shared/pipeline.md

③ Fill placeholders:
   <NUMBER>, <REPO>, <OWNER>, <PROJECT_ID>, <title>, <body>,
   <planner_comment>, <critic_comment>, <builder_comment>,
   <shield_comment>, <TIMESTAMP>

   To fill agent comment placeholders: parse the issue comments array,
   find the most recent comment matching each agent's signature header
   (e.g., > **Planner** to find the plan)

④ Launch Agent tool (subagent_type="general-purpose", model=<opus|sonnet>)

⑤ After agent completes: verify the comment was posted
   (gh issue view --json comments, check latest)
```

- [ ] **Step 3: Write pause behavior**

At `Plan Review` and `Impl Review` steps:
- Print the review comment
- AskUserQuestion: approve / reject / abort
- On reject: move Status back, increment rejection counter
- On abort: leave Status as-is, exit

In `--auto` mode: skip pause, continue automatically.

- [ ] **Step 4: Write retry limit handling**

```
RETRIES = --retries flag || config.retries || 2
Track rejection_count per agent per task (in-memory during run)

On rejection:
  rejection_count[agent] += 1
  if rejection_count[agent] >= RETRIES:
    Post comment: "> ⚠️ [kanban-gh] {Agent} has been rejected {N} times. Pausing for human review."
    Exit (even in --auto mode)
  else:
    Move status back, re-run agent
```

- [ ] **Step 5: Write `--loop` mode**

```
Fetch all items with Status = "Todo" (via getProjectItems + filter)
Sort by issue number ascending
For each:
  Run full pipeline (/kanban-gh-run <number> [--auto])
  If retry-limit pause: stop loop, report which task paused
```

- [ ] **Step 6: Write done transition**

```bash
# 1. Commit any working changes
if [ -n "$(git status --porcelain 2>/dev/null)" ]; then
  git add -A
  git commit -m "feat: <TITLE> [kanban-gh #<NUMBER>]"
fi
COMMIT_HASH=$(git rev-parse --short HEAD 2>/dev/null || echo "no-git")

# 2. Update Status to Done via GraphQL
# 3. Post final comment:
#    > ✅ Pipeline complete. All done-when criteria met.
#    > Commit: $COMMIT_HASH
```

- [ ] **Step 7: Commit**

```bash
git add kanban-gh-run/SKILL.md
git commit -m "feat: add kanban-gh-run pipeline orchestration skill"
```

---

### Task 7: `kanban-gh-refine` Skill

**Files:**
- Create: `kanban-gh-refine/SKILL.md`

- [ ] **Step 1: Write SKILL.md with frontmatter**

```yaml
---
name: kanban-gh-refine
description: "Refine backlog requirements through structured user interview. Updates the GitHub issue body with a clear specification. Usage: /kanban-gh-refine <ID|name>"
license: MIT
---
```

- [ ] **Step 2: Write the refinement procedure**

Adapt from existing `kanban-refine/SKILL.md`:

```
① Resolve ID (issue number or partial title match)
② Read issue: gh issue view <NUMBER> --repo $REPO --json title,body
③ Display current state (title + body)
④ Identify gaps across: WHAT, WHY, SCOPE, ACCEPTANCE, CONSTRAINTS, EDGE CASES, DEPENDENCIES
⑤ Interview user (1–4 questions per round, max 3 rounds, AskUserQuestion)
⑥ Synthesize into structured spec:
   ## Goal
   ## Scope (IN/OUT)
   ## Acceptance Criteria
   ## Edge Cases
   ## Out of Scope
   (omit empty sections)
⑦ Present refined description, ask: Approve / Edit more / Cancel
⑧ Save: gh issue edit <NUMBER> --repo $REPO --body "$NEW_BODY"
⑨ Optionally update project fields (Priority, Level) if discussed
⑩ Post agent_log comment:
   > **Refiner** `sonnet` · <TIMESTAMP>
   > Requirements refined. Updated issue body with structured spec.
```

- [ ] **Step 3: Commit**

```bash
git add kanban-gh-refine/SKILL.md
git commit -m "feat: add kanban-gh-refine requirements refinement skill"
```

---

### Task 8: `kanban-gh-explore` Skill

**Files:**
- Create: `kanban-gh-explore/SKILL.md`

- [ ] **Step 1: Write SKILL.md with frontmatter**

```yaml
---
name: kanban-gh-explore
description: "Explore the codebase for uncertain implementation direction. Produces a direction report and creates phased tasks in GitHub Projects. Usage: /kanban-gh-explore [topic]"
license: MIT
---
```

- [ ] **Step 2: Write the exploration procedure**

Adapt from existing `kanban-explore/SKILL.md`:

```
① Receive and validate topic
   If missing → clarification interview (max 2 questions)
   Check for missing context (which area, why, scope)

② Deep codebase exploration (Agent tool → Explore subagent)
   Structured prompt: structure, relevant code, pain points, constraints

③ Write Exploration Report:
   ## Exploration Report: <topic>
   *Explored: <timestamp> | Project: <REPO>*
   ### Current State
   ### Key Findings
   ### Possible Directions (2-3 with pros/cons/complexity/files)
   ### Recommended Direction

④ Present report + AskUserQuestion to choose direction
   Options: Direction A / B / C / Cancel

⑤ Create anchor issue FIRST (Cancel or Pick):
   gh issue create --repo $REPO --title "[Explore] <topic>" --body "$REPORT"
   Add to project, set Status=Todo, Level=L1, Tags=explore-report
   Save anchor issue number as $REPORT_NUMBER

⑥ On Pick: create 3–7 phased implementation issues:
   Each issue body includes:
   ---
   ## Exploration Context
   **Explore report**: #$REPORT_NUMBER
   **Direction chosen**: <name>
   **Phase**: N of M
   **Rationale**: ...
   ---
   Add each to project, set Status=Todo, Level=L3, Tags=phase:N

⑦ Patch anchor issue with Task Index table:
   gh issue edit $REPORT_NUMBER --repo $REPO --body "$REPORT\n\n## Task Index\n| Phase | # | Title |"

⑧ Output summary

Cancel path: only anchor issue created, no implementation tasks
```

- [ ] **Step 3: Commit**

```bash
git add kanban-gh-explore/SKILL.md
git commit -m "feat: add kanban-gh-explore codebase exploration skill"
```

---

### Task 9: README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Cover:
- Project title and description
- Prerequisites (`gh` CLI, authentication)
- Installation (copy skills to `~/.claude/skills/`)
- Quick Start (`/kanban-gh-init`, `/kanban-gh add`, `/kanban-gh-run`)
- Commands reference (all 5 skills with subcommands)
- Pipeline overview (7-column flow, L1/L2/L3 levels)
- AI Team table (6 agents with roles and models)
- Architecture diagram (text: skills → gh CLI → GitHub Projects v2 GraphQL → GitHub Issues)
- License (MIT)

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with installation and usage guide"
```

---

### Task 10: Final Review & Push

- [ ] **Step 1: Verify all files exist**

```bash
ls -la shared/schema.md shared/graphql.md shared/pipeline.md \
  kanban-gh-init/SKILL.md kanban-gh/SKILL.md kanban-gh-run/SKILL.md \
  kanban-gh-refine/SKILL.md kanban-gh-explore/SKILL.md README.md
```

- [ ] **Step 2: Cross-reference check**

Verify:
- All SKILL.md files reference `~/.claude/skills/shared/` correctly
- Status strings match across all files (title-cased)
- Agent nicknames and models are consistent
- GraphQL operation names in skills match `shared/graphql.md`
- Pipeline transitions in `kanban-gh-run` match `shared/pipeline.md`

- [ ] **Step 3: Run triple code review**

Dispatch `superpowers:code-reviewer` agent 3 times. Each pass:
1. Review all skill files against the spec
2. Check for consistency, completeness, and correctness
3. Fix any issues found before next pass

- [ ] **Step 4: Push to remote**

```bash
git push origin main
```
