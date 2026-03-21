# Shared Schema Reference

This is the canonical reference for field names, status values, agent identities, and config structure used across all kanban-gh skills.

---

## Config File

Location: `.claude/kanban-gh.json`

```json
{
  "project": "My Project Name",
  "owner": "my-org-or-user",
  "ownerType": "organization",
  "repo": "my-repo",
  "retries": 2
}
```

### Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `project` | string | GitHub Projects v2 project title | required |
| `owner` | string | GitHub org or user login | required |
| `ownerType` | `user` \| `organization` | Owner account type | required |
| `repo` | string | Repository name (without owner prefix) | required |
| `retries` | number | Number of retry attempts for API calls | `2` |

### Config Loading Snippet

```bash
CONFIG=$(cat .claude/kanban-gh.json 2>/dev/null)
PROJECT=$(echo "$CONFIG" | jq -r '.project')
OWNER=$(echo "$CONFIG" | jq -r '.owner')
OWNER_TYPE=$(echo "$CONFIG" | jq -r '.ownerType')
REPO=$(echo "$CONFIG" | jq -r '.repo')
RETRIES=$(echo "$CONFIG" | jq -r '.retries // 2')
```

---

## GitHub Project Field Definitions

### Status (Single select)

| Value | Description |
|-------|-------------|
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

## Agent Nicknames

| Nickname | Role | Model | Writes to (issue comment section) |
|----------|------|-------|------------------------------------|
| `Planner` | Plan Agent | `opus` | Plan + Decision Log + Done When |
| `Critic` | Plan Review | `sonnet` | Review verdict + scores |
| `Builder` | Worker | `opus` | Implementation notes |
| `Shield` | TDD Tester | `sonnet` | Test notes |
| `Inspector` | Code Review | `sonnet` | Review verdict + scores |
| `Ranger` | Test Runner | `sonnet` | Test results |

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
