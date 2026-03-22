# Shared Schema Reference

This is the canonical reference for field names, status values, agent identities, and config structure used across all kanban-gh skills.

---

## Config File

Location: `.claude/kanban-gh.json`

Two modes are supported: **project** (default, uses GitHub Projects v2) and **repo** (uses repo labels only).

**Project mode** (default — backward compatible with existing configs):

```json
{
  "mode": "project",
  "project": "2",
  "owner": "evilsquid888",
  "ownerType": "user",
  "repo": "evilsquid888/my-project",
  "retries": 2
}
```

**Repo mode** (label-based, no Project board required):

```json
{
  "mode": "repo",
  "repo": "evilsquid888/my-project",
  "owner": "evilsquid888",
  "retries": 2
}
```

### Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `mode` | `"project"` \| `"repo"` | Tracking mode | `"project"` (if absent) |
| `project` | string | GitHub Projects v2 project number | required (project mode only) |
| `owner` | string | GitHub org or user login | required |
| `ownerType` | `"user"` \| `"organization"` | Owner account type | required (project mode only) |
| `repo` | string | Repository in `owner/repo` format | required |
| `retries` | number | Number of retry attempts for API calls | `2` |

### Config Loading Snippet

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

### Backward Compatibility

If the `mode` field is absent from the config, treat it as `"project"`. All existing project-mode configs continue to work without modification.

### Multi-Repo Resolution

In **project mode**, a GitHub Project can track issues from multiple repos. Each item's `content.url` contains the actual repo (e.g., `https://github.com/owner/frontend/issues/42`). For all per-issue operations (view, comment, edit, close), resolve the repo from the item's URL rather than using the config `$REPO`.

In **repo mode**, all issues come from the config repo, so `$ISSUE_REPO` always equals `$REPO`.

**Helper function:**

```bash
resolve_repo_from_url() {
  # Extract "owner/repo" from a GitHub issue URL
  # Input:  https://github.com/owner/repo/issues/42
  # Output: owner/repo
  local URL="$1"
  echo "$URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/issues/[0-9]+|\1|'
}
```

**Per-issue repo resolution pattern:**

```bash
# Project mode: URL is in content.url
ISSUE_URL=$(echo "$ITEM" | jq -r '.content.url // ""')

# Repo mode: URL is in .url
ISSUE_URL=$(echo "$ITEM" | jq -r '.url // ""')

# Resolve repo, fall back to config $REPO
if [ -n "$ISSUE_URL" ] && [ "$ISSUE_URL" != "null" ]; then
  ISSUE_REPO=$(resolve_repo_from_url "$ISSUE_URL")
else
  ISSUE_REPO="$REPO"
fi
```

Use `$ISSUE_REPO` (resolved per-issue) instead of `$REPO` for all per-issue `gh` commands. Keep config `$REPO` for creating new issues (`/kanban-gh add`, `/kanban-gh-explore`).

---

## GitHub Project Field Definitions

### Status (Single select)

| Value | Description |
|-------|-------------|
| `Backlog` | Parked for later |
| `Todo` | Not yet started |
| `Plan` | Planning in progress |
| `Plan Review` | Plan awaiting review |
| `Implement` | Implementation in progress |
| `Impl Review` | Implementation awaiting review |
| `Test` | Testing in progress |
| `Done` | Complete |

### Priority (Single select)

| Value | Description |
|-------|-------------|
| `low` | Low priority |
| `medium` | Medium priority |
| `high` | High priority |

### Level (Single select)

| Value | Description |
|-------|-------------|
| `L1` | Level 1 (smallest / simplest) |
| `L2` | Level 2 (medium) |
| `L3` | Level 3 (largest / most complex) |

### Tags (Text)

Comma-separated list of free-form tags, e.g. `backend,auth,breaking-change`.

---

## Label Naming Convention (Repo Mode)

In repo mode, project field values are represented as GitHub labels on each issue. Labels use a `prefix:value` naming convention to avoid collisions with user-defined labels.

### Label Format

| Field | Label prefix | Examples |
|-------|-------------|----------|
| Status | `status:` | `status:backlog`, `status:todo`, `status:plan`, `status:plan-review`, `status:implement`, `status:impl-review`, `status:test`, `status:done` |
| Priority | `priority:` | `priority:low`, `priority:medium`, `priority:high` |
| Level | `level:` | `level:L1`, `level:L2`, `level:L3` |

