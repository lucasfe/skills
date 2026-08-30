---
name: bug-to-issues
description: Take an existing bug report — a GitHub issue, Jira ticket, or local file — and turn it into ready-to-work fix issues. Resolves the task source from ralph.config.sh, investigates root cause in the codebase, asks only the questions the code cannot answer, splits the fix into vertical slices, then pauses once to confirm source, slices, and output target. Use when the user wants to work an already-filed bug into actionable tickets.
---

# Bug to Issues

Chain four phases into one flow: **locate → diagnose → clarify → slices → output**.

This is the bug counterpart to `grill-to-issues`. Differences from that skill:
the input is an **already-filed bug**, not a rough idea; there is **no PRD**; and
clarification is **short and objective** (a handful of targeted questions, not an
open-ended interview) because the report already fixes most of the scope.

**Critical rule:** produce NO side effects (no issues, no files, no ticket edits)
until the user has reviewed everything and explicitly chosen an output target in
Phase 4. Everything lives in the conversation until then.

## Phase 0 — Locate the bug report

Resolve where the report lives from the project's Ralph configuration. Do NOT ask
the user — infer, and confirm the inference at the Phase 4 checkpoint.

1. Read `TASK_SOURCE` from `./ralph.config.sh` (fall back to the repo root):
   ```bash
   grep -E '^(TASK_SOURCE|JIRA_JQL|JIRA_DONE_STATUS)=' ralph.config.sh
   ```
   Valid values are `github`, `folder`, `jira`; unset or unrecognized means
   `github`. If there is no `ralph.config.sh`, assume `github`.
2. Fetch the report according to the resolved source and whatever identifier the
   user passed (issue number, ticket key, path):
   - `github` → `gh issue view <number> --comments`
   - `jira` → fetch the ticket by key (`acli` / the `jira` skill); if no key was
     given, list candidates with the repo's `JIRA_JQL` and pick the one the user
     named
   - `folder` → read the matching file under `.ralph/tasks/**` (or the path given)
3. If the identifier is ambiguous or nothing resolves, ask ONE question naming the
   concrete candidates you found. Otherwise proceed silently.

Record the resolved source and identifier — Phase 4 confirms both.

## Phase 1 — Diagnose

Dispatch an Explore subagent (`Agent` with `subagent_type=Explore`) to trace the
bug in the codebase. Find:

- **Where** it manifests (entry point, API response, UI surface)
- **What** code path is involved
- **Why** it fails — the root cause, not the symptom
- **What** already covers it (existing tests, similar patterns that work)

Also check `git log` on the implicated area to tell a regression from a
never-worked path.

Describe findings as modules, behaviors, and contracts. Do NOT anchor them to file
paths or line numbers — those go stale and make the issue useless after a refactor.

## Phase 2 — Clarify (objective, capped)

Ask ONLY what neither the report nor the code answered. Hard rules:

- **At most 3 questions**, all in **ONE AskUserQuestion call**, single submission.
- Skip this phase entirely when nothing material is unresolved. Say so and move on.
- Every question is multiple-choice with a **recommended option first**.
- Never ask something the Explore pass could have answered — go look instead.

Typical survivors: intended behavior when the report only states the broken one,
blast radius (fix just this path, or every caller), and whether a workaround
already shipped that the fix must undo.

Then move directly to Phase 3 — no approval gate.

## Phase 3 — Split the fix into vertical slices

Break the fix into **tracer-bullet** slices. Each cuts through ALL layers
end-to-end (test, fix, surface), never a horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer.
- A completed slice is demoable or verifiable on its own.
- Prefer many thin slices over few thick ones.
- The FIRST slice reproduces the bug as a failing test and fixes that exact path.
- Classify each as **HITL** (needs a human decision or design review) or **AFK**
  (implementable and mergeable autonomously). Prefer AFK.
- A one-line bug is one slice. Do not manufacture slices to look thorough.
</vertical-slice-rules>

For each slice capture: **Title**, **Type** (HITL/AFK), **Blocked by**, and which
part of the reported symptom it removes.

## Phase 4 — Confirm and output (THE ONLY CHECKPOINT)

Present the root-cause summary and the numbered slice breakdown as text, then ask
all three checkpoint questions with the **AskUserQuestion tool** — never as
free-form prose. All three go in **ONE call**, in this order, so the user answers
together and submits once.

<question-format-rules>
- Always AskUserQuestion — all checkpoint questions in a single call, so the user
  picks by number and submits every answer at once.
- Options must be **concrete and tailored to this run** — name the actual slices
  ("Merge 1 and 2", "Split slice 2 by surface", "Drop slice 3"), never generic
  filler like "Looks good" / "Change something".
- Put the recommended option first and lead with it.
- Attach a `preview` to every option that produces a concrete artifact: the exact
  commands, paths, or resulting slice list that choosing it yields. Previews
  render as monospace, so lay them out as a small ASCII block — one line per
  slice, with issue/task number, short title, and HITL/AFK tag aligned in columns.
- Keep `header` ≤ 12 chars (e.g. "Source", "Review", "Output").
- Single-select (`multiSelect: false`) — previews only render for single-select,
  and max 4 options, so keep the list tight.
- EVERY option gets a preview. "No preview available" on a checkpoint question
  means the question was built wrong.
