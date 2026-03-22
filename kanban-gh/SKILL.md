---
name: kanban-gh
description: "Manage tasks in GitHub Projects or via repo labels. Supports add, list, move, edit, remove, stats, context. Uses GitHub Projects v2 or repo labels as the data store. Run /kanban-gh-init first."
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md for field names, status values, config format, and label naming convention. Read ~/.claude/skills/shared/graphql.md for GraphQL operations and label operations. Read ~/.claude/skills/shared/pipeline.md for valid status transitions.

# kanban-gh

CRUD commands for managing tasks in a GitHub Projects v2 board (project mode) or via repo labels (repo mode). Requires `/kanban-gh-init` to have been run first.

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

All subsequent operations branch on `$MODE`. When `$MODE` is `"project"`, use GraphQL operations from `shared/graphql.md`. When `$MODE` is `"repo"`, use label operations from `shared/graphql.md` sections 8–15.

---

## ID Resolution

Many commands accept `<ID|name>` as an argument. Resolve it as follows:

### Issue number

If the argument is a plain integer or starts with `#`, strip the `#` prefix and use it directly as the issue number:

```bash
NUMBER=$(echo "$ARG" | sed 's/^#//')
```

### Partial title match

If the argument is not numeric, treat it as a case-insensitive substring to match against item titles.

**Project mode:** Fetch all items via `getProjectItems` (see GraphQL section), then filter:

```bash
MATCHES=$(echo "$ITEMS" | jq --arg q "$ARG" '
  [.[] | select(.content.title | ascii_downcase | contains($q | ascii_downcase))]
')
COUNT=$(echo "$MATCHES" | jq 'length')
```

**Repo mode:** Search issues directly:

```bash
MATCHES=$(gh issue list --repo "$REPO" --state open --search "$ARG" --json number,title,labels --limit 50)
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

Most commands need to fetch all tracked items. The approach differs by mode.

### Project mode

Resolve `$PROJECT_ID` first, then call `getProjectItems`:

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

### Repo mode

Fetch all open issues plus closed done issues, then filter to tracked items (see `shared/graphql.md` section 13):

```bash
# All open issues
ALL_OPEN=$(gh issue list --repo "$REPO" --state open --json number,title,body,url,labels --limit 200)

# Closed issues with status:done
DONE_CLOSED=$(gh issue list --repo "$REPO" --state closed --label "status:done" --json number,title,body,url,labels --limit 100)

# Combine and deduplicate
ALL_ISSUES=$(echo "$ALL_OPEN" "$DONE_CLOSED" | jq -s 'add | unique_by(.number)')

# Filter to tracked issues (have at least one status: label)
ITEMS=$(echo "$ALL_ISSUES" | jq '[.[] | select(any(.labels[].name; startswith("status:")))]')
```

Parse fields from a single item in repo mode:

```bash
parse_item_repo() {
  local ITEM="$1"
  NUMBER=$(echo "$ITEM" | jq -r '.number')
  TITLE=$(echo "$ITEM" | jq -r '.title')
  STATUS_LABEL=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("status:"))] | first // ""' | sed 's/^status://')
  PRIORITY=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("priority:"))] | first // ""' | sed 's/^priority://')
  LEVEL=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("level:"))] | first // ""' | sed 's/^level://')
  # Convert label value to display name (see shared/schema.md mapping)
  STATUS=$(label_to_display "$STATUS_LABEL")
}
```

---

## Shared: Resolve Field and Option IDs (Project Mode Only)

**This section applies only when `$MODE` is `"project"`.** In repo mode, there are no field/option IDs — labels are used directly by name.

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

**Status column order:** Backlog, Todo, Plan, Plan Review, Implement, Impl Review, Test, Done

**Steps:**

1. Load config and fetch all items (see Shared sections above).
2. Parse each item's field values:

**Project mode:**

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

**Repo mode:**

```bash
parse_item_repo() {
  local ITEM="$1"
  NUMBER=$(echo "$ITEM" | jq -r '.number')
  TITLE=$(echo "$ITEM" | jq -r '.title')
  STATUS_LABEL=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("status:"))] | first // ""' | sed 's/^status://')
  PRIORITY=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("priority:"))] | first // ""' | sed 's/^priority://')
  LEVEL=$(echo "$ITEM" | jq -r '[.labels[].name | select(startswith("level:"))] | first // ""' | sed 's/^level://')
  STATUS=$(label_to_display "$STATUS_LABEL")
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

