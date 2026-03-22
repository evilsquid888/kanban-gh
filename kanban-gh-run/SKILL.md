---
name: kanban-gh-run
description: "Run the AI pipeline for kanban tasks. Dispatches agents (Planner, Critic, Builder, Shield, Inspector, Ranger) using GitHub Issues for I/O. Works in both project and repo mode. Usage: /kanban-gh-run <ID|name> [--auto] [--retries N] [--loop]"
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md, ~/.claude/skills/shared/graphql.md, and ~/.claude/skills/shared/pipeline.md for field definitions, GraphQL/label operations, pipeline levels, agent templates, and scoring rubrics.

# kanban-gh-run

Pipeline orchestration skill that dispatches AI agents and manages pipeline transitions for kanban tasks. Uses GitHub Issues as the I/O layer between agents. Works in both project mode (GitHub Projects v2) and repo mode (labels).

---

## Setup

Run `/kanban-gh-init` before using any command. It creates `.claude/kanban-gh.json` with the project connection details.

---

## Config Loading

Every command begins by loading config from `.claude/kanban-gh.json`:

```bash
CONFIG=$(cat .claude/kanban-gh.json 2>/dev/null)
MODE=$(echo "$CONFIG" | jq -r '.mode // "project"')
REPO=$(echo "$CONFIG" | jq -r '.repo')
OWNER=$(echo "$CONFIG" | jq -r '.owner')
RETRIES=$(echo "$CONFIG" | jq -r '.retries // 2')

if [ "$MODE" = "project" ]; then
  PROJECT=$(echo "$CONFIG" | jq -r '.project')
  OWNER_TYPE=$(echo "$CONFIG" | jq -r '.ownerType')
fi
```

If the config file is missing, exit with:

```
Error: .claude/kanban-gh.json not found. Run /kanban-gh-init first.
```

All subsequent operations branch on `$MODE`. Status reads/writes use GraphQL in project mode and label swaps in repo mode. Agent dispatch is mode-agnostic — agents read/write via `gh issue view/comment` which works identically in both modes.

---

## ID Resolution

Commands that accept `<ID|name>` resolve it identically to `/kanban-gh` — see that skill for the full procedure. In summary:

- Plain integer or `#`-prefixed → use as issue number directly.
- Non-numeric:
  - **Project mode:** case-insensitive substring match against item titles via `getProjectItems`.
  - **Repo mode:** search via `gh issue list --search "$ARG"`.
  - Zero matches → error. One match → use it. Multiple matches → list them and prompt (or error in `--auto` mode).

---

## Commands

---

### `/kanban-gh-run <ID|name> [--auto] [--retries N]`

Full pipeline run for a single task. Default behavior: pause at review steps for user approval. `--auto`: fully automatic, skip all pauses. `--retries N`: override retry limit (default from config or 2).

**Steps:**

1. Load config.
2. Resolve `<ID|name>` to an issue number and current field values (Status, Level).
   - **Project mode:** also resolve item node ID. Read Status/Level from `getProjectItems` field values.
   - **Repo mode:** read Status/Level from issue labels (parse `status:` and `level:` labels).
3. Determine the pipeline level from the Level field (`L1`, `L2`, or `L3`).
4. Parse `--auto` and `--retries N` flags.
5. Execute the orchestration loop (see Orchestration Loop below) from the current status forward.
6. On completion or pause, report the final status.

---

### `/kanban-gh-run step <ID|name>`

Execute only the next pipeline step for a task, then exit. Does not loop or continue to subsequent steps.

**Steps:**

1. Load config.
2. Resolve `<ID|name>` to issue number, current Status and Level (plus item node ID in project mode).
3. Determine the next agent to dispatch based on current Status and Level (see Agent Dispatch Table).
4. Execute the Agent Dispatch Procedure for that single agent.
5. Apply the resulting status transition.
6. Report what happened and exit.

---

### `/kanban-gh-run --loop [--auto] [--retries N]`

Process all Todo items sequentially. Pick next Todo, run full pipeline, repeat until exhausted or retry-limit pause.

**Steps:**

