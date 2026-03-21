---
name: kanban-gh-explore
description: "Explore the codebase for uncertain implementation direction. Produces a direction report and creates phased tasks in GitHub Projects. Usage: /kanban-gh-explore [topic]"
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md for field names, status values, and config format. Read ~/.claude/skills/shared/graphql.md for GraphQL operations.

# kanban-gh-explore

Codebase exploration skill for uncertain implementation direction. Deeply explores the codebase, presents multiple implementation directions, and seeds the GitHub Project board with phased tasks. Does NOT write any source code.

---

## Setup

Run `/kanban-gh-init` before using this skill. It creates `.claude/kanban-gh.json` with the project connection details.

---

## Config Loading

Every command begins by loading config from `.claude/kanban-gh.json`:

```bash
CONFIG=$(cat .claude/kanban-gh.json 2>/dev/null)
PROJECT=$(echo "$CONFIG" | jq -r '.project')
OWNER=$(echo "$CONFIG" | jq -r '.owner')
OWNER_TYPE=$(echo "$CONFIG" | jq -r '.ownerType')
REPO=$(echo "$CONFIG" | jq -r '.repo')
RETRIES=$(echo "$CONFIG" | jq -r '.retries // 2')
```

If the config file is missing, exit with:

```
Error: .claude/kanban-gh.json not found. Run /kanban-gh-init first.
```

---

## Procedure

---

### ① Receive and validate topic

If a topic argument was provided, use it as `$TOPIC`.

If no argument was provided, use AskUserQuestion:

```
What do you want to explore?
Describe the feature, change, or problem area you're investigating:
```

Once a topic is in hand, check for missing context. Ask at most 2 clarification questions (in a single AskUserQuestion call) if any of the following are true:

- No indication of which codebase area is involved
- The "why" (motivation) is completely absent
- The scope is unbounded with no natural boundaries

