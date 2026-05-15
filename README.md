# Lucas's Claude Code Skills

A personal collection of Claude Code skills I use across projects. Each skill
is a self-contained folder with a `SKILL.md` and (optionally) a few auxiliary
markdown files. The collection is shared via the catalog at
[lucasfe.com](https://lucasfe.com) and the AgentHub UI, and installed into any
project on demand.

## Install a skill into your project

Skills live at `~/.claude/skills/<name>/` for Claude Code to discover. Use
[degit](https://github.com/Rich-Harris/degit) to copy a single skill folder
without git history baggage:

```bash
npx degit lucasfe/skills/<category>/<skill-name> ~/.claude/skills/<skill-name>
```

For example, to install the `grill-me` skill:

```bash
npx degit lucasfe/skills/development/grill-me ~/.claude/skills/grill-me
```

Restart Claude Code (or open a new session) and the skill will appear when
relevant — invoke it explicitly with `/<skill-name>`.

## What's in each skill

```
<category>/<skill-name>/
├── SKILL.md           # required: YAML frontmatter (name, description) + instructions
└── *.md               # optional: longer references (patterns, principles, examples)
```

The frontmatter is parsed by Claude Code to decide when to surface the skill.
The instructions are inlined into Claude Code's context when the skill is
invoked.

## Available skills

### Development

| Skill | What it does |
|---|---|
| `grill-me` | Interview the user relentlessly about a plan or design until reaching shared understanding |
| `improve-codebase-architecture` | Find deepening opportunities in a codebase, informed by domain language and ADRs |
| `request-refactor-plan` | Produce a detailed refactor plan with tiny commits via user interview, then file it as an issue |
| `supabase-postgres-best-practices` | Postgres performance optimization and best practices from Supabase |
| `tdd` | Drive a feature or fix through a strict red-green-refactor TDD loop |

### Project management

| Skill | What it does |
|---|---|
| `sec-consult` | Generate a security consult ticket from a tech spec, PRD, or project reference |
| `spec-to-jira` | Turn a PRD + tech spec into an idempotent Jira hierarchy (Epics, Stories, Sub-tasks) with gap detection and dependency inference |
| `to-issues` | Break a plan or PRD into vertical-slice GitHub issues |
| `to-prd` | Synthesize the current conversation into a PRD GitHub issue |
| `triage-issue` | Investigate a bug, find root cause, and file a GitHub issue with a TDD-based fix plan |

### Meta

| Skill | What it does |
|---|---|
| `find-skills` | Discover and install Claude Code skills relevant to the user's request |
| `opensquad` | Multi-agent orchestration framework for creating and running AI squads |

## Adding a new skill

There are two paths:

### Via the AgentHub Skill Creator agent (easy)

1. Open [agenthub.lucasfe.com](https://agenthub.lucasfe.com) and pick the
   "Skill Creator" agent from the catalog.
2. Describe the skill — what it does, when it should fire, the instruction body.
3. The agent drafts a structured issue in this repo containing a complete
   `SKILL.md` ready to paste. Approve.
4. The implementer (Ralph or human) creates the folder inside the right category
   with the proposed `SKILL.md` content, opens a PR, merges to `main`. Done.

### By hand (also fine)

1. Pick the category folder the skill belongs to (`development/`,
   `project-management/`, `meta/`). If none fits, create a new category folder
   at the repo root using the same naming convention (kebab-case, lowercase,
   ASCII).
2. Inside that category, create a folder with a kebab-case name.
3. Add `SKILL.md` with YAML frontmatter (`name`, `description`) and the
   instructions.
4. Optionally add auxiliary `.md` files referenced from `SKILL.md`.
5. Update the relevant table above (or add a new subsection if you introduced a
   new category).
6. Commit, push, merge.

## Structure invariants

These are enforced by convention (no automated linter yet). Keep them stable so
the AgentHub catalog renders cleanly and `degit` installs land in the right
place:

- Skills live inside a category folder, never at the repo root.
- Folder names are **kebab-case**, lowercase, ASCII-only — same convention
  applies to category folders and skill folders.
- Folder name matches `name` in `SKILL.md` frontmatter exactly.
- `SKILL.md` is uppercase. The Claude Code loader is case-sensitive.
- Frontmatter has at least `name` and `description`. Other keys are allowed but
  optional. There is no `category` field — the path is the single source of
  truth.
- Instructions in the markdown body are non-empty (no placeholders).

## License

MIT. Feel free to copy, fork, remix.
