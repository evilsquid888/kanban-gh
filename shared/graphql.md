# Shared GraphQL Templates

GraphQL query and mutation templates for GitHub Projects v2. All `$VARIABLES` are substituted at runtime by skill scripts.

---

## Node ID Resolution

GitHub Projects v2 requires node IDs (opaque base64 strings) rather than human-readable numbers. These are fetched at runtime and are not cached between invocations.

| Variable | Source |
|----------|--------|
| `$PROJECT_ID` | Returned by `getProject` |
| `$FIELD_ID` | Returned by `getProjectFields` |
| `$OPTION_ID` | Returned by `getProjectFields` (nested inside each field's `options`) |
| `$ITEM_ID` | Returned by `getProjectItems` |
| `$ISSUE_NODE_ID` | `gh api /repos/{owner}/{repo}/issues/{number} --jq .node_id` |

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
OPTION_ID=$(echo "$FIELDS" | jq -r '.[] | select(.name == "Status") | .options[] | select(.name == "In Progress") | .id')
```

---

### 3. `createField`

Create a new field on a project. Two variants are provided: single-select (with options) and plain text.

**Single-select field:**

```graphql
mutation {
  createProjectV2Field(input: {
    projectId: "$PROJECT_ID"
    dataType: SINGLE_SELECT
    name: "$FIELD_NAME"
    singleSelectOptions: [
      {name: "$OPTION_1", color: BLUE},
      {name: "$OPTION_2", color: GREEN},
      {name: "$OPTION_3", color: RED}
    ]
  }) {
    projectV2Field { id }
  }
}
```

**Text field:**

```graphql
mutation {
  createProjectV2Field(input: {
    projectId: "$PROJECT_ID"
    dataType: TEXT
    name: "$FIELD_NAME"
  }) {
    projectV2Field { id }
  }
}
```

**Usage:**

```bash
# Single-select field
gh api graphql -f query='
mutation {
  createProjectV2Field(input: {
    projectId: "$PROJECT_ID"
    dataType: SINGLE_SELECT
    name: "$FIELD_NAME"
    singleSelectOptions: [
      {name: "$OPTION_1", color: BLUE},
      {name: "$OPTION_2", color: GREEN},
      {name: "$OPTION_3", color: RED}
    ]
  }) {
    projectV2Field { id }
  }
}' -f projectId="$PROJECT_ID" -f fieldName="$FIELD_NAME"

# Text field
gh api graphql -f query='
mutation {
  createProjectV2Field(input: {
    projectId: "$PROJECT_ID"
    dataType: TEXT
    name: "$FIELD_NAME"
  }) {
    projectV2Field { id }
  }
}' -f projectId="$PROJECT_ID" -f fieldName="$FIELD_NAME"
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