1. Load config.
2. Fetch all items with Status = `Todo` via `getProjectItems` + filter.
3. Sort by issue number ascending.
4. For each Todo item:
   - Run full pipeline: `/kanban-gh-run <number> [--auto] [--retries N]`
   - If retry-limit pause: stop loop, report which task paused and why.
   - If error: stop loop, report error.
5. When all items processed, report summary.

Integrates with Ralph Loop if available — if Ralph Loop is active, `--loop` defers to it for scheduling.

---

### `/kanban-gh-run review <ID|name>`

Trigger code review for a task currently in `Impl Review` status.

**Steps:**

1. Load config.
2. Resolve `<ID|name>` to issue number, item node ID, current Status.
3. Verify Status is `Impl Review`. If not, exit with:
   ```
   Error: #<NUMBER> is in <STATUS>, not Impl Review. Cannot run review.
   ```
4. Execute the Agent Dispatch Procedure for Inspector.
5. Present the review result and prompt for approve/reject/abort (same as normal pause behavior).

---

## Orchestration Loop (Level-Aware)

For each task, determine its Level field and run the appropriate pipeline from the current status forward.

### L1 Quick

```
Todo → dispatch Builder(opus) → dispatch Shield(sonnet) → commit → Done
```

### L2 Standard

```
Todo → dispatch Planner(opus) → Implement
Implement → dispatch Builder(opus) → dispatch Shield(sonnet) → Impl Review
Impl Review → dispatch Inspector(sonnet) → [pause for user unless --auto] → Done / reject → Implement
```

### L3 Full

```
Todo → dispatch Planner(opus) → Plan Review
Plan Review → dispatch Critic(sonnet) → [pause for user unless --auto] → Implement / reject → Plan
Implement → dispatch Builder(opus) → dispatch Shield(sonnet) → Impl Review
Impl Review → dispatch Inspector(sonnet) → [pause for user unless --auto] → Test / reject → Implement
Test → dispatch Ranger(sonnet) → Done / fail → Implement
```

The orchestrator loops until the task reaches `Done`, a pause point is hit (and `--auto` is not set), or a retry limit is reached.

---

## Agent Dispatch Procedure

For every agent dispatch, follow these steps:

```
① Read task:
   - Resolve ISSUE_REPO from the item's URL (see shared/schema.md Multi-Repo Resolution):
     - Project mode: content.url → extract owner/repo
     - Repo mode: .url → extract owner/repo
     - Fallback: use config $REPO if URL is missing
   - Issue data: gh issue view $NUMBER --repo $ISSUE_REPO --json title,body,comments
   - Field values:
     - Project mode: fetch from getProjectItems, filter by issue number
     - Repo mode: parse from issue labels (status:, level:, priority:)
   - Determine current status and level

② Read agent template from ~/.claude/skills/shared/pipeline.md
   (find the section for the appropriate agent)

③ Fill placeholders with actual data:
   $NUMBER, $ISSUE_REPO, $OWNER, $PROJECT_ID
   Previous agent comments: parse from issue comments using jq
   (find by signature header, e.g. "> **Planner**" to get the plan)

④ Dispatch Agent tool:
   Agent(description="<Nickname> for #<NUMBER>", model=<opus|sonnet>, prompt=<filled template>)

⑤ After agent completes: verify the comment was posted
   (gh issue view --json comments, check latest matches agent signature)
```

### Reading Previous Agent Comments

```bash
COMMENTS=$(gh issue view $NUMBER --repo $ISSUE_REPO --json comments --jq '.comments')

# Find most recent comment by agent signature
PLAN=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Planner**"))] | last | .body')
REVIEW=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Critic**"))] | last | .body')
IMPL=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Builder**"))] | last | .body')
TESTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Shield**"))] | last | .body')
CODE_REVIEW=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Inspector**"))] | last | .body')
TEST_RESULTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Ranger**"))] | last | .body')
```

### Verifying Agent Output

After each agent completes, verify the comment was posted:

```bash
LATEST=$(gh issue view $NUMBER --repo $ISSUE_REPO --json comments --jq '.comments | last | .body')
if ! echo "$LATEST" | grep -q '> \*\*'"$AGENT_NICKNAME"'\*\*'; then
  echo "Error: $AGENT_NICKNAME did not post a comment on #$NUMBER"
  exit 1
fi
```

---

## Agent Dispatch Table

| Current Status | Agent | Nickname | Model | Advances To |
|---------------|-------|----------|-------|-------------|
| `Todo` (L2/L3) | Plan Agent | `Planner` | `opus` | `Plan Review` (L3) or `Implement` (L2) |
| `Todo` (L1) | — | — | — | Skip directly to Builder dispatch at `Implement` |
| `Plan Review` | Review Agent | `Critic` | `sonnet` | `Implement` (approved) or `Plan` (rejected) |
| `Implement` | Worker + TDD | `Builder` then `Shield` | `opus` then `sonnet` | `Impl Review` |
| `Impl Review` | Code Review | `Inspector` | `sonnet` | `Test` (L3) or `Done` (L2) if approved, `Implement` if rejected |
| `Test` | Test Runner | `Ranger` | `sonnet` | `Done` (pass) or `Implement` (fail) |

### Status Transitions After Agent Completion

All status updates below use mode-aware writes:

- **Project mode:** `updateFieldValue` via GraphQL (see `shared/graphql.md`).
- **Repo mode:** swap labels via `gh issue edit --remove-label "status:$OLD" --add-label "status:$NEW"` (see `shared/graphql.md` section 10).

Convert display names to label values using `display_to_label` (see `shared/schema.md`) when in repo mode.

**After Planner completes** (L2/L3 only — Planner is skipped for L1):
- L2: Update Status → `Implement`.
- L3: Update Status → `Plan Review`.

**After Critic completes:**
- Parse verdict from comment: `approved` or `changes_requested`.
- `approved`: Update Status → `Implement`.
- `changes_requested`: Update Status → `Plan`. Re-dispatch Planner (subject to retry limit).

**After Builder + Shield complete** (both run at `Implement` status):
- Update Status → `Impl Review`.

**After Inspector completes:**
- Parse verdict: `approved` or `changes_requested`.
- `approved` + L3: Update Status → `Test`.
- `approved` + L2: Update Status → `Done`.
- `changes_requested`: Update Status → `Implement`. Re-dispatch Builder (subject to retry limit).

**After Ranger completes:**
- Parse verdict: `pass` or `fail`.
- `pass`: Update Status → `Done`.
- `fail`: Update Status → `Implement`. Re-dispatch Builder (subject to retry limit).

### Extracting Verdicts

```bash
# Extract verdict from a review/test comment
VERDICT=$(echo "$AGENT_COMMENT" | grep -oP '## Verdict: \K\S+')
# Result: "approved", "changes_requested", "pass", or "fail"
```

---

## Pause Behavior

Pause steps: `Plan Review` and `Impl Review`. At these steps, after the review agent (Critic or Inspector) completes, print the review comment and use AskUserQuestion:

```
[kanban-gh] Review complete for #<NUMBER>. Options:
  approve  — continue to next stage
  reject   — send back with comments
  abort    — stop the pipeline here
```

Handle the response:

- **approve**: Continue the pipeline forward (Critic → Implement, Inspector → Test or Done).
- **reject**: Move status backward (Critic → Plan, Inspector → Implement). Re-dispatch the upstream agent (subject to retry limit).
- **abort**: Leave status at current column. Exit the pipeline.

In `--auto` mode: skip pause, continue automatically based on the agent's verdict.

`Test` is NOT a pause step — Ranger runs autonomously. Its verdict (pass/fail) is applied without user input.

---

## Retry Limit

```
RETRIES = --retries flag || config.retries || 2
Track rejection_count per agent per task (in-memory during run)
```

### On Rejection

```
rejection_count[agent] += 1
if rejection_count[agent] >= RETRIES:
  Post comment via gh issue comment:
  "> ⚠️ [kanban-gh] {Agent} has been rejected {N} times. Pausing for human review."
  Exit (even in --auto mode)
else:
  Move status back, re-run agent
```

### Counting Rejections

