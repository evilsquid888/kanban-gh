---
name: kanban-gh
description: "Manage tasks in GitHub Projects. Supports add, list, move, edit, remove, stats, context. Uses GitHub Projects v2 as the data store. Run /kanban-gh-init first."
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md for field names, status values, and config format. Read ~/.claude/skills/shared/graphql.md for GraphQL operations. Read ~/.claude/skills/shared/pipeline.md for valid status transitions.

# kanban-gh

CRUD commands for managing tasks in a GitHub Projects v2 board. Requires `/kanban-gh-init` to have been run first.

---

## Setup

Run `/kanban-gh-init` before using any command. It creates `.claude/kanban-gh.json` with the project connection details.

Add `.claude/kanban-gh.json` to `.gitignore` to avoid committing credentials:

```bash
echo '.claude/kanban-gh.json' >> .gitignore
```

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

## ID Resolution

Many commands accept `<ID|name>` as an argument. Resolve it as follows:

### Issue number

If the argument is a plain integer or starts with `#`, strip the `#` prefix and use it directly as the issue number:

```bash
NUMBER=$(echo "$ARG" | sed 's/^#//')
```

### Partial title match

If the argument is not numeric, treat it as a case-insensitive substring to match against item titles. Fetch all items via `getProjectItems` (see GraphQL section), then filter:

```bash
MATCHES=$(echo "$ITEMS" | jq --arg q "$ARG" '
  [.[] | select(.content.title | ascii_downcase | contains($q | ascii_downcase))]
')
COUNT=$(echo "$MATCHES" | jq 'length')
```

- **Zero matches** → exit with: `Error: No items found matching "<ARG>".`
- **One match** → use it automatically.
- **Multiple matches** → list all with issue numbers:

  ```
  Multiple items match "<ARG>":
    #3  Add authentication
    #7  Add auth middleware
    #12 Refactor auth module
  ```

  Then use AskUserQuestion to ask the user which one to use. In `--auto` mode, do not prompt — exit with:

  ```
  Error: Multiple items match "<ARG>". Use an issue number to be specific.
  ```

---

## Shared: Fetch All Items

Most commands need to fetch all project items. Resolve `$PROJECT_ID` first, then call `getProjectItems`:

```bash
# Resolve PROJECT_ID
if [ "$OWNER_TYPE" = "organization" ]; then
  RESPONSE=$(gh api graphql -f query='
  query {
    organization(login: "'"$OWNER"'") {
      projectV2(number: '"$PROJECT"') { id }
    }
  }')
  PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.organization.projectV2.id')
else
  RESPONSE=$(gh api graphql -f query='
  query {
    user(login: "'"$OWNER"'") {
      projectV2(number: '"$PROJECT"') { id }
    }
  }')
  PROJECT_ID=$(echo "$RESPONSE" | jq -r '.data.user.projectV2.id')
fi

# Fetch all items
ITEMS_RESPONSE=$(gh api graphql -f query='
query {
  node(id: "'"$PROJECT_ID"'") {
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
}')
ITEMS=$(echo "$ITEMS_RESPONSE" | jq '.data.node.items.nodes')
```

---

## Shared: Resolve Field and Option IDs

Before setting a field value, fetch field metadata:

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

# Example: get Status field ID and the option ID for "Todo"
STATUS_FIELD_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .id')
TODO_OPTION_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .options[] | select(.name == "Todo") | .id')
```

---

## Commands

---

### `/kanban-gh list`

Fetch all items and render a markdown table sorted by status column order.

**Status column order:** Todo, Plan, Plan Review, Implement, Impl Review, Test, Done

**Steps:**

1. Load config and fetch all items (see Shared sections above).
2. Parse each item's field values:

```bash
# Extract fields from a single item node
parse_item() {
  local ITEM="$1"
  NUMBER=$(echo "$ITEM" | jq -r '.content.number // ""')
  TITLE=$(echo "$ITEM" | jq -r '.content.title // ""')
  STATUS=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Status") | .name // ""' 2>/dev/null | head -1)
  PRIORITY=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Priority") | .name // ""' 2>/dev/null | head -1)
  LEVEL=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Level") | .name // ""' 2>/dev/null | head -1)
}
```

3. Sort items by status order and render:

```
| # | Status | Priority | Level | Title |
|---|--------|----------|-------|-------|
| 8 | Todo | high | L2 | Add logging |
| 3 | Implement | medium | L3 | Add auth |
| 1 | Done | low | L1 | Setup project |
```

If there are no items, output: `No items in project.`

---

### `/kanban-gh add <title>`

Create a new GitHub issue and add it to the project.

**Steps:**

1. Load config.
2. Use AskUserQuestion to collect:

   ```
   Adding task: "<TITLE>"

   Priority? (low / medium / high) [medium]:
   Level? (L1 / L2 / L3) [L2]:
   Description (optional, press Enter to skip):
   Tags (optional, comma-separated, e.g. backend,auth):
   ```

   Use the bracketed values as defaults if the user presses Enter without typing.

3. Create the GitHub issue:

```bash
ISSUE_OUTPUT=$(gh issue create \
  --repo "$REPO" \
  --title "$TITLE" \
  --body "$DESCRIPTION")
