# CLAUDE.md

## Project Overview

kanban-gh is a set of Claude Code skills that turn GitHub Issues into an autonomous AI dev pipeline. It uses the `gh` CLI and GitHub Projects v2 (or repo labels) as its data store -- no database, no server, no web UI. Seven specialized AI agents (Planner, Critic, Builder, Shield, Inspector, Ranger, Refiner) collaborate through GitHub Issue comments to plan, implement, review, and test code changes.

## Language & Framework

- **Language:** Markdown-based Claude Code skills (no compiled code)
- **Runtime:** Claude Code with skills support + `gh` CLI
- **APIs:** GitHub Projects v2 GraphQL API, GitHub REST API (via `gh`)
- **Dependencies:** `gh` CLI (authenticated with `project,repo` scopes), `jq`

## Repository Structure

```
kanban-gh/              # Main task CRUD skill (add, list, move, edit, remove, stats, context)
kanban-gh-init/         # Initialization skill (connect to Project or repo, create fields/labels)
kanban-gh-run/          # Pipeline orchestration (dispatches agents, manages transitions)
kanban-gh-refine/       # Requirements refinement via structured interview
kanban-gh-explore/      # Codebase exploration and task seeding
shared/
  schema.md             # Canonical field names, status values, config format, label conventions
  graphql.md            # GraphQL query/mutation templates for Projects v2 API
  pipeline.md           # Pipeline levels (L1-L3), transitions, agent prompt templates, scoring rubrics
docs/superpowers/       # Design specs and implementation plans
```

Each skill directory contains a single `SKILL.md` file that serves as both documentation and the skill definition (read by Claude Code at runtime).

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

No build step. No package manager. No server.

## Build / Run / Test Commands

There are no build or test commands. This project consists entirely of Markdown skill files consumed by Claude Code at runtime. There is no compiled code, no test suite, and no CI pipeline.

To use, invoke slash commands in Claude Code after installation:
- `/kanban-gh-init <url>` -- connect to a GitHub Project or repo
- `/kanban-gh add <title>` -- create a task
- `/kanban-gh-run <ID>` -- run the AI pipeline on a task
- `/kanban-gh list` -- view the board

## Key Architecture Notes

- **Two modes:** Project mode (GitHub Projects v2 GraphQL API) and Repo mode (GitHub Issue labels). Mode is set during `/kanban-gh-init` and stored in `.claude/kanban-gh.json`.
- **No local state beyond config:** `.claude/kanban-gh.json` is the only local file. All task data lives in GitHub (Projects fields or Issue labels).
- **Agent I/O via Issue comments:** Each agent reads previous agents' comments by matching the signature header (`> **AgentName**`) and writes its output as a new comment.
- **Three pipeline levels:** L1 (quick, 2 steps), L2 (standard, 5 steps), L3 (full, 7 steps with all agents).
- **Retry limit handling:** Review agents (Critic, Inspector, Ranger) track rejection counts from comments. After N rejections (configurable, default 2), the pipeline pauses for human review.
- **Multi-repo support:** In project mode, a single board can track issues across multiple repos. Per-issue repo is resolved from the item's URL.

## Coding Conventions

- Each skill is a single `SKILL.md` with YAML front matter (`name`, `description`, `license`).
- Shared references are loaded via the `> Shared context:` directive at the top of each SKILL.md.
- All shell snippets use `gh` CLI and `jq` for GitHub API interactions.
- Agent comments follow a strict signature header format: `> **[Nickname]** \`[model]\` · [ISO 8601 timestamp]Z`
- Status values use Title Case in display (`Plan Review`) and kebab-case in labels (`status:plan-review`).
- Config is always loaded at the start of each command via the standard snippet from `shared/schema.md`.
- ID resolution supports both issue numbers (`#12`) and partial title matching.
