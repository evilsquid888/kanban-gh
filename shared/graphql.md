# Shared GraphQL Templates & Label Operations

GraphQL query and mutation templates for GitHub Projects v2, and `gh` CLI label commands for repo mode. All `$VARIABLES` are substituted at runtime by skill scripts.

---

## Node ID Resolution

GitHub Projects v2 requires node IDs (opaque base64 strings) rather than human-readable numbers. These are fetched at runtime and are not cached between invocations.

| Variable | Source |
|----------|--------|
| `$PROJECT_ID` | Returned by `getProject` |
| `$FIELD_ID` | Returned by `getProjectFields` |
| `$OPTION_ID` | Returned by `getProjectFields` (nested inside each field's `options`) |
| `$ITEM_ID` | Returned by `getProjectItems` |
| `$ISSUE_NODE_ID` | `gh api /repos/{owner}/{repo}/issues/{number} --jq .node_id` — use `$ISSUE_REPO` (resolved from item URL) instead of config `$REPO` for per-issue lookups |
| `$ISSUE_REPO` | Resolved per-issue from item URL via `resolve_repo_from_url` (see `shared/schema.md`); falls back to config `$REPO` |

Typical resolution order for a write operation:

1. Call `getProject` → get `$PROJECT_ID`
2. Call `getProjectFields` → get `$FIELD_ID` and `$OPTION_ID`
3. Call `getProjectItems` (if updating an existing item) → get `$ITEM_ID`
4. Call the mutation with all resolved IDs

---

## Operations

### 1. `getProject`

Resolve the project node ID from owner login and project number. Use the `user` variant when `ownerType` is `user`, and the `organization` variant when `ownerType` is `organization`.

**User-owned project:**

```graphql
query {
  user(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}
```

**Organization-owned project:**

```graphql
query {
  organization(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}
```

**Usage:**

```bash
# User-owned
gh api graphql -f query='
query {
  user(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}' -f owner="$OWNER" -F number=$PROJECT_NUMBER

# Org-owned
gh api graphql -f query='
query {
  organization(login: "$OWNER") {
    projectV2(number: $PROJECT_NUMBER) {
      id
      title
    }
  }
}' -f owner="$OWNER" -F number=$PROJECT_NUMBER
```

Extract the node ID:

```bash
PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.user.projectV2.id')
# or for org:
PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.organization.projectV2.id')
```

---

### 2. `getProjectFields`

List all fields on a project with their types and, for single-select fields, their options.

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

**Usage:**

```bash
gh api graphql -f query='
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
}' -f id="$PROJECT_ID"
```

Extract a field ID and option ID by name:

```bash
FIELDS=$(echo "$RESPONSE" | jq '.data.node.fields.nodes')
FIELD_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .id')
OPTION_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .options[] | select(.name == "Implement") | .id')
```

---

### 3. `createField`

Create a new field on a project. Two variants are provided: single-select (with options) and plain text.

**Important:** Single-select options require `name`, `color`, and `description` (all three are required by the GitHub API).

Valid colors: `GRAY`, `BLUE`, `GREEN`, `YELLOW`, `ORANGE`, `RED`, `PINK`, `PURPLE`.

**Single-select field:**

```graphql
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: SINGLE_SELECT
    name: "$FIELD_NAME"
    singleSelectOptions: [
      {name: "$OPTION_1", color: BLUE, description: "$DESC_1"},
      {name: "$OPTION_2", color: GREEN, description: "$DESC_2"},
      {name: "$OPTION_3", color: RED, description: "$DESC_3"}
    ]
  }) {
    projectV2Field {
      ... on ProjectV2SingleSelectField {
        id
        options { id name }
      }
    }
  }
}
```

**Text field:**

```graphql
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: TEXT
    name: "$FIELD_NAME"
  }) {
    projectV2Field { ... on ProjectV2Field { id } }
  }
}
```

**Usage:**

```bash
# Single-select field (example: Priority)
gh api graphql -f query='
mutation($projectId: ID!) {
  createProjectV2Field(input: {
    projectId: $projectId
    dataType: SINGLE_SELECT
    name: "Priority"
    singleSelectOptions: [
      {name: "low", color: GREEN, description: "Low priority"},
      {name: "medium", color: YELLOW, description: "Medium priority"},
      {name: "high", color: RED, description: "High priority"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { id options { id name } } }
  }
}' -f projectId="$PROJECT_ID"

# Text field (example: Tags)
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

---

### 3b. `updateField`

Update an existing single-select field's options. Use this when a field already exists but has the wrong options (e.g., the default `Status` field on a new GitHub Project has `Todo`, `In Progress`, `Done` — we need to replace these with our 8-column pipeline statuses).

**Important:** `updateProjectV2Field` takes `fieldId` only — NOT `projectId`.

```graphql
mutation($fieldId: ID!) {
  updateProjectV2Field(input: {
    fieldId: $fieldId
    singleSelectOptions: [
      {name: "$OPTION_1", color: GREEN, description: "$DESC_1"},
      {name: "$OPTION_2", color: BLUE, description: "$DESC_2"}
    ]
  }) {
    projectV2Field {
      ... on ProjectV2SingleSelectField {
        options { id name }
      }
    }
  }
}
```

**Usage (replace Status options with pipeline statuses):**

```bash
gh api graphql -f query='
mutation($fieldId: ID!) {
  updateProjectV2Field(input: {
    fieldId: $fieldId
    singleSelectOptions: [
      {name: "Backlog", color: BLUE, description: "Parked for later"},
      {name: "Todo", color: GREEN, description: "Not yet started"},
      {name: "Plan", color: BLUE, description: "Planning in progress"},
      {name: "Plan Review", color: PURPLE, description: "Plan awaiting review"},
      {name: "Implement", color: ORANGE, description: "Implementation in progress"},
      {name: "Impl Review", color: PINK, description: "Implementation awaiting review"},
      {name: "Test", color: YELLOW, description: "Testing in progress"},
      {name: "Done", color: GRAY, description: "Complete"}
    ]
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { options { id name } } }
  }
}' -f fieldId="$STATUS_FIELD_ID"
```

---

### 4. `getProjectItems`

Fetch all items in a project along with their field values.

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

**Usage:**

```bash
gh api graphql -f query='
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
}' -f id="$PROJECT_ID"
```

Extract a specific item's ID by issue number:

```bash
ITEM_ID=$(echo "$RESPONSE" | jq -r --argjson num "$ISSUE_NUMBER" \
  '.data.node.items.nodes[] | select(.content.number == $num) | .id')