Example clarification prompt (adapt to what's actually missing):

```
Before I start exploring, a couple of quick questions:

1. Which part of the codebase is most likely involved — e.g., frontend, API, auth, data layer?
2. What's the goal or problem this should solve?
```

Do not ask more than 2 questions. If you can infer reasonable answers from the topic, skip this step entirely.

---

### ② Deep codebase exploration

Launch an Agent tool subagent (`subagent_type="Explore"`) with the following structured prompt. Substitute `$TOPIC` where shown.

```
You are a codebase analyst performing a deep exploration for the topic: "$TOPIC"

Explore the codebase systematically and produce a structured report covering all four sections below. Cite specific file paths and line numbers for every claim. Do not speculate — if something is unclear, say so explicitly.

## A. PROJECT STRUCTURE
- List top-level directories and their apparent purpose
- Identify main entry files (e.g., index.ts, main.py, app.js, server.go)
- Note relevant config files (package.json, tsconfig.json, Makefile, pyproject.toml, etc.)

## B. TOPIC-RELEVANT CODE
- Which files and modules relate to "$TOPIC"?
- Trace the data flow or control flow relevant to the topic
- Identify the key abstractions, interfaces, and patterns in use
- Note any existing tests covering this area

## C. PAIN POINTS & GAPS
- Missing abstractions or responsibilities not yet modeled
- Duplication or inconsistency across modules
- Conflicts between current implementation and the stated goal
- Technical debt in the relevant area

## D. TECHNOLOGY CONSTRAINTS
- Libraries, frameworks, and tools in active use
- Test setup (framework, coverage tooling, conventions)
- Build and lint tooling
- Patterns that must be respected (e.g., "all DB access goes through the repository layer")

Return raw findings. Do not propose directions — just report what you observe.
```

Wait for the subagent to complete before continuing.

---

### ③ Write Exploration Report

Using the subagent's findings, compose the full exploration report in Markdown. Save it as `$FULL_REPORT`.

```markdown
## Exploration Report: <topic>
*Explored: <ISO 8601 timestamp>Z | Project: <REPO>*

### Current State
[2–4 sentences summarising the relevant area of the codebase as it exists today. Include 2–3 file references.]

### Key Findings
- <finding> (`path/to/file.ts:line`)
- <finding> (`path/to/file.ts:line`)
- ...

### Possible Directions

#### Direction A: <name>
**Approach**: [1–2 sentences describing the implementation strategy]
**Pros**:
- ...
**Cons**:
- ...
**Estimated complexity**: Low / Medium / High
**Files likely touched**:
- `path/to/file.ts`
- ...

#### Direction B: <name>
**Approach**: [1–2 sentences]
**Pros**:
- ...
**Cons**:
- ...
**Estimated complexity**: Low / Medium / High
**Files likely touched**:
- `path/to/file.ts`
- ...

<!-- Include Direction C if a third meaningfully distinct option exists -->

### Recommended Direction
[State which direction you recommend and WHY, citing specific codebase evidence (file paths, patterns observed). Do not recommend a direction you cannot justify from the findings.]
```

Rules for directions:
- Present 2–3 directions. Do not fabricate options — every direction must be grounded in the findings.
- If only one viable direction exists, present it as Direction A and explain why alternatives are not feasible.
- Keep direction names short and descriptive (e.g., "Extend existing middleware", "New standalone service", "Thin wrapper on library X").

---

### ④ Present report to user

Display the full `$FULL_REPORT` to the user, then use AskUserQuestion with the following options (adjust direction labels to match what was written):

```
Exploration complete. Which direction would you like to pursue?

  A — <Direction A name>
  B — <Direction B name>
  C — <Direction C name>   (if exists)
  Cancel — save the report as an issue only, don't create tasks
```

Save the user's answer as `$PICK`.

---

### ⑤ Create anchor issue (always — regardless of pick or cancel)

```bash
REPORT_URL=$(gh issue create \
  --repo "$REPO" \
  --title "[Explore] $TOPIC" \
  --body "$FULL_REPORT")
REPORT_NUMBER=$(echo "$REPORT_URL" | grep -oE '[0-9]+$')
```

Resolve the issue node ID and add it to the project:

```bash
ISSUE_NODE_ID=$(gh api /repos/$REPO/issues/$REPORT_NUMBER --jq .node_id)
```

Add to project via `addIssueToProject` (see graphql.md):

```bash
ADD_RESPONSE=$(gh api graphql -f query='
mutation {
  addProjectV2ItemById(input: {
    projectId: "'"$PROJECT_ID"'"
    contentId: "'"$ISSUE_NODE_ID"'"
  }) {
    item { id }
  }
}')
ITEM_ID=$(echo "$ADD_RESPONSE" | jq -r '.data.addProjectV2ItemById.item.id')
```

Resolve field IDs via `getProjectFields`, then set:
- **Status** = `Todo`
- **Level** = `L1`
- **Priority** = `low`
- **Tags** = `explore-report`

Use `updateFieldValue` (see graphql.md) for each field.

Save `$REPORT_NUMBER` for use in subsequent steps.

---

### ⑥ On Pick (not Cancel): plan and create phased implementation issues

Skip this step entirely if `$PICK` is `Cancel`.

**Planning phase — do this before creating any issues:**

Based on the chosen direction, plan 3–7 implementation tasks. Each task must be:
- Independently completable (no circular dependencies)
- Scoped to at most 3 unrelated files
- Ordered by phase (Phase 1 = foundational, later phases build on earlier ones)

Assign priority by phase:
- Phase 1: `high`
- Phase 2: `medium`
- Phase 3+: `low`

**Creation phase — create each task in order:**

For each planned task:

```bash
TASK_URL=$(gh issue create \
  --repo "$REPO" \
  --title "<task title>" \
  --body "$(cat <<'BODY'
---
## Exploration Context
**Explore report**: #$REPORT_NUMBER
**Direction chosen**: <direction name>
**Phase**: N of M
**Rationale**: [1–2 sentences explaining why this task exists and what it unblocks]
---

<task-specific requirements, acceptance criteria, and relevant file paths>
BODY
)")
TASK_NUMBER=$(echo "$TASK_URL" | grep -oE '[0-9]+$')
```

For each task, resolve its node ID, add to project, and set fields:

```bash
TASK_NODE_ID=$(gh api /repos/$REPO/issues/$TASK_NUMBER --jq .node_id)
```

Add to project via `addIssueToProject`, then set:
- **Status** = `Todo`
- **Level** = `L3`
- **Priority** = `high` / `medium` / `low` (by phase as above)
- **Tags** = `phase:N, explore-<topic-slug>` (where topic-slug is the topic lowercased with spaces replaced by hyphens)

Collect all created task numbers and titles into `$TASK_LIST` for use in step ⑦.

---

### ⑦ Patch anchor issue with Task Index (only if tasks were created)

Skip this step if `$PICK` is `Cancel`.

Build the task index table, then patch the anchor issue:

```bash
TASK_INDEX="## Task Index
| Phase | # | Title | Priority | Level |
|-------|---|-------|----------|-------|
| 1 | #<N> | <title> | high | L3 |
| 2 | #<N> | <title> | medium | L3 |
..."

gh issue edit $REPORT_NUMBER --repo $REPO --body "$FULL_REPORT

$TASK_INDEX"
```

---

### ⑧ Output summary

**If tasks were created**, print:

```
| Phase | # | Title | Priority |
|-------|---|-------|----------|
| 1     | #N | <title> | high |
| 2     | #N | <title> | medium |
...

Exploration complete. <N> tasks created in `Todo`.
Run `/kanban-gh-refine <ID>` on any task to add detail.
Run `/kanban-gh-run <ID>` when ready to execute.
Report saved to #<REPORT_NUMBER>.
```

**If Cancel was chosen**, print:

```
Report saved to #<REPORT_NUMBER>. No tasks created.
```

---

## Guardrails

- **No implementation**: This skill does NOT write source files, modify existing code, or produce implementation artifacts of any kind.
- **No assumptions**: If the codebase is ambiguous or the subagent returned conflicting signals, say so explicitly in the report rather than guessing.
- **Evidence-based**: Every claim in the report must cite a file path. Uncited claims must be labelled as uncertain.
- **Honest about uncertainty**: Do not fabricate implementation directions. If only one direction is viable, say so and explain why.
- **Task granularity**: Each phased task must touch at most 3 unrelated files. Split larger changes into multiple tasks.
- **Report is permanent**: The anchor issue is created regardless of whether the user picks a direction or cancels. It is never deleted by this skill.
- **Topic-slug format**: When building Tags, convert the topic to lowercase, replace spaces and special characters with hyphens, and truncate to 30 characters if needed.

---

## Error Handling

- **Config missing**: Exit with error directing user to run `/kanban-gh-init`.
- **Subagent returns no findings**: Report the failure and exit. Do not generate a report with empty sections.
- **Issue creation failure**: Report the `gh` error and exit. Do not proceed to create task issues if the anchor issue failed.
- **GraphQL field update failure**: Log the error but do not abort. Report which fields could not be set at the end.
- **Cancel at step ④**: Create the anchor issue (step ⑤) then exit after printing the cancel message. Do not create any task issues.