# Extract issue number from output URL: https://github.com/owner/repo/issues/42
NUMBER=$(echo "$ISSUE_OUTPUT" | grep -oP '(?<=/issues/)\d+')
```

4. Resolve the issue node ID:

```bash
ISSUE_NODE_ID=$(gh api /repos/$REPO/issues/$NUMBER --jq .node_id)
```

5. Resolve `$PROJECT_ID` (see Shared section).

6. Add the issue to the project:

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

7. Resolve field and option IDs (see Shared section), then set Status=Todo, Priority, and Level:

```bash
# Set Status = Todo
gh api graphql -f query='
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "'"$PROJECT_ID"'"
    itemId: "'"$ITEM_ID"'"
    fieldId: "'"$STATUS_FIELD_ID"'"
    value: { singleSelectOptionId: "'"$TODO_OPTION_ID"'" }
  }) {
    projectV2Item { id }
  }
}'

# Set Priority (resolve PRIORITY_FIELD_ID and PRIORITY_OPTION_ID similarly)
# Set Level (resolve LEVEL_FIELD_ID and LEVEL_OPTION_ID similarly)
```

8. If tags were provided, set the Tags text field:

```bash
TAGS_FIELD_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Tags") | .id')
gh api graphql -f query='
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "'"$PROJECT_ID"'"
    itemId: "'"$ITEM_ID"'"
    fieldId: "'"$TAGS_FIELD_ID"'"
    value: { text: "'"$TAGS"'" }
  }) {
    projectV2Item { id }
  }
}'
```

9. Output:

```
✅ Created #<NUMBER>: <TITLE> (Status: Todo, Priority: <PRIORITY>, Level: <LEVEL>)
```

---

### `/kanban-gh move <ID|name> <status>`

Move an item to a new status, validating the transition against pipeline rules.

**Steps:**

1. Load config, fetch all items.
2. Resolve `<ID|name>` to an issue number and item node ID (see ID Resolution above).
3. Get the item's current Status and Level field values.
4. Validate the transition using the pipeline rules from `shared/pipeline.md`:

   **L1** (Level = L1): `Todo → Implement → Done`

   **L2** (Level = L2):
   - Forward: `Todo → Plan → Implement → Impl Review → Done`
   - Backward (reject): `Plan → Todo`, `Impl Review → Implement`

   **L3** (Level = L3):
   - Forward: `Todo → Plan → Plan Review → Implement → Impl Review → Test → Done`
   - Backward (reject): `Plan → Todo`, `Plan Review → Plan`, `Impl Review → Implement`, `Test → Implement`

   If the target status is not a valid next step from the current status for this item's level, exit with:

   ```
   Error: Cannot move #<NUMBER> from <CURRENT_STATUS> to <TARGET_STATUS>.
   Valid next: <comma-separated list of valid statuses>
   ```

5. Resolve `$PROJECT_ID`, `$ITEM_ID`, and the target option ID.
6. Update the Status field:

```bash
TARGET_OPTION_ID=$(echo "$FIELDS" | jq -r --arg s "$TARGET_STATUS" \
  '.[] | select(.name == "Status") | .options[] | select(.name == $s) | .id')

gh api graphql -f query='
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "'"$PROJECT_ID"'"
    itemId: "'"$ITEM_ID"'"
    fieldId: "'"$STATUS_FIELD_ID"'"
    value: { singleSelectOptionId: "'"$TARGET_OPTION_ID"'" }
  }) {
    projectV2Item { id }
  }
}'
```

7. Output:

```
✅ Moved #<NUMBER> to <TARGET_STATUS>
```

---

### `/kanban-gh edit <ID|name>`

Edit fields on an existing task.

**Steps:**

1. Load config, fetch all items.
2. Resolve `<ID|name>` to an issue number and item node ID.
3. Fetch current values:

```bash
# Issue body and title
ISSUE_DATA=$(gh issue view $NUMBER --repo $REPO --json title,body)
CURRENT_TITLE=$(echo "$ISSUE_DATA" | jq -r '.title')
CURRENT_BODY=$(echo "$ISSUE_DATA" | jq -r '.body')

