# AGENTS.md

## Overview

This is a **content-only repository** — no code, no build system, no tests. It
contains a collection of reusable **Claude Code skills**: self-contained markdown
folders that extend Claude Code's capabilities when installed into a project.

The repo is consumed in three ways:

1. **Direct install** via `npx degit lucasfe/skills/<category>/<name> ~/.claude/skills/<name>`
2. **AgentHub catalog** at [agenthub.lucasfe.com](https://agenthub.lucasfe.com) (reads from GitHub Contents API)
3. **Ralph** — an autonomous agent that picks up open GitHub issues and implements them as PRs

## Repository structure

```
/
├── AGENTS.md                          # this file
├── README.md                          # user-facing docs and install instructions
├── PROMPT.md                          # project context for Ralph (autonomous agent)
├── ralph.config.sh                    # Ralph configuration (branch, merge, commands)
├── .claude/commands/ralph.md          # slash command to trigger Ralph loop
├── development/                       # development-focused skills
│   ├── grill-me/                      #   relentless design interview
│   ├── improve-codebase-architecture/ #   find deepening opportunities via ADRs + domain language
│   ├── request-refactor-plan/         #   interview-driven refactor planning → GitHub issue
│   ├── supabase-postgres-best-practices/ # Postgres optimization reference (8 rule categories)
│   └── tdd/                           #   red-green-refactor vertical-slice TDD loop
├── project-management/                # project management skills
│   ├── sec-consult/                   #   generate security consult ticket from specs
│   ├── spec-to-jira/                  #   PRD + tech spec → idempotent Jira hierarchy
│   ├── to-issues/                     #   plan → vertical-slice GitHub issues
│   ├── to-prd/                        #   conversation context → PRD GitHub issue
│   └── triage-issue/                  #   bug root-cause analysis → TDD fix plan issue
└── meta/                              # meta/tooling skills
    ├── find-skills/                   #   discover + install skills from the ecosystem
    └── opensquad/                     #   multi-agent orchestration framework
```

## Skill anatomy

Every skill is a folder inside a category directory:

```
<category>/<skill-name>/
├── SKILL.md           # required — YAML frontmatter (name, description) + instructions
└── *.md               # optional — auxiliary references the skill points to
```

**Invariants:**
- Folder names are **kebab-case**, lowercase, ASCII-only
- Folder name must match the `name` field in `SKILL.md` frontmatter exactly
- `SKILL.md` must be uppercase (Claude Code loader is case-sensitive)
- Frontmatter requires at least `name` and `description`
- Skills never live at the repo root — always inside a category folder

## Skills catalog

### Development

| Skill | Description |
|---|---|
| **grill-me** | Interviews the user one question at a time about a plan or design, walking down every branch of the decision tree until reaching shared understanding. |
| **improve-codebase-architecture** | Reads CONTEXT.md and ADRs, explores the codebase for shallow modules and friction points, applies a "deletion test" to find unnecessary abstractions, and presents deepening candidates. Updates CONTEXT.md and optionally records ADRs. |
| **request-refactor-plan** | Guides user through an interview to understand refactoring goals, explores the codebase to verify assertions, designs test coverage, breaks implementation into minimal commits, and files a GitHub issue with the full plan. |
| **supabase-postgres-best-practices** | Comprehensive Postgres performance guide organized into 8 priority-based rule categories (Query Performance, Connection Management, Security & RLS, Schema Design, Concurrency & Locking, Data Access Patterns, Monitoring & Diagnostics, Advanced Features) with correct/incorrect examples. |
| **tdd** | Drives features or fixes through strict vertical-slice TDD: one test, one implementation, repeat. Includes planning, tracer bullet, incremental loop, and refactor phases. References auxiliary files on deep modules, mocking, and interface design. |

### Project Management

| Skill | Description |
|---|---|
| **sec-consult** | Generates a security consult ticket from a tech spec, PRD, or project reference. Accepts Jira ticket IDs, Confluence URLs, or local files as input. Produces a structured consult text and a Jira-ready summary. |
| **spec-to-jira** | Accepts a PRD + tech spec (local files, Google Docs, Jira key, or pasted content), detects milestones, derives user stories, builds a coverage matrix, identifies gaps, infers sub-task dependencies, writes a review plan file, then idempotently syncs the hierarchy to Jira. |
| **to-issues** | Breaks a plan or PRD into independently-grabbable GitHub issues using vertical-slice tracer-bullet decomposition. Classifies slices as HITL (human-in-the-loop) or AFK (autonomous). Creates issues in dependency order. |
| **to-prd** | Synthesizes the current conversation and codebase understanding into a PRD — explores the repo, identifies major modules, proposes testing strategy — and submits it as a GitHub issue. |
| **triage-issue** | Investigates a bug by exploring the codebase to find root cause, determines the minimal fix, designs a TDD fix plan with RED-GREEN cycles, and creates a GitHub issue with root cause analysis and acceptance criteria. |

### Meta

| Skill | Description |
|---|---|
| **find-skills** | Discovers and installs skills from the open agent skills ecosystem using the Skills CLI (`npx skills`). Checks the skills.sh leaderboard, searches by query, verifies quality, and installs if desired. |
| **opensquad** | Multi-agent orchestration framework for creating and running AI squads. Handles onboarding, squad creation/execution, skills browsing, and company profile management. |

## Working with this repo

### Adding a skill

1. Pick or create a category folder (`development/`, `project-management/`, `meta/`)
2. Create a kebab-case folder inside it
3. Add `SKILL.md` with YAML frontmatter (`name`, `description`) and markdown instructions
4. Optionally add auxiliary `.md` reference files
5. Update the tables in `README.md`
6. PR to `main`, squash-merge

### Ralph (autonomous agent)

Ralph picks up open GitHub issues labeled for implementation, creates a branch
(`issue-N`), adds the skill folder with the proposed `SKILL.md`, opens a PR with
`Closes #N`, and squash-merges. Configuration lives in `ralph.config.sh`. Trigger
manually with `/ralph`.

### Keeping documentation in sync

When adding, removing, or renaming a skill you **must** update both:

1. **README.md** — the skill table under the appropriate category section
2. **AGENTS.md** — the skills catalog table and the directory tree

Do this in the same PR as the skill change. A skill that exists on disk but is
missing from these files is invisible to users browsing the repo or the AgentHub
catalog. A stale entry pointing to a deleted skill is worse — it breaks install
commands.

### No build, no tests

There is no `package.json`, no CI pipeline, no linter. Validation is manual:
check frontmatter format, folder naming, and instruction body completeness before
merging.