```

---

### 4b. `getProjectItem`

Fetch a single project item by issue number. This is a convenience wrapper — it uses `getProjectItems` and filters by issue number.

```bash
# Fetch all items and filter to a specific issue number
ITEM=$(gh api graphql -f query='
  query($projectId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        items(first: 100) {
          nodes {
            id
            content {
              ... on Issue { number title body url }
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
' -f projectId="$PROJECT_ID" \
  --jq ".data.node.items.nodes[] | select(.content.number == $ISSUE_NUMBER)")
```

Extract fields from the result:

```bash
ITEM_ID=$(echo "$ITEM" | jq -r '.id')
STATUS=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Status") | .name')
LEVEL=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Level") | .name')
```

---

### 5. `updateFieldValue`

Set a single-select field value on a project item.

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

**Usage:**

```bash
gh api graphql -f query='
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "$PROJECT_ID"
    itemId: "$ITEM_ID"
    fieldId: "$FIELD_ID"
    value: { singleSelectOptionId: "$OPTION_ID" }
  }) {
    projectV2Item { id }
  }
}' \
  -f projectId="$PROJECT_ID" \
  -f itemId="$ITEM_ID" \
  -f fieldId="$FIELD_ID" \
  -f optionId="$OPTION_ID"
```

---

### 6. `addIssueToProject`

Add an existing GitHub issue to a project by its node ID.

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

**Usage:**

```bash
# First, resolve the issue node ID
ISSUE_NODE_ID=$(gh api /repos/$REPO/issues/$ISSUE_NUMBER --jq .node_id)

# Then add to project
gh api graphql -f query='
mutation {
  addProjectV2ItemById(input: {
    projectId: "$PROJECT_ID"
    contentId: "$ISSUE_NODE_ID"
  }) {
    item { id }
  }
}' \
  -f projectId="$PROJECT_ID" \
  -f contentId="$ISSUE_NODE_ID"
```

The returned `item.id` is the new `$ITEM_ID` for subsequent field updates.

---

### 7. `removeItemFromProject`

Remove an item from a project by its item node ID.

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

**Usage:**

```bash
gh api graphql -f query='
mutation {
  deleteProjectV2Item(input: {
    projectId: "$PROJECT_ID"
    itemId: "$ITEM_ID"
  }) {
    deletedItemId
  }
}' \
  -f projectId="$PROJECT_ID" \
  -f itemId="$ITEM_ID"
```

---

## Label Operations (Repo Mode)

When `mode` is `"repo"`, status/priority/level tracking uses GitHub labels instead of Project fields. All operations use the `gh` CLI — no GraphQL needed.

**Reference:** See `shared/schema.md` for the full label naming convention, color table, and display name mapping.

---

### 8. Create Labels (Init)

Create all required labels on a repo. Uses `--force` to update color/description if the label already exists (idempotent).

```bash
# Status labels
gh label create "status:todo"         --color "0e8a16" --description "Not yet started"                    --repo "$REPO" --force
gh label create "status:plan"         --color "0075ca" --description "Planning in progress"                --repo "$REPO" --force
gh label create "status:plan-review"  --color "7057ff" --description "Plan awaiting review"                --repo "$REPO" --force
gh label create "status:implement"    --color "e99695" --description "Implementation in progress"          --repo "$REPO" --force
gh label create "status:impl-review"  --color "d876e3" --description "Implementation awaiting review"      --repo "$REPO" --force
gh label create "status:test"         --color "fbca04" --description "Testing in progress"                 --repo "$REPO" --force
gh label create "status:done"         --color "ededed" --description "Complete"                             --repo "$REPO" --force

# Priority labels
gh label create "priority:low"    --color "0e8a16" --description "Low priority"    --repo "$REPO" --force
gh label create "priority:medium" --color "fbca04" --description "Medium priority" --repo "$REPO" --force
gh label create "priority:high"   --color "d73a4a" --description "High priority"   --repo "$REPO" --force

# Level labels
gh label create "level:L1" --color "0e8a16" --description "Level 1 — Quick"    --repo "$REPO" --force
gh label create "level:L2" --color "fbca04" --description "Level 2 — Standard" --repo "$REPO" --force
gh label create "level:L3" --color "d73a4a" --description "Level 3 — Full"     --repo "$REPO" --force
```

---

### 9. Add Labels to an Issue

```bash
# Set initial labels when adding a new task
gh issue edit $NUMBER --repo "$REPO" --add-label "status:todo,priority:$PRIORITY,level:$LEVEL"
```

---

### 10. Swap a Status Label (Move)

Remove the old status label and add the new one in a single edit:

```bash
gh issue edit $NUMBER --repo "$REPO" --remove-label "status:$OLD_STATUS" --add-label "status:$NEW_STATUS"
```

Where `$OLD_STATUS` and `$NEW_STATUS` are the label-format values (e.g., `plan-review`, `implement`).

---

### 11. Swap a Field Label (Edit)

For priority or level changes:

```bash
# Change priority
gh issue edit $NUMBER --repo "$REPO" --remove-label "priority:$OLD" --add-label "priority:$NEW"

# Change level
gh issue edit $NUMBER --repo "$REPO" --remove-label "level:$OLD" --add-label "level:$NEW"
```

---

### 12. Remove All Kanban Labels

Strip all `status:`, `priority:`, and `level:` labels from an issue:

```bash
# Get current kanban labels
KANBAN_LABELS=$(gh issue view $NUMBER --repo "$REPO" --json labels --jq '[.labels[].name | select(startswith("status:") or startswith("priority:") or startswith("level:"))] | join(",")')

# Remove them
if [ -n "$KANBAN_LABELS" ]; then
  gh issue edit $NUMBER --repo "$REPO" --remove-label "$KANBAN_LABELS"
fi
```

---

### 13. Fetch All Tracked Issues (Repo Mode)

Fetch all open issues and closed done issues, then parse labels client-side:

```bash
# All open issues (includes tracked and untracked)
ALL_OPEN=$(gh issue list --repo "$REPO" --state open --json number,title,body,url,labels --limit 200)

# Closed issues with status:done
DONE_CLOSED=$(gh issue list --repo "$REPO" --state closed --label "status:done" --json number,title,body,url,labels --limit 100)

# Combine and deduplicate
ALL_ISSUES=$(echo "$ALL_OPEN" "$DONE_CLOSED" | jq -s 'add | unique_by(.number)')
```

Filter to only tracked issues (those with at least one `status:` label):

```bash
TRACKED=$(echo "$ALL_ISSUES" | jq '[.[] | select(.labels[].name | startswith("status:"))]')
```

Parse fields from a single issue:

```bash
parse_item_repo() {
  local ITEM="$1"
  NUMBER=$(echo "$ITEM" | jq -r '.number')
  TITLE=$(echo "$ITEM" | jq -r '.title')
  STATUS=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("status:"))] | first // ""' | sed 's/^status://')
  PRIORITY=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("priority:"))] | first // ""' | sed 's/^priority://')
  LEVEL=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("level:"))] | first // ""' | sed 's/^level://')
}
```

---

### 14. Fetch Todo Issues (Repo Mode — for `--loop`)

```bash
TODO_ISSUES=$(gh issue list --repo "$REPO" --state open --label "status:todo" --json number,title,labels --limit 100 | jq 'sort_by(.number)')
```

---

### 15. Search Issues by Title (Repo Mode — for ID Resolution)

```bash
MATCHES=$(gh issue list --repo "$REPO" --state open --search "$ARG" --json number,title,labels --limit 50)
```