If there are no items, output: `No items found.`

In repo mode, if there are open issues without any `status:` label, show them in a separate section:

```
### Untracked Issues (no status label)
| # | Title |
|---|-------|
| 15 | Some old issue |
```

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

4. **Branch by mode:**

### Project mode (steps 4–8)

4p. Resolve the issue node ID:

```bash
ISSUE_NODE_ID=$(gh api /repos/$REPO/issues/$NUMBER --jq .node_id)
```

5p. Resolve `$PROJECT_ID` (see Shared section).

6p. Add the issue to the project:

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

7p. Resolve field and option IDs (see Shared section), then set Status=Todo, Priority, and Level:

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

8p. If tags were provided, set the Tags text field:

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

### Repo mode (step 4r)

4r. Add status, priority, and level labels to the issue:

```bash
# Convert priority/level to label values (already lowercase)
gh issue edit $NUMBER --repo "$REPO" --add-label "status:todo,priority:$PRIORITY,level:$LEVEL"
```

No project ID, field ID, or option ID resolution needed.

---

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

5. **Branch by mode:**

### Project mode

5p. Resolve `$PROJECT_ID`, `$ITEM_ID`, and the target option ID.

6p. Update the Status field:

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

### Repo mode

5r. Resolve per-issue repo (see `shared/schema.md` Multi-Repo Resolution), then convert current and target status to label format and swap labels:

```bash
ISSUE_URL=$(echo "$ITEM" | jq -r '.url // ""')
if [ -n "$ISSUE_URL" ] && [ "$ISSUE_URL" != "null" ]; then
  ISSUE_REPO=$(echo "$ISSUE_URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/issues/[0-9]+|\1|')
else
  ISSUE_REPO="$REPO"
fi

OLD_LABEL="status:$(display_to_label "$CURRENT_STATUS")"
NEW_LABEL="status:$(display_to_label "$TARGET_STATUS")"

gh issue edit $NUMBER --repo "$ISSUE_REPO" --remove-label "$OLD_LABEL" --add-label "$NEW_LABEL"
```

---

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
3. Resolve the per-issue repo (see `shared/schema.md` Multi-Repo Resolution):

```bash
# Project mode: URL is in content.url; Repo mode: URL is in .url
ISSUE_URL=$(echo "$ITEM" | jq -r '.content.url // .url // ""')
if [ -n "$ISSUE_URL" ] && [ "$ISSUE_URL" != "null" ]; then
  ISSUE_REPO=$(echo "$ISSUE_URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/issues/[0-9]+|\1|')
else
  ISSUE_REPO="$REPO"
fi
```

4. Fetch current values:

```bash
# Issue body and title
ISSUE_DATA=$(gh issue view $NUMBER --repo $ISSUE_REPO --json title,body)
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
     gh issue edit $NUMBER --repo $ISSUE_REPO --title "$NEW_TITLE"
     ```
   - **description changed:**
     ```bash
     gh issue edit $NUMBER --repo $ISSUE_REPO --body "$NEW_BODY"
     ```
   - **priority changed:**
     - **Project mode:** resolve new option ID and call `updateProjectV2ItemFieldValue` for Priority field.
     - **Repo mode:** swap priority labels:
       ```bash
       gh issue edit $NUMBER --repo "$ISSUE_REPO" --remove-label "priority:$OLD" --add-label "priority:$NEW"
       ```
   - **level changed:**
     - **Project mode:** resolve new option ID and call `updateProjectV2ItemFieldValue` for Level field.
     - **Repo mode:** swap level labels:
       ```bash
       gh issue edit $NUMBER --repo "$ISSUE_REPO" --remove-label "level:$OLD" --add-label "level:$NEW"
       ```
   - **tags changed:**
     - **Project mode:** call `updateProjectV2ItemFieldValue` for Tags field with `value: { text: "$NEW_TAGS" }`.
     - **Repo mode:** Tags remain as text in the issue body — no label change needed. If tags need storing, append them to the issue body.

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
3. **Branch by mode:**

