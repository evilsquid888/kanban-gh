---
name: kanban-gh-init
description: "Connect to a GitHub Project, create pipeline fields, and write local config. Usage: /kanban-gh-init [project-url-or-number]"
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md for field names and config format. Read ~/.claude/skills/shared/graphql.md for GraphQL operations.

# kanban-gh-init

Initialize a local kanban-gh workspace by connecting to a GitHub Project, creating required pipeline fields, and writing `.claude/kanban-gh.json`.

---

## Step 1 — Auth check

Run:

```bash
gh auth status
```

If this fails, tell the user:

```
gh auth login --scopes project,repo
```

Then exit. Do not proceed until auth succeeds.

---

## Step 2 — Parse argument

The skill accepts an optional argument: a project URL or a bare project number.

### Case A — URL provided

If the argument matches `https://github.com/users/<OWNER>/projects/<N>` or `https://github.com/orgs/<OWNER>/projects/<N>`:

```bash
# Extract from URL
OWNER=$(echo "$ARG" | sed -E 's|https://github.com/(users|orgs)/([^/]+)/projects/[0-9]+|\2|')
PROJECT_NUMBER=$(echo "$ARG" | sed -E 's|.*/projects/([0-9]+)|\1|')
```

### Case B — Number provided

If the argument is a plain integer:

```bash
PROJECT_NUMBER="$ARG"
OWNER=$(gh repo view --json owner --jq '.owner.login')
```

### Case C — Nothing provided

Run:

```bash
gh project list --owner @me --format json
```

Parse the output and present the list to the user. Use AskUserQuestion:

```
Which project do you want to connect to?
List the projects as numbered options, e.g.:
  1. My Project (https://github.com/users/alice/projects/3)
  2. Another Project (https://github.com/users/alice/projects/7)
Enter a number or paste a project URL:
```

Once the user answers, extract `OWNER` and `PROJECT_NUMBER` from their selection.

---

## Step 3 — Determine ownerType

```bash
OWNER_TYPE=$(gh api /repos/$OWNER/$REPO --jq '.owner.type' | tr '[:upper:]' '[:lower:]')
```

Where `$REPO` is the current repository name (from `gh repo view --json name --jq '.name'`).

If the API call fails (e.g., the owner is a user with no matching repo), fall back to:

```bash
OWNER_TYPE=$(gh api /users/$OWNER --jq 'if .type == "Organization" then "organization" else "user" end')
```

`OWNER_TYPE` must be either `"user"` or `"organization"`.

---

## Step 4 — Resolve project node ID

Use the `getProject` operation from `shared/graphql.md`. Choose the user or org variant based on `OWNER_TYPE`.

**User-owned:**

```bash
RESPONSE=$(gh api graphql -f query='
query {
  user(login: "'"$OWNER"'") {
    projectV2(number: '"$PROJECT_NUMBER"') {
      id
      title
    }
  }
}')
PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.user.projectV2.id')
PROJECT_TITLE=$(echo "$RESPONSE" | jq -r '.data.user.projectV2.title')
```

**Org-owned:**

```bash
RESPONSE=$(gh api graphql -f query='
query {
  organization(login: "'"$OWNER"'") {
    projectV2(number: '"$PROJECT_NUMBER"') {
      id
      title
    }
  }
}')
PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.organization.projectV2.id')
PROJECT_TITLE=$(echo "$RESPONSE" | jq -r '.data.organization.projectV2.title')
```

If `PROJECT_ID` is `null` or empty, exit with:

```
Error: Could not resolve project #<PROJECT_NUMBER> for owner '<OWNER>'.
Check the URL or number and try again.
```

---

## Step 5 — Check existing fields

Use the `getProjectFields` operation from `shared/graphql.md`:

```bash
FIELDS_RESPONSE=$(gh api graphql -f query='
query {
  node(id: "'"$PROJECT_ID"'") {
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
}')
FIELDS=$(echo "$FIELDS_RESPONSE" | jq '.data.node.fields.nodes')
```

For each required field, check if it exists by name:

```bash
STATUS_FIELD=$(echo "$FIELDS" | jq '.[] | select(.name == "Status")')
PRIORITY_FIELD=$(echo "$FIELDS" | jq '.[] | select(.name == "Priority")')
LEVEL_FIELD=$(echo "$FIELDS" | jq '.[] | select(.name == "Level")')
TAGS_FIELD=$(echo "$FIELDS" | jq '.[] | select(.name == "Tags")')
```

---

## Step 6 — Create missing fields / handle conflicts

For each required field below, apply the conflict resolution logic, then create if missing.

### Conflict resolution logic

For each field:

1. **Field does not exist** → create it (see creation commands below).
2. **Field exists with correct type and all required options present** → skip, no action needed.
3. **Field exists with correct type but missing required options** → update the field's options using `updateField` from `shared/graphql.md` (section 3b). This is the **common case** for `Status` — every new GitHub Project has a default `Status` field with `Todo`, `In Progress`, `Done`. We replace these with our 7-column pipeline options.
4. **Field exists with wrong type** (e.g. `TEXT` instead of `SINGLE_SELECT`) → warn and exit:

```
⚠️ Field '<NAME>' exists but has wrong type. Cannot auto-fix.
Current type: <show current dataType>
Required type: SINGLE_SELECT
Fix manually in GitHub Projects settings, then re-run /kanban-gh-init.
```

To check option presence:

```bash
# Example: check that "Plan" exists in Status options
HAS_PLAN=$(echo "$STATUS_FIELD" | jq -r '.options[]? | select(.name == "Plan") | .name')
```

To update existing field options (see `shared/graphql.md` section 3b `updateField`):

