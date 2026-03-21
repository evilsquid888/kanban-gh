# Shared Pipeline Definitions & Agent Templates

Pipeline levels, transition rules, agent context flow, and prompt templates for all 6 agents. Agents interact via GitHub Issue comments instead of database fields.

**References:** `shared/schema.md` (field names, agent nicknames), `shared/graphql.md` (GraphQL operations)

---

## 1. Pipeline Levels and Transitions

### L1 Quick

```
Todo → Implement → Done
```

Agents involved: Builder + Shield. No planning or review steps.

### L2 Standard

```
Todo → Plan → Implement → Impl Review → Done
```

Backward transitions on rejection:

```
Plan → Todo
Impl Review → Implement
```

Agents involved: Planner, Builder, Inspector.

### L3 Full

```
Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
```

Backward transitions on rejection:

```
Plan → Todo
Plan Review → Plan
Impl Review → Implement
Test → Implement
```

Agents involved: Planner, Critic, Builder, Shield, Inspector, Ranger.

### Transition Table

| From | To (forward) | To (reject) | Agent that triggers |
|------|-------------|-------------|---------------------|
| `Todo` | `Plan` | — | Orchestrator |
| `Plan` | `Plan Review` (L3) / `Implement` (L2) | `Todo` | Planner |
| `Plan Review` | `Implement` | `Plan` | Critic |
| `Implement` | `Impl Review` | — | Builder |
| `Impl Review` | `Test` (L3) / `Done` (L2) | `Implement` | Inspector |
| `Test` | `Done` | `Implement` | Ranger |

---

## 2. Agent Context Flow

How agents read and write via GitHub Issues:

| Agent | Reads | Writes | How |
|-------|-------|--------|-----|
| `Planner` | Issue body (description) | Issue comment (plan + decision log + done_when) | `gh issue comment` |
| `Critic` | Issue body + Planner comment | Issue comment (review verdict) | `gh issue comment` |
| `Builder` | Issue body + Planner comment + Critic comment (if exists) | Issue comment (impl notes) | `gh issue comment` |
| `Shield` | Issue body + Builder comment | Issue comment (test notes) | `gh issue comment` |
| `Inspector` | Issue body + Planner comment + Builder comment + Shield comment | Issue comment (review verdict) | `gh issue comment` |
| `Ranger` | Builder comment + Shield comment | Issue comment (test results) | `gh issue comment` |
| `Refiner` | Issue body | Rewrites issue body with structured spec | `gh issue edit` |

**Reading previous agent comments:** Agents fetch all comments via `gh issue view <NUMBER> --repo <REPO> --json comments` and identify relevant ones by signature header (e.g., find the comment starting with `> **Planner**` to read the plan). See Section 5 for parsing snippets.

---

## 3. Agent Prompt Templates

Every agent comment begins with the signature header defined in `shared/schema.md`:

```
> **[Nickname]** `[model]` · [ISO 8601 timestamp]Z
```

Status updates use the `updateFieldValue` mutation from `shared/graphql.md`. The orchestrator resolves `$PROJECT_ID`, `$ITEM_ID`, `$FIELD_ID`, and `$OPTION_ID` before invoking each agent and passes them as environment variables.

---

### 3.1 Planner

**Identity:** `Planner`, `opus`, task #`<NUMBER>`

**Reads:**

```bash
# Issue body (the task description)
BODY=$(gh issue view $NUMBER --repo $REPO --json body --jq '.body')
```

**Guidelines:**

- Produce a concrete, actionable plan grounded in the codebase
- List every file to modify or create
- Identify edge cases and design trade-offs
- Write a `Done When` checklist with at least 2 verifiable criteria
- If you cannot write a meaningful done_when checklist, recommend `/kanban-gh-refine` in your comment and explain why the task needs decomposition

**Output format:**

```markdown
> **Planner** `opus` · <TIMESTAMP>

## Plan
- Files to modify/create
- Step-by-step approach
- Key design decisions
- Edge cases to handle

## Done When
- [ ] <observable outcome 1>
- [ ] <observable outcome 2>

## Key Decisions
| Decision | Why | Alternatives | Trade-off |
|----------|-----|-------------|-----------|
| ... | ... | ... | ... |
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$PLANNER_OUTPUT"
```

**Status update:**

- On completion: update Status → `Plan Review` (L3) or `Implement` (L2) via `updateFieldValue` (see `shared/graphql.md`)

---

### 3.2 Critic

**Identity:** `Critic`, `sonnet`, task #`<NUMBER>`

**Reads:**

