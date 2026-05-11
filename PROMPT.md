# Project context for Ralph

## Stack

This is a **content repository**, not a code project. There is no package.json, no
build step, no test suite, no lint config. Each skill is a folder inside a
category folder (`development/`, `project-management/`, `meta/`, …) at the repo
root, with a `SKILL.md` file (YAML frontmatter `name`, `description`, plus a
markdown instruction body) and optional auxiliary `.md` files.

Layout:

```
/
├── README.md                         # repo overview, install instructions
├── <category>/
│   └── <skill-name>/
│       ├── SKILL.md                  # required: frontmatter + instructions
│       └── <auxiliary>.md            # optional: longer references the skill points to
└── ...
```

`<category>` and `<skill-name>` are both **kebab-case**, lowercase, ASCII-only.
`<skill-name>` matches the `name` in `SKILL.md`'s frontmatter exactly. There is
no `category` field in frontmatter — the path is the single source of truth.

## Validation (no test suite — content checks only)

Since `TEST_CMD` and `LINT_CMD` are intentionally empty in `ralph.config.sh`, do
the following manual sanity checks before opening a PR:

1. The new (or modified) `SKILL.md` file starts with a valid YAML frontmatter
   block: `---` line, `name: <kebab-case>` line, `description: <one-line>` line,
   `---` close line, blank line, then the markdown body.
2. The folder name matches the `name` in frontmatter exactly.
3. The skill folder lives inside a category folder (e.g. `development/<skill>/`,
   not `<skill>/` at the repo root). If the issue does not pin a category and
   none of the existing ones fits, create a new category folder using the same
   kebab-case / lowercase / ASCII convention.
4. The instruction body is at least a few sentences long — empty / placeholder
   skills are not acceptable.
5. If the issue requests auxiliary files, they exist and are referenced from
   `SKILL.md`.

If a check fails, fix it before opening the PR. If you cannot satisfy the issue
(e.g., contradictory acceptance criteria), mark it failed.

## Workflow

- **Single-branch flow:** PRs target `main` (default branch). `Closes #N` in the
  PR body **does** trigger auto-close on merge.
- Branches: `issue-N` off `main`, push, open PR with
  `gh pr create --base main --head issue-N --title "..." --body "Closes #N"`.
- Merge: `gh pr merge <pr> --auto --squash --delete-branch`. Wait for the
  auto-merge.
- After merge, the issue closes automatically — verify via
  `gh issue view N --json state` after the PR shows `MERGED`.

## MCPs

None configured for this repo. If you need GitHub data, use `gh`. There is no
database, no edge functions, no Supabase here — it is purely markdown content.

## Sub-agents / slash commands

No sub-agent team. Work end-to-end yourself. Most issues are extremely small —
"create folder X with this SKILL.md content" — and finish in a single iteration.

## Extra restrictions

- **Never edit** `ralph.config.sh`, `PROMPT.md`, or anything under `.claude/` —
  those are infra/config, out of scope for any skill issue.
- **Never run** any `npm`, `pip`, `cargo`, etc. installs. There is no code stack
  to install.
- **Never push** directly to `main`. Always via PR.
- Folder names must be lowercase, kebab-case, ASCII-only.
- `SKILL.md` filenames must be exactly `SKILL.md` (uppercase). The Claude Code
  loader is case-sensitive.
- Issues in this repo always describe a single skill bootstrap or update. If an
  issue asks for code changes outside the skill folders or `README.md`, mark it
  failed.

## Reference

- Issues that come in via the AgentHub "Skill Creator" agent will follow a fixed
  three-section structure: `## Proposed SKILL.md`, `## Notes`, `## Acceptance
  criteria`. The proposed `SKILL.md` block is the source of truth — paste it
  verbatim into the new file (do not "improve" it). The `## Notes` section is
  context only, not for the skill body. The acceptance criteria is the gate for
  the PR.
- The repo is the source of truth consumed by AgentHub's `/skills` page (live
  GitHub Contents API) and by
  `npx degit lucasfe/skills/<category>/<name> ~/.claude/skills/<name>` installs
  in any project.