Track rejection counts in-memory during the current run. Alternatively, count from issue comments:

```bash
# Count Critic rejections since last approval
CRITIC_REJECTIONS=$(echo "$COMMENTS" | jq '[
  .[] | select(.body | startswith("> **Critic**"))
  | {body, approved: (.body | test("## Verdict: approved"))}
] | [foreach .[] as $c (0; if $c.approved then 0 else . + 1)] | last // 0')
```

### Reset

Rejection count resets to 0 when the agent posts an `approved` or `pass` verdict.

---

## `--loop` Mode

```
Fetch all items with Status = "Todo"
Sort by issue number ascending
For each:
  Run full pipeline (/kanban-gh-run <number> [--auto] [--retries N])
  If retry-limit pause: stop loop, report which task paused
  If error: stop loop, report error
```

### Fetching Todo Items

**Project mode:**

```bash
# After fetching all items via getProjectItems
TODO_ITEMS=$(echo "$ITEMS" | jq '[
  .[] | select(
    .fieldValues.nodes[] |
    select(.field.name == "Status") | .name == "Todo"
  )
] | sort_by(.content.number)')
```

**Repo mode:**

```bash
# Fetch Todo issues directly by label
TODO_ITEMS=$(gh issue list --repo "$REPO" --state open --label "status:todo" --json number,title,labels --limit 100 | jq 'sort_by(.number)')
```

### Loop Output

```
[kanban-gh] --loop: Found <N> Todo items.
[kanban-gh] Processing #<NUMBER>: <TITLE> (Level: <LEVEL>)
...
[kanban-gh] #<NUMBER> complete → Done
[kanban-gh] Processing #<NUMBER>: <TITLE> (Level: <LEVEL>)
...
[kanban-gh] Loop complete. <N> tasks processed, <M> moved to Done.
```

If a retry-limit pause occurs:

```
[kanban-gh] Loop paused at #<NUMBER>: <Agent> rejected <N> times. Needs human review.
```

Integrates with Ralph Loop if available — if Ralph Loop is active, `--loop` defers to it for scheduling.

---

## Done Transition

When the pipeline determines a task should move to Done:

```bash
# 1. Commit any working changes
if [ -n "$(git status --porcelain 2>/dev/null)" ]; then
  git add -A
  git commit -m "feat: <TITLE> [kanban-gh #<NUMBER>]"
fi
COMMIT_HASH=$(git rev-parse --short HEAD 2>/dev/null || echo "no-git")

# 2. Update Status to Done (mode-aware)
```

**Project mode:**

```bash
DONE_OPTION_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .options[] | select(.name == "Done") | .id')
gh api graphql -f query='
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "'"$PROJECT_ID"'"
    itemId: "'"$ITEM_ID"'"
    fieldId: "'"$STATUS_FIELD_ID"'"
    value: { singleSelectOptionId: "'"$DONE_OPTION_ID"'" }
  }) {
    projectV2Item { id }
  }
}'
```

**Repo mode:**

```bash
# Get current status label
OLD_STATUS_LABEL=$(gh issue view $NUMBER --repo "$ISSUE_REPO" --json labels --jq '[.labels[].name | select(startswith("status:"))] | first // ""')
# Swap to done
if [ -n "$OLD_STATUS_LABEL" ]; then
  gh issue edit $NUMBER --repo "$ISSUE_REPO" --remove-label "$OLD_STATUS_LABEL" --add-label "status:done"
else
  gh issue edit $NUMBER --repo "$ISSUE_REPO" --add-label "status:done"
fi
```

```bash
# 3. Post final comment
gh issue comment $NUMBER --repo $ISSUE_REPO --body "> ✅ Pipeline complete. All done-when criteria met.
> Commit: $COMMIT_HASH"
```

---

## Error Handling

- **Config missing**: Exit with error directing user to run `/kanban-gh-init`.
- **Item not found**: Exit with error from ID resolution.
- **GraphQL failure**: Report the error, do not transition status.
- **Agent fails to post comment**: Report the error, do not transition status.
- **Invalid status for command**: Report current status and what was expected.
- **Abort**: Leave status unchanged, exit cleanly.