```bash
# Issue body
BODY=$(gh issue view $NUMBER --repo $REPO --json body --jq '.body')

# Planner's comment (find by signature header)
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
PLAN=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Planner**"))] | last | .body')
```

**Scoring rubric (1-5 each):**

| Dimension | 1 | 3 | 5 |
|-----------|---|---|---|
| Clarity | Vague, no actionable steps | Reasonable but gaps | Crystal clear, step-by-step |
| Done-When Quality | Missing or untestable | Present but incomplete | Specific, verifiable, comprehensive |
| Reversibility | Destructive changes, no rollback | Partially reversible | Fully reversible, incremental |

**Decision rule:**

- Average >= 4.0 → `approved`
- Average < 3.0 OR any dimension = 1 → `changes_requested`
- Done-When Quality <= 2 → `changes_requested` + recommend `/kanban-gh-refine`

**Output format:**

```markdown
> **Critic** `sonnet` · <TIMESTAMP>

| Dimension | Score | Comment |
|-----------|-------|---------|
| Clarity | /5 | ... |
| Done-When Quality | /5 | ... |
| Reversibility | /5 | ... |
| **Average** | /5 | |

## Verdict: approved / changes_requested
<feedback>
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$CRITIC_OUTPUT"
```

**Status update:**

- On `approved`: update Status → `Implement`
- On `changes_requested`: update Status → `Plan`

---

### 3.3 Builder

**Identity:** `Builder`, `opus`, task #`<NUMBER>`

**Reads:**

```bash
# Issue body
BODY=$(gh issue view $NUMBER --repo $REPO --json body --jq '.body')

# Planner's comment
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
PLAN=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Planner**"))] | last | .body')

# Critic's comment (if L3 pipeline)
REVIEW=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Critic**"))] | last | .body')
```

**Guidelines:**

- Think before coding: read the plan fully, understand the context
- Simplicity first: prefer the simplest correct solution
- Surgical changes: modify only what is necessary, avoid unrelated refactors
- Verify every done_when item: check each criterion from the Planner's checklist
- Leave notes for Shield: mention edge cases worth testing

**Output format:**

```markdown
> **Builder** `opus` · <TIMESTAMP>

## What I Did
<summary of changes>

### Files Modified
- `path/to/file.ts` — <what changed>

### Key Decisions
- <decision and rationale>

### Done When Verification
- [x] <criterion from plan> — <how verified>
- [x] <criterion from plan> — <how verified>

### Notes for Shield
- Edge cases to test
- Areas of risk
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$BUILDER_OUTPUT"
```

**Status update:**

- Builder does NOT update status. The orchestrator advances to `Impl Review` after Builder completes.

---

### 3.4 Shield

**Identity:** `Shield`, `sonnet`, task #`<NUMBER>`

**Reads:**

```bash
# Issue body
BODY=$(gh issue view $NUMBER --repo $REPO --json body --jq '.body')

# Builder's comment (find by signature header)
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
IMPL=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Builder**"))] | last | .body')
```

**Guidelines:**

- Write tests that cover the Builder's changes
- Focus on edge cases mentioned in the Builder's "Notes for Shield"
- Verify done_when criteria are testable
- Prefer integration tests over unit tests when both are viable
- Keep tests focused: one assertion per test where practical

**Output format:**

```markdown
> **Shield** `sonnet` · <TIMESTAMP>

## Tests Written

### New Test Files
- `path/to/test-file.test.ts` — <what it tests>

### Edge Cases Covered
- <edge case> — <how tested>

### Coverage Notes
- <any gaps or caveats>
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$SHIELD_OUTPUT"
```

**Status update:**

- Shield does NOT update status. The orchestrator advances to `Impl Review` after Shield completes.

---

### 3.5 Inspector

**Identity:** `Inspector`, `sonnet`, task #`<NUMBER>`

**Reads:**

```bash
# Issue body
BODY=$(gh issue view $NUMBER --repo $REPO --json body --jq '.body')

# Planner's comment
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
PLAN=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Planner**"))] | last | .body')

# Builder's comment
IMPL=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Builder**"))] | last | .body')

# Shield's comment (if L3 pipeline)
TESTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Shield**"))] | last | .body')
```

**Scoring rubric (1-5 each):**

| Dimension | 1 | 3 | 5 |
|-----------|---|---|---|
| Code Quality | Spaghetti, no structure | Acceptable, some issues | Clean, well-organized |
| Error Handling | No error handling | Basic happy path | Comprehensive, graceful |
| Type Safety | No types, any everywhere | Partial typing | Fully typed, strict |
| Security | Vulnerabilities present | No obvious issues | Proactively hardened |
| Performance | Obvious bottlenecks | Acceptable | Optimized where it matters |
| Test Coverage | No tests | Partial coverage | Comprehensive, edge cases |
| Completion | Done-when items not met | Most items met | All items verified |

