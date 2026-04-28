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
npx degit lucasfe/skills/<skill-name> ~/.claude/skills/<skill-name>
```

For example, to install the `grill-me` skill:

```bash
npx degit lucasfe/skills/grill-me ~/.claude/skills/grill-me
```

Restart Claude Code (or open a new session) and the skill will appear when
relevant — invoke it explicitly with `/<skill-name>`.

## What's in each skill

```
<skill-name>/
├── SKILL.md           # required: YAML frontmatter (name, description) + instructions
└── *.md               # optional: longer references (patterns, principles, examples)
```

The frontmatter is parsed by Claude Code to decide when to surface the skill.
The instructions are inlined into Claude Code's context when the skill is
invoked.

## Available skills

| Skill | What it does |
|---|---|
| `grill-me` | Interview the user relentlessly about a plan or design until reaching shared understanding |
| `to-prd` | Synthesize the current conversation into a PRD GitHub issue |
| `to-issues` | Break a plan or PRD into vertical-slice GitHub issues |
| `triage-issue` | Investigate a bug, find root cause, and file a GitHub issue with a TDD-based fix plan |
| `tdd` | Drive a feature or fix through a strict red-green-refactor TDD loop |
| `request-refactor-plan` | Produce a detailed refactor plan with tiny commits via user interview, then file it as an issue |
| `improve-codebase-architecture` | Find deepening opportunities in a codebase, informed by domain language and ADRs |
| `find-skills` | Discover and install Claude Code skills relevant to the user's request |
| `opensquad` | Multi-agent orchestration framework for creating and running AI squads |
| `supabase-postgres-best-practices` | Postgres performance optimization and best practices from Supabase |

## Adding a new skill

There are two paths:

### Via the AgentHub Skill Creator agent (easy)

1. Open [agenthub.lucasfe.com](https://agenthub.lucasfe.com) and pick the
   "Skill Creator" agent from the catalog.
2. Describe the skill — what it does, when it should fire, the instruction body.
3. The agent drafts a structured issue in this repo containing a complete
   `SKILL.md` ready to paste. Approve.
4. The implementer (Ralph or human) creates the folder with the proposed
   `SKILL.md` content, opens a PR, merges to `main`. Done.

### By hand (also fine)

1. Create a folder at the repo root with a kebab-case name.
2. Add `SKILL.md` with YAML frontmatter (`name`, `description`) and the
   instructions.
3. Optionally add auxiliary `.md` files referenced from `SKILL.md`.
4. Update the table above.
5. Commit, push, merge.

## Structure invariants

These are enforced by convention (no automated linter yet). Keep them stable so
the AgentHub catalog renders cleanly and `degit` installs land in the right
place:

- Folder names are **kebab-case**, lowercase, ASCII-only.
- Folder name matches `name` in `SKILL.md` frontmatter exactly.
- `SKILL.md` is uppercase. The Claude Code loader is case-sensitive.
- Frontmatter has at least `name` and `description`. Other keys are allowed but
  optional.
- Instructions in the markdown body are non-empty (no placeholders).

## License

MIT. Feel free to copy, fork, remix.