Label names are lowercase. Spaces in status values become hyphens (e.g., `Plan Review` becomes `status:plan-review`). Tags remain as free-form text in the issue body — they are NOT mapped to labels.

### Display Name Mapping

Bidirectional mapping between label values and display names:

| Label value | Display name |
|-------------|-------------|
| `backlog` | `Backlog` |
| `todo` | `Todo` |
| `plan` | `Plan` |
| `plan-review` | `Plan Review` |
| `implement` | `Implement` |
| `impl-review` | `Impl Review` |
| `test` | `Test` |
| `done` | `Done` |

### Label Colors

| Label | Color hex | Description |
|-------|-----------|-------------|
| `status:backlog` | `c5def5` | Parked for later |
| `status:todo` | `0e8a16` | Not yet started |
| `status:plan` | `0075ca` | Planning in progress |
| `status:plan-review` | `7057ff` | Plan awaiting review |
| `status:implement` | `e99695` | Implementation in progress |
| `status:impl-review` | `d876e3` | Implementation awaiting review |
| `status:test` | `fbca04` | Testing in progress |
| `status:done` | `ededed` | Complete |
| `priority:low` | `0e8a16` | Low priority |
| `priority:medium` | `fbca04` | Medium priority |
| `priority:high` | `d73a4a` | High priority |
| `level:L1` | `0e8a16` | Level 1 (smallest) |
| `level:L2` | `fbca04` | Level 2 (medium) |
| `level:L3` | `d73a4a` | Level 3 (largest) |

### Parsing Labels from an Issue

```bash
# Extract field values from issue labels
STATUS=$(echo "$LABELS" | jq -r '[.[].name | select(startswith("status:"))] | first // ""' | sed 's/^status://')
PRIORITY=$(echo "$LABELS" | jq -r '[.[].name | select(startswith("priority:"))] | first // ""' | sed 's/^priority://')
LEVEL=$(echo "$LABELS" | jq -r '[.[].name | select(startswith("level:"))] | first // ""' | sed 's/^level://')
```

### Converting Label Value to Display Name

```bash
label_to_display() {
  case "$1" in
    backlog) echo "Backlog" ;;
    todo) echo "Todo" ;;
    plan) echo "Plan" ;;
    plan-review) echo "Plan Review" ;;
    implement) echo "Implement" ;;
    impl-review) echo "Impl Review" ;;
    test) echo "Test" ;;
    done) echo "Done" ;;
    *) echo "$1" ;;
  esac
}

display_to_label() {
  case "$1" in
    Backlog) echo "backlog" ;;
    Todo) echo "todo" ;;
    Plan) echo "plan" ;;
    "Plan Review") echo "plan-review" ;;
    Implement) echo "implement" ;;
    "Impl Review") echo "impl-review" ;;
    Test) echo "test" ;;
    Done) echo "done" ;;
    *) echo "$1" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' ;;
  esac
}
```

### Edge Cases

- **No status label**: Issue is treated as untracked. `list` shows it in a separate "(untracked)" section. `move` and `run` refuse to operate.
- **Multiple status labels**: Use the first one found and warn the user. Consider stripping duplicates automatically.
- **Closed issues with `status:done`**: Include in `list` and `stats` by fetching closed issues separately.

---

## Agent Nicknames

| Nickname | Role | Model | Writes to (issue comment section) |
|----------|------|-------|------------------------------------|
| `Planner` | Plan Agent | `opus` | Plan + Decision Log + Done When |
| `Critic` | Plan Review | `sonnet` | Review verdict + scores |
| `Builder` | Worker | `opus` | Implementation notes |
| `Shield` | TDD Tester | `sonnet` | Test notes (separate comment from Builder's) |
| `Inspector` | Code Review | `sonnet` | Review verdict + scores |
| `Ranger` | Test Runner | `sonnet` | Test results |
| `Refiner` | Requirements Refinement | `sonnet` | Refined description (rewrites issue body) |

---

## Signature Header Rule

Every agent comment must start with a signature header on its own line:

```
> **[Nickname]** `[model]` · [ISO 8601 timestamp]Z
```

Examples:

```
> **Planner** `opus` · 2026-03-21T14:32:00Z

> **Critic** `sonnet` · 2026-03-21T15:10:45Z
```

The timestamp is UTC and always ends with `Z`. The `·` separator is U+00B7 (middle dot).