**Decision rule:**

- Average >= 4.0 → `approved`
- Average < 3.0 OR Security = 1 OR Type Safety = 1 → `changes_requested`
- Completion = 1 → hard reject (always `changes_requested`)

**Output format:**

```markdown
> **Inspector** `sonnet` · <TIMESTAMP>

| Dimension | Score | Comment |
|-----------|-------|---------|
| Code Quality | /5 | ... |
| Error Handling | /5 | ... |
| Type Safety | /5 | ... |
| Security | /5 | ... |
| Performance | /5 | ... |
| Test Coverage | /5 | ... |
| Completion | /5 | ... |
| **Average** | /5 | |

## Verdict: approved / changes_requested
<feedback>
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$INSPECTOR_OUTPUT"
```

**Status update:**

- On `approved`: update Status → `Test` (L3) or `Done` (L2)
- On `changes_requested`: update Status → `Implement`

---

### 3.6 Ranger

**Identity:** `Ranger`, `sonnet`, task #`<NUMBER>`

**Reads:**

```bash
# Builder's comment
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
IMPL=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Builder**"))] | last | .body')

# Shield's comment
TESTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Shield**"))] | last | .body')
```

**Guidelines:**

- Run the full lint, build, and test suite
- Report results with enough detail to diagnose failures
- Do not fix code — only report results
- If tests fail, include the failure output in the details block

**Output format:**

```markdown
> **Ranger** `sonnet` · <TIMESTAMP>

## Test Results
- Lint: pass / fail
- Build: pass / fail
- Tests: pass / fail (<N> passed, <M> failed)

## Verdict: pass / fail

<details>
<summary>Full output</summary>

<lint/build/test output here>

</details>
```

**Writes:**

```bash
gh issue comment $NUMBER --repo $REPO --body "$RANGER_OUTPUT"
```

**Status update:**

- On `pass`: update Status → `Done`
- On `fail`: update Status → `Implement`

---

## 4. Retry Limit Handling

The orchestrator tracks how many times each review agent (Critic, Inspector, Ranger) has rejected the current item. The limit is configurable via `retries` in `.claude/kanban-gh.json` (default: `2`).

### Behavior on Nth rejection (N = retries limit):

1. Post a comment on the issue:

```markdown
> **[kanban-gh]** {Agent} has rejected this item {N} times. Pausing for human review.
```

2. Leave the item at its current status (do not transition backward)
3. Exit the pipeline for this item, even if running in `--auto` mode

### Rejection count tracking:

- The orchestrator counts rejection comments from each review agent by parsing issue comments
- Count resets to 0 when the agent posts an `approved` or `pass` verdict
- Each review step has its own independent counter

### Counting rejections from comments:

```bash
# Count Critic rejections since last approval
CRITIC_REJECTIONS=$(echo "$COMMENTS" | jq '[
  .[] | select(.body | startswith("> **Critic**"))
  | {body, approved: (.body | test("## Verdict: approved"))}
] | [foreach .[] as $c (0; if $c.approved then 0 else . + 1)] | last // 0')
```

---

## 5. Comment Parsing Reference

Agents identify previous agent outputs by their signature header (`> **AgentName**`). Always use the **most recent** comment from each agent.

### Fetch all comments

```bash
COMMENTS=$(gh issue view $NUMBER --repo $REPO --json comments --jq '.comments')
```

### Find most recent comment by agent

```bash
# Planner
PLAN=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Planner**"))] | last | .body')

# Critic
REVIEW=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Critic**"))] | last | .body')

# Builder
IMPL=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Builder**"))] | last | .body')

# Shield
TESTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Shield**"))] | last | .body')

# Inspector
CODE_REVIEW=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Inspector**"))] | last | .body')

# Ranger
TEST_RESULTS=$(echo "$COMMENTS" | jq -r '[.[] | select(.body | startswith("> **Ranger**"))] | last | .body')
```

### Check if a comment exists

```bash
if [ "$PLAN" = "null" ] || [ -z "$PLAN" ]; then
  echo "No Planner comment found"
fi
```

### Extract verdict from a review comment

```bash
# Extract verdict line from Critic/Inspector/Ranger comment
VERDICT=$(echo "$REVIEW_COMMENT" | grep -oP '## Verdict: \K\S+')
# Result: "approved", "changes_requested", "pass", or "fail"
```
