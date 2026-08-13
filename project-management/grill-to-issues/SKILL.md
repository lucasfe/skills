---
name: grill-to-issues
description: Run a full planning pipeline — relentlessly grill the user on a plan, synthesize a PRD, break it into vertical-slice issues, then let the user choose where to output (GitHub, a Ralph filesystem task tree, or markdown files). Use when the user wants to take a rough idea all the way to ready-to-work tickets in one flow.
---

# Grill to Issues

Chain three planning phases into one flow: **grill → PRD → slices → output**.
Runs grilling and synthesis straight through, then pauses ONCE at the end to
review the full package and pick an output target. This skill embeds condensed
versions of the `grill-me`, `to-prd`, and `to-issues` skills — do not depend on
invoking those skills separately.

**Critical rule:** produce NO side effects (no GitHub issues, no files) until
the user has reviewed everything and explicitly chosen an output target in
Phase 4. The PRD and slice breakdown live in the conversation until then.

## Phase 1 — Grill

Interview the user relentlessly about every aspect of the plan until you reach
shared understanding. Walk down each branch of the design tree, resolving
dependencies between decisions one-by-one.

- Ask questions **one at a time**.
- For each question, provide your **recommended answer**.
- If a question can be answered by exploring the codebase, explore instead of
  asking.

Continue until the decision tree is resolved. Then move directly to Phase 2 —
do NOT stop for approval.

## Phase 2 — Synthesize the PRD

Synthesize the grilling + codebase understanding into a PRD. Do NOT interview
further and do NOT write it anywhere yet. First sketch the major modules to
build/modify, favoring **deep modules** (lots of functionality behind a simple,
stable, testable interface). Then render the PRD with this template:

<prd-template>
## Problem Statement
The problem the user faces, from the user's perspective.

## Solution
The solution, from the user's perspective.

## User Stories
A LONG, numbered list, each: `As an <actor>, I want <feature>, so that <benefit>`.
Cover all aspects of the feature.

## Implementation Decisions
Modules to build/modify, their interfaces, technical clarifications,
architectural decisions, schema changes, API contracts, specific interactions.
Do NOT include file paths or code snippets — they go stale.

## Testing Decisions
What makes a good test (test external behavior, not implementation details),
which modules get tested, and prior art for those tests in the codebase.

## Out of Scope
What is explicitly not covered.

## Further Notes
Anything else worth recording.
</prd-template>

Move directly to Phase 3.

## Phase 3 — Break into vertical-slice issues

Break the PRD into **tracer-bullet** issues. Each is a thin vertical slice
cutting through ALL layers end-to-end (schema, API, UI, tests) — never a
horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer.
- A completed slice is demoable or verifiable on its own.
- Prefer many thin slices over few thick ones.
- Classify each slice as **HITL** (needs human interaction — decisions, design
  review) or **AFK** (implementable and mergeable autonomously). Prefer AFK.
</vertical-slice-rules>

For each slice capture: **Title**, **Type** (HITL/AFK), **Blocked by** (which
slices must finish first), and **User stories covered**.

## Phase 4 — Review and output (THE ONLY CHECKPOINT)

Present the PRD and the numbered slice breakdown together. Ask the user:

- Does the granularity feel right? Are dependencies correct? Any splits/merges?
- Are the HITL/AFK classifications right?

Iterate until approved. Then ask **where to output** — offer these three
targets and follow the matching section. Write NOTHING until they choose.

### Output: GitHub issues

1. Optionally create the PRD as a parent issue with `gh issue create`; note its
   number for `Parent`/`Blocked by` references.
2. Create slices in dependency order (blockers first) so you can reference real
   issue numbers. Use the body template below.

<issue-template>
## Parent
#<parent-issue-number> (omit if no PRD parent issue)

## What to build
Concise description of this vertical slice — end-to-end behavior, not
layer-by-layer implementation.

## Acceptance criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Blocked by
- Blocked by #<issue-number>  (or "None - can start immediately")
</issue-template>

Do NOT close or modify any parent issue.

### Output: Ralph filesystem tasks

Writes tasks into a project's `.ralph/tasks/` tree for the local Ralph loop
(the loop lives at `~/repos/ralph`; tasks live in the *target project's* tree).

1. **Ask for the target repo root** (default: current working directory). Tasks
   go under `<root>/.ralph/tasks/`.
2. **Map slice type to lane:** AFK → `.ralph/tasks/afk/todo/`,
   HITL → `.ralph/tasks/hitl/todo/`. Ralph auto-picks only from `afk/todo/`;
   `hitl/todo/` is a human parking lot.
3. **Numbering:** the leading integer is the task's stable identity. The next
   number is `max(N) + 1` scanned across ALL directories in BOTH lanes so a
   number is never reused. Compute it:
   ```bash
   find <root>/.ralph/tasks -name '*.md' 2>/dev/null \
     | grep -oE '/[0-9]+-' | tr -dc '0-9\n' | sort -n | tail -1
   ```
   (empty output → start at 1). Assign numbers in dependency order.
4. **One file per slice**, named `NNN-short-slug.md`, with this exact format:
   ```markdown
   ---
   title: <slice title>
   labels: <comma list, e.g. bug, auth>   # optional
   ---

   <body: what to build, acceptance criteria, and a "Blocked by task NNN"
   note if it depends on another slice — folder mode has no native dep links,
   so ordering is by number>
   ```
   `title` and body map 1:1 onto a GitHub issue's title/body.
5. **Do not `git add` task files** — the `.ralph/` tree is gitignored; use plain
   `mkdir -p` and file writes. Create blocker slices with lower numbers so they
   are picked first.
6. **PRD:** the PRD is not a Ralph task. Write it to
   `<root>/.ralph/prd-<slug>.md` (also gitignored) and reference it from each
   task body for shared context.

### Output: Plain markdown files

1. Ask for a target directory.
2. Write the PRD to `<dir>/prd-<slug>.md`.
3. Write one file per slice, `NN-<slug>.md`, using the GitHub issue body
   template above (with a `# Title` heading). Order files by dependency.

## Notes

- Grilling (Phase 1) is inherently interactive; the "single checkpoint" refers
  to phases 2–4 running without extra approval gates until the Phase 4 review.
- If the user passes an existing plan/PRD as input, you may compress Phase 1 to
  confirming gaps rather than grilling from scratch.