- The user can always pick "Other" to steer freely — do not add your own
  escape-hatch option.
</question-format-rules>

**Question 1 — source confirmation.** Header "Source". State the source and
identifier you resolved in Phase 0 and where it came from. First option confirms;
the rest are the other plausible readings (a different ticket, a different source).
Preview shows the resolved identity and the fetch command used:

```
TASK_SOURCE=jira  (ralph.config.sh:94)
   → PROJ-482  "login redirect drops ?next"
   fetched: acli jira workitem view PROJ-482
```

**Question 2 — slice review.** Header "Review". Ask whether the breakdown looks
right (granularity, dependencies, HITL/AFK calls). First option approves as-is;
the rest are the 1–3 most plausible concrete edits you would consider. Each
option's preview is **the resulting slice list after that edit** — renumbered, so
the user sees the outcome rather than a description of it:

```
1 — failing test + preserve ?next through redirect   AFK
2 — audit other redirect callers                     HITL
```

**Question 3 — output target.** Header "Output". Phrase it around the real counts,
e.g. "Where should I write the three fix slices?" Options are the three targets
below, each previewing exactly what it will do:

`1. GitHub issues`

```
gh issue create × 3
   → #N   failing test + preserve ?next   AFK
   → #N+1 regression test for oauth path  AFK
   → #N+2 audit other redirect callers    HITL
```

`2. Ralph filesystem tasks`

```
.ralph/tasks/afk/todo/
   → 012-preserve-next-through-redirect.md   AFK
   → 013-oauth-path-regression-test.md       AFK
.ralph/tasks/hitl/todo/
   → 014-audit-other-redirect-callers.md     HITL
```

`3. Plain markdown files`

```
<dir>/01-preserve-next-through-redirect.md   AFK
<dir>/02-oauth-path-regression-test.md       AFK
<dir>/03-audit-other-redirect-callers.md     HITL
```

Use the real numbers, slugs, and paths for the run at hand — for Ralph, the task
numbers you computed in step 3 of that section, not placeholders.

**Handling the answers together.** Q3's previews are built from the breakdown as
it stands, so a Q2 edit shifts the numbers shown there. That is fine — the target
choice does not depend on granularity. On submit:

- Q1 corrected → re-fetch from the right source, redo phases 1–3, re-ask.
- Q2 approved as-is → go straight to the chosen output section and write.
- Q2 chose an edit → apply it, then re-ask **only Question 2** (single question,
  same format) against the revised list. Keep the Q1 and Q3 answers; do not
  re-ask them. Repeat until approved, then write.

Never write to the target before Question 2 is approved, even though its answer
arrived in the same submission.

### Output: GitHub issues

Create slices in dependency order (blockers first) so you can reference real issue
numbers. Use the body template below.

<issue-template>
## Parent
#<bug-issue-number> (the source report, if it is a GitHub issue — omit otherwise)

## Problem
What happens (actual), what should happen (expected), and how to reproduce.

## Root cause
The code path involved and why it fails. Modules, behaviors, and contracts — no
file paths or line numbers.

## What to fix
The end-to-end behavior this slice changes, not a layer-by-layer implementation.

## Acceptance criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Regression test covers the reported path
- [ ] Existing tests still pass

## Blocked by
- Blocked by #<issue-number>  (or "None - can start immediately")
</issue-template>

Do NOT close, label, or otherwise modify the source report.

### Output: Ralph filesystem tasks

Writes tasks into the project's `.ralph/tasks/` tree for the local Ralph loop.

1. **Ask for the target repo root** (default: current working directory).
2. **Map slice type to lane:** AFK → `.ralph/tasks/afk/todo/`, HITL →
   `.ralph/tasks/hitl/todo/`. Ralph auto-picks only from `afk/todo/`.
3. **Numbering:** the leading integer is the task's stable identity. The next
   number is `max(N) + 1` across ALL directories in BOTH lanes:
   ```bash
   find <root>/.ralph/tasks -name '*.md' 2>/dev/null \
     | grep -oE '/[0-9]+-' | tr -dc '0-9\n' | sort -n | tail -1
   ```
   (empty output → start at 1). Assign numbers in dependency order.
4. **One file per slice**, named `NNN-short-slug.md`:
   ```markdown
   ---
   title: <slice title>
   labels: bug
   ---

   <body: the issue template above, minus Parent, plus a "Blocked by task NNN"
   note when it depends on another slice — folder mode has no native dep links>
   ```
5. **Do not `git add` task files** — `.ralph/` is gitignored; use `mkdir -p` and
   plain file writes. Give blocker slices lower numbers so they are picked first.
6. Reference the source report (issue number, ticket key, or path) in every task
   body so the agent can read the original.

### Output: Plain markdown files

1. Ask for a target directory.
2. Write one file per slice, `NN-<slug>.md`, using the issue body template above
   (with a `# Title` heading). Order files by dependency.

## Notes

- Phases 0–3 run straight through; the Phase 4 checkpoint is the only gate.
- If the user pastes a bug report inline instead of pointing at a tracked one,
  skip Phase 0's fetch and confirm the origin as "pasted in conversation" in
  Question 1.
- When the diagnosis contradicts the report (the reported cause is wrong, or the
  behavior is intended), say so in the Phase 4 presentation before the questions —
  the user may want to close the report instead of splitting a fix.
</content>
