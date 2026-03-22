# kanban-gh

Turn GitHub Issues into an autonomous dev pipeline powered by Claude Code.

Point it at a repo or a Project board. Six AI agents plan it, build it, review the code, and run the tests. No database, no server, minimal setup — just `gh` CLI and a few copied skill files.

- **Two modes** — track tasks on a GitHub Project board, or directly with repo issue labels
- **Fully autonomous pipeline** — plan, implement, review, and test without manual handoffs
- **Seven specialized agents** — Planner, Critic, Builder, Shield, Inspector, Ranger, Refiner
- **Multi-repo support** — one board can track issues across multiple repos
- **Three pipeline levels** — from quick config fixes (L1) to full-feature builds (L3)
- **Just slash commands** — runs as Claude Code skills, nothing else to install

---

## Prerequisites

- [`gh` CLI](https://cli.github.com/) installed and authenticated
- Required scopes: `project` and `repo`
  ```bash
  gh auth login --scopes project,repo
  ```
- Claude Code with skills support

---

## Installation

```bash
git clone https://github.com/evilsquid888/kanban-gh.git
cp -R kanban-gh/kanban-gh         ~/.claude/skills/
cp -R kanban-gh/kanban-gh-init    ~/.claude/skills/
cp -R kanban-gh/kanban-gh-run     ~/.claude/skills/
cp -R kanban-gh/kanban-gh-refine  ~/.claude/skills/
cp -R kanban-gh/kanban-gh-explore ~/.claude/skills/
cp -R kanban-gh/shared            ~/.claude/skills/
```

No `pnpm install`. No server. No `start.sh`.

---

## Quick Start

**With a GitHub Project board:**

```
/kanban-gh-init https://github.com/users/your-username/projects/1
/kanban-gh add Implement user authentication
/kanban-gh-run 1
```

**With repo issues only (no Project board needed):**

```
/kanban-gh-init https://github.com/your-username/your-repo
/kanban-gh add Fix login redirect bug
/kanban-gh-run 1
```

---

## Commands Reference

| Command | Flags / Arguments | Description |
|---------|-------------------|-------------|
| `/kanban-gh-init` | `[url\|number]` | Connect to a GitHub Project or repo |
| `/kanban-gh list` | | View board |
| `/kanban-gh add` | `<title>` | Create task |
| `/kanban-gh move` | `<ID\|name> <status>` | Move task |
| `/kanban-gh edit` | `<ID\|name>` | Edit task |
| `/kanban-gh remove` | `<ID\|name>` | Remove task |
| `/kanban-gh stats` | | Task statistics |
| `/kanban-gh context` | | Pipeline state summary |
| `/kanban-gh-run` | `<ID\|name> [--auto] [--retries N]` | Run full AI pipeline |
| `/kanban-gh-run step` | `<ID\|name>` | Run single pipeline step |
| `/kanban-gh-run --loop` | `[--auto] [--retries N]` | Process all Todo items |
| `/kanban-gh-run review` | `<ID\|name>` | Trigger code review |
| `/kanban-gh-refine` | `<ID\|name>` | Refine requirements |
| `/kanban-gh-explore` | `[topic]` | Explore codebase, seed tasks |

- `--auto` — skip review pauses, apply agent verdicts automatically
- `--retries N` — max rejections before pausing (default: 2)

---

## Pipeline Overview

```
Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
```

| Level | Path | Use Case |
|-------|------|----------|
| L1 Quick | Todo → Implement → Done | Config changes, typo fixes |
| L2 Standard | Todo → Plan → Implement → Impl Review → Done | Bug fixes, refactoring |
| L3 Full | Full 7-column pipeline | New features, architecture |

---

## AI Team

| Agent | Role | Model |
|-------|------|-------|
| Planner | Writes plan + done-when checklist | opus |
| Critic | Scores plan on 3 dimensions | sonnet |
| Builder | Implements the plan | opus |
| Shield | Writes TDD tests | sonnet |
| Inspector | Scores code on 7 dimensions | sonnet |
| Ranger | Runs lint, build, tests | sonnet |
| Refiner | Refines requirements via interview | sonnet |

---

## Architecture

```
                  ┌→ GitHub Projects v2 GraphQL API  (project mode)
Claude Code Skills → gh CLI ─┤
                  └→ GitHub Issues + Labels          (repo mode)
                   → gh CLI → GitHub Issues (comments, both modes)
```

No local database. No web server. GitHub IS the board.

---

## Setting Up Your Board View

After running `/kanban-gh-init`, configure your GitHub Project for a kanban-style board view:

### 1. Open your GitHub Project

Go to `github.com/users/<your-username>/projects/<number>` (the URL printed by `/kanban-gh-init`).

### 2. Switch to Board layout

- Click the **View** dropdown (top-left, next to the view name)
- Select **Board**
- The board groups items by the `Status` field by default — this is what you want

### 3. Configure columns

Your board should show 7 columns matching the pipeline:

```
Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
```

If columns are out of order, drag them to match this sequence. If any pipeline status is missing from the column headers, click **+ New column** and select the missing status option.

### 4. Add useful fields to cards

Click the **⚙️** (settings) icon on the board view, then under **Fields**:
- Enable **Priority** — shows priority badge on each card
- Enable **Level** — shows L1/L2/L3 on each card
- Enable **Labels** — if you use GitHub labels

### 5. Create a filtered view (optional)

For focused work, create additional views:

- **Active Work** — filter: `Status:Plan,Implement,"Plan Review","Impl Review",Test`
- **Backlog** — filter: `Status:Todo`
- **Completed** — filter: `Status:Done`

To create a view: click **+ New view** → choose **Board** or **Table** → set filters.

### 6. Save as default

Click the **▾** next to your view name → **Save changes**. This persists your layout, column order, and field visibility.

---

## License

MIT
