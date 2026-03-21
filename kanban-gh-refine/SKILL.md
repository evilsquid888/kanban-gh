---
name: kanban-gh-refine
description: "Refine backlog requirements through structured user interview. Updates the GitHub issue body with a clear specification. Usage: /kanban-gh-refine <ID|name>"
license: MIT
---

> Shared context: read ~/.claude/skills/shared/schema.md for field names, status values, and config format. Read ~/.claude/skills/shared/graphql.md for GraphQL operations.

# kanban-gh-refine

Requirements refinement skill that guides you through a structured interview to clarify a backlog task, then updates the GitHub issue body with a clear specification.

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

## ID Resolution

Accepts `<ID|name>` as an argument. Resolve it as follows:

### Issue number

If the argument is a plain integer or starts with `#`, strip the `#` prefix and use it directly as the issue number:

```bash
NUMBER=$(echo "$ARG" | sed 's/^#//')
```

### Partial title match

If the argument is not numeric, treat it as a case-insensitive substring to match against item titles. Fetch all items via `getProjectItems` (see graphql.md), then filter:

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

  Then use AskUserQuestion to ask the user which one to use.

---

## Procedure

---

### ① Resolve ID

Resolve the `<ID|name>` argument to an issue number using the ID Resolution rules above.

---

### ② Read Issue

```bash
ISSUE_DATA=$(gh issue view $NUMBER --repo $REPO --json title,body)
TITLE=$(echo "$ISSUE_DATA" | jq -r '.title')
BODY=$(echo "$ISSUE_DATA" | jq -r '.body')
```

---

### ③ Display Current State

Output:

```
Current requirements for #<NUMBER>: <TITLE>
```

Then show the current body. If `$BODY` is empty or null, show `(no description)`.

---

### ④ Identify Gaps

Before interviewing the user, internally assess which of the following dimensions are missing or unclear in the current body:

- **WHAT**: What exactly should be built?
- **WHY**: What problem does this solve?
- **SCOPE**: What's included vs excluded?
- **ACCEPTANCE**: How do we know it's done?
- **CONSTRAINTS**: Technical limitations?
- **EDGE CASES**: Error states, boundary conditions?
- **DEPENDENCIES**: Other tasks or external systems?

Use this gap analysis to form targeted questions in the next step.

---

### ⑤ Interview User (MANDATORY)

Run up to 3 rounds of questions using AskUserQuestion. Each round:

- Ask 1–4 focused questions grouped by topic.
- Prioritize the most critical gaps first.
- Stop early if the user says "enough" or all gaps are filled.

Example first round for a sparse task:

```
I'd like to clarify the requirements for #<NUMBER>: <TITLE>.

1. What exactly should be built? (brief description of the feature or change)
2. What problem does this solve for users?
3. Are there any technical constraints or existing systems to integrate with?
4. How will we know this is done — what's the key acceptance criterion?
```

Adapt subsequent rounds based on answers received. Do not re-ask questions that were already answered.

---

### ⑥ Synthesize Refined Description

Using the answers collected, produce a structured description using this template:

```markdown
## Goal
<what and why — combine the WHAT and WHY answers into a clear statement>

## Scope
**In scope:**
- ...

**Out of scope:**
- ...

## Acceptance Criteria
- [ ] <verifiable criterion>
- [ ] <verifiable criterion>

## Edge Cases
- ...

## Constraints
- ...
```

Omit any section that has no content based on the interview.

---

### ⑦ Present to User

Display the full refined description, then use AskUserQuestion:

```
Refined requirements for #<NUMBER>: <TITLE>

<REFINED_DESCRIPTION>

What would you like to do?
  approve  — save this to the issue
  edit     — continue refining
  cancel   — discard changes
```

If the user chooses **edit**, return to step ⑤ for another round of questions (up to the 3-round limit). If the round limit is reached and the user still wants to edit, allow one final freeform edit pass before presenting again.

If the user chooses **cancel**, output:

```
Cancelled. No changes made to #<NUMBER>.
```

And exit.

---

### ⑧ Save on Approve

```bash
gh issue edit $NUMBER --repo $REPO --body "$NEW_BODY"
```

If the interview surfaced clear values for Priority, Level, or Tags that differ from current values, optionally update those project fields using `updateProjectV2ItemFieldValue` (see graphql.md for mutation patterns). Only do this if the values were explicitly discussed — do not infer or guess.

---

### ⑨ Post Agent Log Comment

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
gh issue comment $NUMBER --repo $REPO --body "> **Refiner** \`sonnet\` · $TIMESTAMP
> Requirements refined. Updated issue body with structured spec."
```

---

### ⑩ Confirm

Output:

```
✅ Requirements updated for #<NUMBER>.
```

---

## Error Handling

- **Config missing**: Exit with error directing user to run `/kanban-gh-init`.
- **Item not found**: Exit with error from ID resolution.
- **Issue read failure**: Report the `gh` error and exit.
- **Issue edit failure**: Report the `gh` error; do not post the log comment.
- **Cancel at any point**: Leave the issue unchanged and exit cleanly.