```bash
FIELD_ID=$(echo "$STATUS_FIELD" | jq -r '.id')
gh api graphql -f query='
mutation($fieldId: ID!) {
  updateProjectV2Field(input: {
    fieldId: $fieldId
    singleSelectOptions: [
      {name: "Todo", color: GREEN, description: "Not yet started"},
      {name: "Plan", color: BLUE, description: "Planning in progress"},
      ...
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { options { id name } } }
  }
}' -f fieldId="$FIELD_ID"
```

### Required fields

#### Status (SINGLE_SELECT)

Required options (in order): `Todo`, `Plan`, `Plan Review`, `Implement`, `Impl Review`, `Test`, `Done`

If field does not exist, create it. If it exists as SINGLE_SELECT but with wrong options, update it using `updateField` (section 3b of `shared/graphql.md`).

```bash
# Create new Status field (if it doesn't exist at all)
gh api graphql -f query='
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: SINGLE_SELECT
    name: "Status"
    singleSelectOptions: [
      {name: "Todo",        color: GREEN,  description: "Not yet started"},
      {name: "Plan",        color: BLUE,   description: "Planning in progress"},
      {name: "Plan Review", color: PURPLE, description: "Plan awaiting review"},
      {name: "Implement",   color: ORANGE, description: "Implementation in progress"},
      {name: "Impl Review", color: PINK,   description: "Implementation awaiting review"},
      {name: "Test",        color: YELLOW, description: "Testing in progress"},
      {name: "Done",        color: GRAY,   description: "Complete"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { id options { id name } } }
  }
}' -f projectId="$PROJECT_ID"

# OR update existing Status field options (common case — new projects have default Status)
FIELD_ID=$(echo "$STATUS_FIELD" | jq -r '.id')
gh api graphql -f query='
mutation($fieldId: ID!) {
  updateProjectV2Field(input: {
    fieldId: $fieldId
    singleSelectOptions: [
      {name: "Todo",        color: GREEN,  description: "Not yet started"},
      {name: "Plan",        color: BLUE,   description: "Planning in progress"},
      {name: "Plan Review", color: PURPLE, description: "Plan awaiting review"},
      {name: "Implement",   color: ORANGE, description: "Implementation in progress"},
      {name: "Impl Review", color: PINK,   description: "Implementation awaiting review"},
      {name: "Test",        color: YELLOW, description: "Testing in progress"},
      {name: "Done",        color: GRAY,   description: "Complete"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { options { id name } } }
  }
}' -f fieldId="$FIELD_ID"
```

#### Priority (SINGLE_SELECT)

Required options: `low`, `medium`, `high`

```bash
gh api graphql -f query='
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: SINGLE_SELECT
    name: "Priority"
    singleSelectOptions: [
      {name: "low",    color: GREEN,  description: "Low priority"},
      {name: "medium", color: YELLOW, description: "Medium priority"},
      {name: "high",   color: RED,    description: "High priority"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { id options { id name } } }
  }
}' -f projectId="$PROJECT_ID"
```

#### Level (SINGLE_SELECT)

Required options: `L1`, `L2`, `L3`

```bash
gh api graphql -f query='
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: SINGLE_SELECT
    name: "Level"
    singleSelectOptions: [
      {name: "L1", color: GREEN,  description: "Quick"},
      {name: "L2", color: YELLOW, description: "Standard"},
      {name: "L3", color: RED,    description: "Full"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { id options { id name } } }
  }
}' -f projectId="$PROJECT_ID"
```

#### Tags (TEXT)

```bash
gh api graphql -f query='
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: TEXT
    name: "Tags"
  }) {
    projectV2Field { ... on ProjectV2Field { id } }
  }
}' -f projectId="$PROJECT_ID"
```

After processing all four fields, print a summary:

```
Fields:
  Status   — ✓ created / ✓ already exists
  Priority — ✓ created / ✓ already exists
  Level    — ✓ created / ✓ already exists
  Tags     — ✓ created / ✓ already exists
```

---

## Step 7 — Check for existing config

Before writing the config, check if `.claude/kanban-gh.json` already exists:

```bash
if [ -f ".claude/kanban-gh.json" ]; then
  CURRENT_CONFIG=$(cat .claude/kanban-gh.json)
  echo "Existing config found:"
  echo "$CURRENT_CONFIG" | jq .
fi
```

If it exists, use AskUserQuestion:

```
.claude/kanban-gh.json already exists (shown above).
Overwrite with new settings, or keep the current config?
  1. Overwrite
  2. Keep as-is
```

If the user selects **Keep as-is**: exit with:

```
Keeping existing config. No changes made.
Run /kanban-gh-init again to reconfigure.
```

---

## Step 8 — Ask target repo

Get the default repo:

```bash
DEFAULT_REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

Use AskUserQuestion:

```
Which repo should issues be created in?
Default: <DEFAULT_REPO>
Press Enter to accept the default, or type owner/repo:
```

If the user presses Enter or provides an empty answer, use `DEFAULT_REPO`. Otherwise use their input as `REPO`.

---

## Step 9 — Write config

Create `.claude/` directory if it does not exist:

```bash
mkdir -p .claude
```

Write `.claude/kanban-gh.json` using the Write tool:

```json
{
  "project": "<PROJECT_NUMBER>",
  "owner": "<OWNER>",
  "ownerType": "<user|organization>",
  "repo": "<OWNER/REPO>",
  "retries": 2
}
```

All values are strings except `retries` which is a number.

---

## Step 10 — Output confirmation

Print:

```
✅ kanban-gh initialized.

  Project: github.com/users/<OWNER>/projects/<PROJECT_NUMBER>
  Repo:    <OWNER>/<REPO>
  Config:  .claude/kanban-gh.json

Add tasks with /kanban-gh add <title>
```

For org-owned projects, use `github.com/orgs/<OWNER>/projects/<PROJECT_NUMBER>` in the Project line.
