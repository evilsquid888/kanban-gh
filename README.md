# kanban-gh

AI-powered kanban pipeline for Claude Code — powered by GitHub Projects.

Seven autonomous agents, one GitHub Project board. No database. No local server.

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

```
/kanban-gh-init https://github.com/users/your-username/projects/1
/kanban-gh add Implement user authentication
/kanban-gh-run 1
```

---

## Commands Reference

| Command | Description |
|---------|-------------|
| `/kanban-gh-init [url]` | Connect to a GitHub Project |
| `/kanban-gh list` | View board |
| `/kanban-gh add <title>` | Create task |
| `/kanban-gh move <ID\|name> <status>` | Move task |
| `/kanban-gh edit <ID\|name>` | Edit task |
| `/kanban-gh remove <ID\|name>` | Remove task |
| `/kanban-gh stats` | Task statistics |
| `/kanban-gh context` | Pipeline state summary |
| `/kanban-gh-run <ID\|name> [--auto]` | Run AI pipeline |
| `/kanban-gh-run step <ID\|name>` | Run single pipeline step |
| `/kanban-gh-run --loop [--auto]` | Process all todo items |
| `/kanban-gh-run review <ID\|name>` | Trigger code review |
| `/kanban-gh-refine <ID\|name>` | Refine requirements |
| `/kanban-gh-explore [topic]` | Explore codebase, seed tasks |

---

## Pipeline Overview

```
Todo → Plan → Plan Review → Implement → Impl Review → Test → Done
```

| Level | Path | Use Case |
|-------|------|----------|
| L1 Quick | Todo → Implement → Done | Config changes, typo fixes |
| L2 Standard | Todo → Plan → Implement → Review → Done | Bug fixes, refactoring |
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

---

## Architecture

```
Claude Code Skills → gh CLI → GitHub Projects v2 GraphQL API
                   → gh CLI → GitHub Issues (comments)
```

No local database. No web server. GitHub IS the board.

---

## License

MIT