# Field values from items response
CURRENT_PRIORITY=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Priority") | .name // ""')
CURRENT_LEVEL=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Level") | .name // ""')
CURRENT_TAGS=$(echo "$ITEM" | jq -r '.fieldValues.nodes[] | select(.field.name == "Tags") | .text // ""')
```

4. Use AskUserQuestion to show current values and ask what to change:

   ```
   Editing #<NUMBER>: <CURRENT_TITLE>

   Current values:
     Title:       <CURRENT_TITLE>
     Description: <first 80 chars of body or "(empty)">
     Priority:    <CURRENT_PRIORITY>
     Level:       <CURRENT_LEVEL>
     Tags:        <CURRENT_TAGS or "(none)">

   Which fields do you want to change? (comma-separated list)
   Options: title, description, priority, level, tags
   Enter field names or press Enter to cancel:
   ```

5. For each selected field, use AskUserQuestion to collect the new value.

6. Apply updates:

   - **title changed:**
     ```bash
     gh issue edit $NUMBER --repo $REPO --title "$NEW_TITLE"
     ```
   - **description changed:**
     ```bash
     gh issue edit $NUMBER --repo $REPO --body "$NEW_BODY"
     ```
   - **priority changed:** resolve new option ID and call `updateProjectV2ItemFieldValue` for Priority field.
   - **level changed:** resolve new option ID and call `updateProjectV2ItemFieldValue` for Level field.
   - **tags changed:** call `updateProjectV2ItemFieldValue` for Tags field with `value: { text: "$NEW_TAGS" }`.

7. Output a summary of what changed:

   ```
   ✅ Updated #<NUMBER>:
     priority: medium → high
     tags: "" → backend,auth
   ```

   If nothing was changed, output: `No changes made.`

---

### `/kanban-gh remove <ID|name>`

Remove an item from the project and optionally close its issue.

**Steps:**

1. Load config, fetch all items.
2. Resolve `<ID|name>` to an issue number and item node ID.
3. Resolve `$PROJECT_ID`.
4. Remove from project:

```bash
gh api graphql -f query='
mutation {
  deleteProjectV2Item(input: {
    projectId: "'"$PROJECT_ID"'"
    itemId: "'"$ITEM_ID"'"
  }) {
    deletedItemId
  }
}'
```

5. Use AskUserQuestion:

   ```
   #<NUMBER> has been removed from the project.
   Also close the GitHub issue? (y/n):
   ```

6. If yes:

```bash
gh issue close $NUMBER --repo $REPO
```

7. Output:

```
✅ Removed #<NUMBER> from project
```

Or if the issue was also closed:

```
✅ Removed #<NUMBER> from project and closed issue
```

---

### `/kanban-gh stats`

Show a count of items by status.

**Steps:**

1. Load config, fetch all items.
2. Count items by Status field value.
3. Render table in pipeline order (Todo first, Done last):

```
| Status | Count |
|--------|-------|
| Todo | 5 |
| Plan | 2 |
| Plan Review | 0 |
| Implement | 3 |
| Impl Review | 1 |
| Test | 0 |
| Done | 10 |
| **Total** | **21** |
```

Include all status values even if count is 0.

---

### `/kanban-gh context`

Show a human-readable pipeline state summary.

**Steps:**

1. Load config, fetch all items.
2. Group items by Status field value.
3. Determine "Recently Done": items with Status=Done whose linked issue was closed within the last 3 days. Check closure date via:

```bash
CLOSED_AT=$(gh issue view $NUMBER --repo $REPO --json closedAt --jq '.closedAt')
```

If `closedAt` is within 3 days of now, include in "Recently Done".

4. Determine "Next Todo": items with Status=Todo, up to 5 items (show the lowest issue numbers first as a proxy for priority order).

5. Output:

```
## Pipeline State

**Implement**: #3 Add auth, #7 Fix bug
**Plan Review**: (none)
**Impl Review**: #5 Refactor API
**Test**: (none)
**Plan**: (none)
**Todo**: 8 items (showing next 5): #8 Add logging, #9 Write docs, #10 Setup CI, #11 Add tests, #12 Update deps
**Recently Done** (last 3 days): #1 Setup project, #2 Init repo
**Done total**: 10
```

Show active statuses first (Implement, Plan Review, Impl Review, Test, Plan), then Todo summary, then Recently Done, then total Done count.

If a status has no items, show `(none)`.