### Project mode

3p. Resolve `$PROJECT_ID`.

4p. Remove from project:

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

### Repo mode

3r. Resolve per-issue repo (see `shared/schema.md` Multi-Repo Resolution), then remove all kanban labels from the issue:

```bash
ISSUE_URL=$(echo "$ITEM" | jq -r '.url // ""')
if [ -n "$ISSUE_URL" ] && [ "$ISSUE_URL" != "null" ]; then
  ISSUE_REPO=$(echo "$ISSUE_URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/issues/[0-9]+|\1|')
else
  ISSUE_REPO="$REPO"
fi

KANBAN_LABELS=$(gh issue view $NUMBER --repo "$ISSUE_REPO" --json labels --jq '[.labels[].name | select(startswith("status:") or startswith("priority:") or startswith("level:"))] | join(",")')
if [ -n "$KANBAN_LABELS" ]; then
  gh issue edit $NUMBER --repo "$ISSUE_REPO" --remove-label "$KANBAN_LABELS"
fi
```

---

5. Use AskUserQuestion:

   ```
   #<NUMBER> has been removed from tracking.
   Also close the GitHub issue? (y/n):
   ```

6. If yes:

```bash
gh issue close $NUMBER --repo $ISSUE_REPO
```

7. Output:

```
✅ Removed #<NUMBER> from tracking
```

Or if the issue was also closed:

```
✅ Removed #<NUMBER> from tracking and closed issue
```

---

### `/kanban-gh stats`

Show a count of items by status.

**Steps:**

1. Load config, fetch all items.
2. Count items by Status field value.
   - **Project mode:** group by `fieldValues.nodes[] | select(.field.name == "Status") | .name`
   - **Repo mode:** group by `status:` label value, converted to display name via `label_to_display`
3. Render table in pipeline order (Backlog first, Done last):

```
| Status | Count |
|--------|-------|
| Backlog | 3 |
| Todo | 5 |
| Plan | 2 |
| Plan Review | 0 |
| Implement | 3 |
| Impl Review | 1 |
| Test | 0 |
| Done | 10 |
| **Total** | **24** |
```

Include all status values even if count is 0 (including Backlog).

---

### `/kanban-gh context`

Show a human-readable pipeline state summary.

**Steps:**

1. Load config, fetch all items.
2. Group items by Status field value.
   - **Project mode:** group by `fieldValues.nodes[] | select(.field.name == "Status") | .name`
   - **Repo mode:** group by `status:` label value, converted to display name via `label_to_display`
3. Determine "Recently Done": items with Status=Done whose linked issue was closed within the last 3 days. Resolve per-issue repo first (see `shared/schema.md` Multi-Repo Resolution), then check closure date:

```bash
# Resolve ISSUE_REPO from item URL (content.url in project mode, .url in repo mode)
ISSUE_URL=$(echo "$ITEM" | jq -r '.content.url // .url // ""')
if [ -n "$ISSUE_URL" ] && [ "$ISSUE_URL" != "null" ]; then
  ISSUE_REPO=$(echo "$ISSUE_URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/issues/[0-9]+|\1|')
else
  ISSUE_REPO="$REPO"
fi

CLOSED_AT=$(gh issue view $NUMBER --repo $ISSUE_REPO --json closedAt --jq '.closedAt')
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
**Backlog**: 3 items
**Recently Done** (last 3 days): #1 Setup project, #2 Init repo
**Done total**: 10
```

Show active statuses first (Implement, Plan Review, Impl Review, Test, Plan), then Todo summary, then Backlog summary, then Recently Done, then total Done count.

If a status has no items, show `(none)`.
