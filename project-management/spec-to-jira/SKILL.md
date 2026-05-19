---
name: spec-to-jira
description: Turn a PRD and a stack-organized tech spec into a Jira hierarchy — one Epic per milestone, one Story per user story, one Sub-task per stack — with a gap-aware coverage matrix, inferred sub-task dependencies, and idempotent re-runs. Use when user wants to break down a PRD into Jira issues, plan a multi-stack project, or sync a plan with Jira.
---

# Spec to Jira

Take a PRD and a stack-organized tech spec — each accepted as a local `.md` path, a Google Docs URL, a Jira issue key, or content pasted directly into chat — detect milestone boundaries in the PRD, derive a list of user stories **per milestone**, perform an inverted scan of the tech spec to build a **gap-aware** coverage matrix (user story × stack) **per milestone**, infer blocking dependencies between Sub-tasks from the tech spec prose, write a `plano-<slug>.md` plan file in the current working directory structured as `## Epic N — <name>` sections, and **pause** for the user to review and edit it. When the user types `ok`, the skill re-reads `plano-<slug>.md` from disk and treats it as the source of truth: it creates one Epic per milestone, one Story per user story under the right Epic, one Sub-task per covered (story × stack) cell, an extra `pm`-stack Sub-task for every gap the user accepted in the **Pendências PM** checklist, and Jira `blocks` / `blocked by` links between Sub-tasks for every dependency the user approved in the **Dependências** checklist — exactly as the file now describes them.

Key behaviors:

1. **Gaps** (`❓`): a coverage matrix cell that has no tech spec mention is marked `❓` (instead of being silently blank). Each `❓` cell shows up as a checkbox under a per-Epic **Pendências PM** subsection. During the review pause, the user resolves each `❓` either as a **real gap** (checks the box → Phase 2 creates a `pm`-stack Sub-task to track the missing PM/coordination work) or as **N/A** (deletes the line → Phase 2 blanks the matrix cell). Lines left undecided stay as `❓` and are flagged in the summary.
2. **Multi-stack split**: a story covered in multiple stacks always produces one Sub-task **per covered stack**, never a single multi-stack Sub-task. The skill emits proposed `blocks` links between those Sub-tasks (cloud → web → data → pm by default order from `team.yaml`) as checkboxes under the per-Epic **Dependências** subsection.
3. **Inferred sub-task ↔ sub-task dependencies**: the skill scans the tech spec for blocking phrases ("after X", "depends on X", "requires Y exists", "once X is in place", "blocked by", "before Y", "must precede", and similar) and proposes additional sub-task ↔ sub-task dependencies as checkboxes in the same **Dependências** subsection. The user approves, edits, adds, or deletes proposals before approval. On `ok`, every checked dependency becomes a Jira `blocks` / `blocked by` link between the corresponding Sub-tasks.
4. **Idempotent re-runs**: Phase 2 writes the real Jira keys back into `plano-<slug>.md` next to every Epic heading, every Story line, every Sub-task line, and every approved Dependências line it creates. On any subsequent invocation, the skill re-reads the file (still the source of truth) and decides per item:
   - **No key** → create the item in Jira.
   - **Key, mutable fields match Jira** → skip (zero Jira writes).
   - **Key, mutable fields diverge from Jira** → call `editJiraIssue` (or the equivalent link-update path for dependencies) exactly once to bring Jira back in sync with the file. Mutable fields are: summary / title, description, labels, parent / Epic Link, and link relations.
   - **Key exists in Jira but the line was removed from the `.md`** → list as an **orphan** at the end of the run for manual review. The skill never deletes Jira data automatically.
   The skill never sets, updates, or clears `status` or `assignee` on any existing Jira issue, ever. Re-running on an unchanged `.md` therefore produces zero Jira writes (true idempotency); re-running after editing a single title produces exactly one update; re-running after deleting a line surfaces one orphan and creates / updates nothing else.
5. **Histórico de execuções**: every run appends a single timestamped entry to a `## Histórico de execuções` section at the bottom of the `.md`. The entry carries the run kind (`first-run` or `re-run`) and the counts of **created / updated / orphaned** items, grouped by issue type (Epic, Story, Sub-task, Link). The exact same counts are echoed in the chat summary printed at the end of the run, so the chat output and the file's log always match.

Sub-task ↔ sub-task links within a single Epic **are** in scope and ARE created (and idempotently re-synced) in Jira via the Atlassian MCP. Creation of Jira "blocks" links between **Epics** and cross-Epic Sub-task ↔ Sub-task links are out of scope — the per-Epic `Depends on:` line is informational.

## Invocation

On every invocation, **before reading any source input**, the skill scans the current working directory for files matching `plano-*.md`. The presence of such a file is the signal that a prior run paused for review.

- **No matches**: proceed directly to Phase 1 (input resolution → matrix → gap detection → dependency inference → write plan).
- **One or more matches**: list them in chat (filename, `Status:` value, and last-modified time) and ask the user:
  > Found <N> existing plan file(s): <list>. Continue with one of these, or create a new plan?
  - If the user picks one of the existing files, **skip Phase 1** and jump straight to Phase 2 with that file as the input. Do not re-read the PRD or tech spec — the plan file is now the source of truth. Do not re-prompt for sources. Phase 2 inspects the file's `Status:` value and branches:
    - `Status: pending review` → the **first-run flow**. Phase 2 creates Jira issues for every line that does not yet carry a real key, writes the real keys back into the file, flips `Status:` to `applied`, and appends an entry to `## Histórico de execuções` with `kind: first-run` plus the created counts.
    - `Status: applied` → the **idempotent re-sync flow**. Phase 2 reads each line, classifies it as **skip / update / create / orphan**, performs only the necessary Jira writes (zero writes when nothing diverged), keeps `Status:` at `applied`, and appends an entry to `## Histórico de execuções` with `kind: re-run` plus the created / updated / orphaned counts. There is no second confirmation prompt for re-syncing an `applied` plan — the operation is by design safe to repeat.
    - There is no separate `pending re-run` status — Phase 2 itself decides per item what to do based on key presence and per-field divergence with Jira.
  - If the user picks "new" (or "create new", or similar), proceed with Phase 1. Do not delete or rename the existing plan files — leave them in place; Phase 1 picks a non-colliding slug (see Phase 1, step 6).
  - If the user's response is ambiguous, ask again. Do not auto-resume without explicit confirmation, even when only one plan file is present.

## Inputs

The skill accepts the PRD and the tech spec independently — each can be supplied in any of the supported formats, and the two formats do not need to match. Inputs are consumed in Phase 1 only; Phase 2 reads the plan file alone and never re-touches the original PRD or tech spec.

- **PRD source** (required in Phase 1): the **first** positional argument: `/spec-to-jira <prd-source> <tech-spec-source>`. May be a local `.md` path, a Google Docs URL (any `https://docs.google.com/document/...` link), or a Jira issue key (e.g., `PROJ-123`). If the argument is omitted, the skill enters interactive mode for the PRD (see "Input resolution" below).
- **Tech spec source** (required in Phase 1): the **second** positional argument. Same supported formats as the PRD. If omitted, the skill enters interactive mode for the tech spec. If only one argument is given and its role is ambiguous, ask the user to clarify which is the PRD and which is the tech spec.
- **No-argument interactive mode**: invoking `/spec-to-jira` with **no arguments** (and the user has chosen "new" at the invocation check above, or no plan file existed) enters interactive mode for both sources. For each source, ask the user to either paste content, give a local `.md` path, give a Google Docs URL, or give a Jira issue key. PRD comes first, tech spec second.
- **Jira target**: read from `./.spec-to-jira.yaml` in the current working directory. If the file is missing, run the auto-detect flow (see "Configuration" below) and write it. Once present, the skill never asks the user for the project key, cloud ID, or issue type names again in this cwd.

## Configuration

The skill reads two YAML files: one **global** (stack definitions, shared across all projects) and one **per-cwd** (Jira target, specific to this repository).

### Global: `~/.config/spec-to-jira/team.yaml`

Stack definitions used by the coverage matrix, Sub-task creation, and the default cross-stack dependency order. Read once per run. If the file is missing, the skill writes a default file with `cloud`, `web`, `data`, and the virtual `pm` stack, then continues using that default.

Schema:

```yaml
stacks:
  - name: cloud
    label: stack:cloud
    title_prefix: "[cloud]"
    component: Cloud        # optional Jira component name
  - name: web
    label: stack:web
    title_prefix: "[web]"
    component: Web
  - name: data
    label: stack:data
    title_prefix: "[data]"
    component: Data
  - name: pm                # virtual stack — project-management work, not a real engineering stack
    label: stack:pm
    title_prefix: "[pm]"
  - name: qa                # virtual stack — QA test plans, one per story
    label: stack:qa
    title_prefix: "[qa]"
    assignee: ""            # display name of the default QA assignee (looked up via Jira user search)

# Per-stack default assignees — when set, Phase 2 assigns newly created
# Sub-tasks (and QA test plan sub-tasks) to the named person on creation.
# Stories are assigned to the `story_assignee` if set.
# Assignees are ONLY set on creation, never on updates or re-runs.
assignees:
  story_assignee: ""        # display name for all new Stories (optional)
  stack_overrides:          # per-stack assignee overrides (optional)
    # cloud: "Alice Smith"
    # web: "Bob Jones"
    # qa: "Lara Monteiro"
```

Notes on `team.yaml`:

- The matrix columns and the Sub-task iteration follow the order in which stacks appear in `team.yaml`. Reordering the list reorders the matrix.
- The same order also drives the **default multi-stack dependency direction**: when a story is covered in two or more stacks, the skill proposes `blocks` links from the earlier stack to the later stack (e.g., `cloud → web → data → pm` with the default ordering). The user can flip, edit, or delete those proposals during review.
- The `pm` stack is **virtual**: it represents project-management work (rollout coordination, comms, milestone tracking) rather than an engineering stack. It uses the same Sub-task mechanic — the inverted scan can map "pm" coverage to tech-spec sections describing rollout / comms / coordination work, and unresolved gaps the user marks as real gaps in **Pendências PM** are tracked as additional `pm` Sub-tasks in Phase 2.
- The `qa` stack is **virtual**: it represents QA test plan work. When present in `team.yaml`, the skill **automatically generates one QA test plan Sub-task per Story** in Phase 1 (step 5c). Each QA Sub-task summarizes the key test scenarios derived from the Story's acceptance criteria and the tech spec's test plan section. QA Sub-tasks are always generated — they do not depend on the coverage matrix scan. If `team.yaml` has an `assignee` field on the `qa` stack entry, Phase 2 assigns newly created QA Sub-tasks to that person. The `qa` stack does NOT appear as a column in the coverage matrix — it is always present (one per Story) and therefore not gap-tracked.
- `component` is **optional**. When set, the skill attaches the corresponding Jira component to each Sub-task in that stack (subject to the project's component configuration). When absent, no component is attached.
- Users can add, remove, or rename stacks in `team.yaml` between runs. The next run picks up the new vocabulary automatically — the matrix layout follows.

### Per-cwd: `./.spec-to-jira.yaml`

Jira target configuration. **Committable** — it points at the team's Jira target, not at credentials, so it is safe to check into version control alongside the PRDs and tech specs.

Schema:

```yaml
cloud_id: 11111111-2222-3333-4444-555555555555    # UUID returned by getAccessibleAtlassianResources
cloud_url: https://<your-domain>.atlassian.net    # optional, for human-readable references
project_key: PROJ
issue_types:
  epic: Epic
  story: Story
  sub_task: Sub-task
link_types:
  blocks: Blocks
  blocked_by: is blocked by
```

Notes on `.spec-to-jira.yaml`:

- `issue_types` are project-specific. The Jira project may use the standard `Epic` / `Story` / `Sub-task` names or localized / customized variants — the auto-detect flow discovers and persists whatever the project actually uses.
- `link_types` carry the inward / outward names for the "blocks" relationship. **The skill uses them**: every approved Sub-task ↔ Sub-task dependency in the **Dependências** checklist is created as a Jira link with the outward name `blocks` and the inward name `blocked_by`. If either field is empty (the project has no "blocks" link type), the skill skips link creation and warns the user in the summary; everything else still proceeds normally.
- `cloud_url` is optional but recommended — useful for human-readable summaries and direct links in `plano-<slug>.md`.
- Once the file exists, the skill reads it silently. Do not prompt the user for any value already in the file.

### Auto-detect (first run in a new cwd)

When `./.spec-to-jira.yaml` is missing, the skill runs the following sequence **before** reading the PRD or the tech spec:

1. Call `getAccessibleAtlassianResources` (Atlassian MCP) to list the cloud resources the user has access to. If exactly one is returned, use it without asking. Otherwise, show the user the list (id + name + url) and ask which to use.
2. Call `getVisibleJiraProjects` on the chosen cloud ID. Show the user the list (project key + name) and ask which to use.
3. Call `getJiraProjectIssueTypesMetadata` (falling back to `getJiraIssueTypeMetaWithFields` if needed) for the chosen project. Map the returned issue type names to the `epic`, `story`, and `sub_task` slots. If the project does not expose an Epic, a Story, or a Sub-task equivalent, stop and tell the user — this skill needs all three.
4. Call `getIssueLinkTypes` to fetch the project's link type names. Find the "blocks" relationship and record the outward name as `blocks` and the inward name as `blocked_by`. If the project has no "blocks" link type, leave both fields empty in the file and continue — the skill still creates everything else and just skips link creation in Phase 2 (with a warning).
5. Write `./.spec-to-jira.yaml` with all the values gathered above (plus `cloud_url` if the resource exposed one). Echo the file path to the user so they can review and commit it.

If the Atlassian MCP is not configured or authentication fails at any point during auto-detect, stop and tell the user how to authenticate. Do **not** write a partial `.spec-to-jira.yaml`.

## Input resolution

The same resolution algorithm runs for the PRD source and again for the tech spec source. Each source produces a `(content, identifier)` pair: `content` is the raw Markdown the rest of the skill operates on; `identifier` is the string written into the `plano-<slug>.md` metadata header (a path, URL, Jira key, or `pasted (<short hash>)`).

### Format detection

Classify the source string before fetching:

- **Google Docs URL**: matches `https?://docs.google.com/document/`. Anything after that (including `/d/<doc-id>/edit`, query strings, and fragments) is part of the identifier — preserve it verbatim.
- **Jira issue key**: matches the regex `^[A-Z][A-Z0-9_]+-\d+$`. Case-sensitive — Jira keys are uppercase.
- **Local `.md` path**: ends in `.md` and points at an existing file on disk. Resolve relative paths against the current working directory.
- **Interactive sentinel**: the user supplied no argument for this source. Trigger the interactive prompt described below.

If the string matches none of the above (e.g., a URL on a different host, a bare slug, a non-`.md` file), stop and tell the user which formats are supported. Do not silently guess.

### Fetching by format

- **Google Docs URL** → call the Google Drive MCP to fetch the document content as Markdown (or as plain text if Markdown export is not available; the skill will still parse it as Markdown). The `content` is the fetched body; the `identifier` is the original URL. If the Google Drive MCP is **not configured**, returns an **authentication error**, returns an **access error** (the user is not allowed to read this doc), or returns an **empty body**, fall back to the paste-or-path prompt described below.
- **Jira issue key** → call the Atlassian MCP's `getJiraIssue` against the cloud ID from `.spec-to-jira.yaml`. Use the issue's **description** (rendered as Markdown — convert Atlassian Document Format if needed) as the `content`; the `identifier` is the issue key. If the Atlassian MCP is not configured or the issue is not found, stop and tell the user — do **not** fall back to paste-or-path for Jira keys, because the same MCP is needed later to create the Epic and there is no point pretending it works.
- **Local `.md` path** → Read the file with the Read tool. The `content` is the file body; the `identifier` is the path the user provided (preserve relative vs. absolute as given). If the path does not exist or is not a `.md` file, stop and tell the user.
- **Pasted content** (only reachable via the paste-or-path fallback or interactive mode) → take whatever the user pastes into chat as the `content`. The `identifier` is the literal string `pasted (<short hash>)`, where `<short hash>` is the first 8 hex characters of the SHA-256 of the pasted content, so two pastes of the same body produce the same identifier in the plan file.

### Paste-or-path fallback (Google Docs only)

When the Google Drive MCP cannot deliver the document, prompt the user with exactly this choice — paraphrase the wording but keep both options visible:

> The Google Drive MCP is unavailable / unauthorized / returned no content. Paste the document content here, or give me an `.md` path on disk.

Accept either response:

- If the user pastes content, take it as-is as the `content`; record the identifier as `pasted (<short hash>) [originally <google-docs-url>]` so the plan file still traces back to the original URL the user tried.
- If the user gives an `.md` path, re-run the local-path branch above. Identifier becomes the path, but again append `[originally <google-docs-url>]` so traceability is preserved.

Never fall back to paste-or-path for any format other than Google Docs URLs. Local-path errors and Jira-key errors must stop the run with a clear message.

### Interactive mode (no-argument invocation, or argument-less per-source prompt)

If the user invoked `/spec-to-jira` with no arguments, ask sequentially — PRD first, then tech spec. For each, prompt:

> Provide the **PRD** (or **tech spec**) as: (a) a local `.md` path, (b) a Google Docs URL, (c) a Jira issue key, or (d) pasted content. Reply with one of those.

Run the resolution algorithm above on the response. If the user chooses "pasted content", capture whatever they paste next and treat it as the `content`, with the identifier `pasted (<short hash>)` (no `[originally ...]` suffix, since there is no original URL to record).

Do not ask the user to confirm the format — the detection is unambiguous, and the user already chose by typing.

## Process

The skill runs in two phases separated by a human review pause. Phase 1 builds the plan file; the pause lets the user edit it and resolve gaps and dependencies; Phase 2 creates **or idempotently re-syncs** the Jira issues and links from whatever the file now contains, then appends a one-line entry to `## Histórico de execuções` recording what happened. If the user picked an existing `plano-*.md` at the invocation check, **skip directly to Phase 2** with that file. Otherwise, run Phase 1 first.

### Phase 1 — build the plan file

#### 1. Load configuration

- Read `~/.config/spec-to-jira/team.yaml`. If missing, write the default file (cloud / web / data / pm) and continue with that default. Echo the path to the user so they know where to edit it.
- Read `./.spec-to-jira.yaml`. If missing, run the auto-detect flow (see above) and write the file. If present, read it silently — do not prompt the user for any value it already carries.

#### 2. Resolve the PRD and the tech spec

- Run the "Input resolution" algorithm above on the PRD source argument (or the interactive prompt response when no argument was given). The result is `(prd_content, prd_identifier)`.
- Run the same algorithm on the tech spec source argument. The result is `(tech_spec_content, tech_spec_identifier)`.
- Capture a snapshot timestamp at this point as an ISO 8601 string (e.g., `2026-05-11T14:23:07Z`). This is the value that goes into the plan file's metadata header so a reader can tell when the source documents were captured.
- Pick a slug for the plan filename from the PRD's first H1 title (lowercased, kebab-case, ASCII-only). If there is no H1, derive the slug from `prd_identifier`: take the basename for a path, the document ID segment for a Google Docs URL, the lowercased issue key for a Jira key, or `pasted` for pasted content. Still kebab-case and ASCII-only.

#### 3. Detect milestones and derive user stories per milestone

The PRD becomes one or more milestones; each milestone produces exactly one Epic in Phase 2. Use this detection order — first match wins:

1. **Explicit milestone headings (preferred)**: scan for level-2 headings whose text starts with one of `Milestone`, `Marco`, `Fase`, `Phase`, or `Release` (case-insensitive). Examples that match: `## Milestone 1`, `## Milestone — Authentication`, `## Phase 2: Admin Tools`, `## Marco 1 — Lançamento`, `## Release: V2`. Each matched heading opens a new milestone whose **name** is the heading text with the leading prefix (and any `N — ` / `N: ` numbering) stripped — e.g., `## Milestone 1 — Authentication` → `Authentication`. If the heading is just `## Milestone 1` with no name part, fall back to `Milestone 1`.
2. **Inferred groupings (fallback)**: if no explicit milestone headings exist, look for other structural signals — sub-headings like `### Now / Later`, `### MVP / V2`, `### Short-term / Long-term`, numbered roadmap-style lists, or prose like "First, ship X. Once X is in place, add Y." Each detected grouping becomes a milestone whose name is taken from the grouping label (or synthesized as `Milestone 1`, `Milestone 2`, … if no label is available). Be conservative — only split when the signal is unambiguous. When in doubt, prefer fewer milestones; the user can split further during review.
3. **Single-milestone fallback**: if no structural signal at all is found, treat the entire PRD as one milestone whose name is the PRD's H1 title.

Number the detected milestones starting at 1, in the order they appear in the PRD. The plan file always uses the `## Epic N — <name>` structure even when N=1 — there is no special-case single-Epic shorthand.

For each detected milestone, derive its own list of well-defined user stories from the prose belonging to that milestone section. Prefer stories that are already written in `As a <actor>, I want <feature>, so that <benefit>` form; if the PRD lacks them, synthesize the list from the milestone's problem statement, solution, and acceptance criteria. Each story must be a single, independently-meaningful unit of user value — do not stitch multiple goals into one story, and do not split a single goal across multiple stories.

Stories that appear in a "shared", "cross-cutting", or otherwise milestone-agnostic section of the PRD are assigned to **Milestone 1** by default. The user can move them in the review step.

For each derived user story, also extract:

- **PRD reference**: the specific PRD section / user story ID (e.g., "4.2.11") that drives this story, including a short quoted excerpt of the relevant requirement text and a link back to the PRD source document. When multiple PRD sections apply, list all of them. This reference is included in the Story's Jira description (see step 10) so reviewers can trace back to the requirement.
- **Acceptance criteria**: a checklist of concrete, testable conditions that must be true for the story to be considered complete. Derive these from the PRD's functional requirements, the tech spec's success metrics, and the tech spec's design constraints. Each criterion should be a single, verifiable statement (not a vague goal). These are included in the Story's Jira description as a bulleted checklist.

Also capture, per milestone, any **dependencies** declared in the PRD — e.g. "Milestone 2 depends on Milestone 1", "blocked by the auth rollout in Phase 1", "requires X to ship first". Record them as a list of milestone numbers / names that the current milestone depends on. If a stated dependency points at a milestone not present in the PRD, keep it as a free-text note; do not invent dependencies. The user can edit the dependency list during review. The skill does **not** create the corresponding Jira "blocks" links between Epics — the per-Epic dependency list is informational and persisted in the plan file and in each Epic's Jira description (Epic-level Jira links are reserved for a future version).

#### 4. Identify the tech spec's stack sections

The tech spec is assumed to be **stack-organized**: top-level sections grouped per stack, each section describing the implementation work needed for that stack across the whole product. Map headings in the tech spec to the stacks defined in `team.yaml`:

- **cloud**: backend, services, APIs, jobs, infra, deployment, observability that lives server-side.
- **web**: client-side application, UI, components, routing, client state, browser concerns.
- **data**: schema, migrations, ETL/ELT, pipelines, analytics, data contracts, warehousing.
- **pm** (virtual): rollout plan, comms, milestone tracking, stakeholder coordination, training, documentation hand-off.
- Any other custom stacks the user has added in `team.yaml`: use the stack name and label as the mapping signal, plus any context the user has documented in the tech spec.

Capture, per stack, the list of headings (with their heading text and an anchor or section number) that belong to that stack. Headings that do not clearly belong to any configured stack are ignored in the inverted scan — they will not appear in the matrix. If a tech spec heading legitimately spans two stacks, count it under both.

If none of the configured stacks are represented in the tech spec at all (e.g., the tech spec is empty or unrelated), stop and tell the user — there is nothing to scan.

#### 5. Inverted scan → gap-aware coverage matrix per milestone

For each derived user story (across **all** milestones), scan **all configured stack sections** of the tech spec and identify the specific tech spec section(s) that describe implementation work needed to deliver that story. The scan is **inverted**: the user story drives the lookup, and the tech spec is the lookup target. The tech spec itself does not need to be split per milestone — the same tech spec sections may legitimately cover stories from multiple milestones.

The output is **one coverage matrix per milestone**, with one row per user story in that milestone and one column per stack defined in `team.yaml` (in the order they appear there). Each cell is in **one of three states**:

- **Covered** — `✅ <citation>`: at least one tech spec section in this stack describes the work needed for this story. The citation is the heading text (or section number) used. Multiple citations are joined with `; ` inside the same cell. Generates one Sub-task per covered cell in Phase 2.
- **Gap candidate** — `❓`: no tech spec section in this stack mentions this story. The cell is marked `❓` and surfaced as an unchecked checkbox in the per-Epic **Pendências PM** subsection so the user can decide during review whether this is a real gap (track via a `pm`-stack Sub-task) or N/A (no work needed). Phase 2 generates a Sub-task **only if** the user checks the box; if the user deletes the line, the cell is blanked instead. Lines left undecided stay `❓` and are flagged in the summary.
- **N/A (only after user review)** — empty cell: the user resolved the `❓` as N/A by deleting the corresponding line from **Pendências PM** during the review pause. Phase 2 blanks the matrix cell in the rewrite. No Sub-task is created.

Phase 1 only ever produces `✅` or `❓` cells — empty cells are never written by the skill itself; they only appear after a Phase 2 rewrite that follows a user N/A decision.

#### 5a. Multi-stack split → one Sub-task per covered stack

A story covered in multiple stacks **always** produces one Sub-task **per covered stack**. Multi-stack Sub-tasks (a single Sub-task that mixes work from two or more stacks) are forbidden — do not emit them, even if the tech spec section literally combines stacks under one heading. If a tech spec section spans two stacks, count it under both stack columns of the same story (each citation appears in both cells), and emit two separate Sub-tasks — one per stack.

For every story that ends up with two or more covered stacks, emit a **default cross-stack dependency proposal** in the per-Epic **Dependências** subsection: one `blocks` link per adjacent pair of covered stacks, in the order the stacks appear in `team.yaml`. With the default `cloud → web → data → pm` order and a story covered in `cloud` and `web`, that produces:

```
- [x] [TBD] (cloud, story #1) blocks [TBD] (web, story #1) — multi-stack split
```

If the same story is covered in `cloud`, `web`, and `data`, emit two pre-checked proposals: `cloud → web` and `web → data`. The user can re-order, flip, edit, or delete these proposals during review (e.g., if the team's actual sequencing is `data → cloud → web`, the user edits the proposals before approval). Pre-check the multi-stack split proposals by default — they are the safe assumption; the user can uncheck or delete the ones that do not apply.

#### 5b. Infer sub-task ↔ sub-task dependencies from tech spec prose

After building the coverage matrix and the multi-stack split, scan the tech spec prose for **explicit blocking signals** between sections. Patterns to look for (case-insensitive — match in both English and Portuguese where applicable):

- "after X", "once X (is) in place", "once X ships", "blocked by X"
- "depends on X", "requires X exists", "needs X to be done first", "presupõe X"
- "before Y", "must precede Y", "X must ship before Y", "antes de Y"
- Ordering words within a numbered list of implementation steps: "first do X, then do Y", "primeiro X, depois Y"

For each match, identify which Sub-task each side resolves to:

- A reference to a tech spec section maps to every Sub-task that cites that section.
- A reference to a story name (or paraphrase) maps to every Sub-task whose Story matches that name within the same Epic.
- A reference to a stack ("after the cloud changes") plus a story context maps to that stack's Sub-task for that story within the same Epic.

For each unambiguous match, emit a **pre-checked** proposal in the per-Epic **Dependências** subsection:

```
- [x] [TBD] (data, story #2) blocks [TBD] (cloud, story #2) — "the cloud service reads from the warehouse table" (tech spec §3.2)
```

If the resolution is ambiguous (e.g., "after the cloud work" with no story context, or a phrase that could refer to two different sections), emit an **unchecked** proposal with the ambiguity noted in free text — the user disambiguates during review:

```
- [ ] [TBD] (cloud, story #1) blocks [TBD] (cloud, story #3) — "story #3 depends on the auth from story #1" (low confidence; ambiguous reference in tech spec §1.4)
```

Only propose dependencies **within the same Epic** — cross-Epic Sub-task ↔ Sub-task links are handled by the per-Epic `Depends on:` line at the Epic level (informational only) and are out of scope for Phase 2 link creation. If a tech spec match clearly crosses milestones (e.g., a Phase 2 story depends on a Phase 1 story), record it as a free-text note under **Dependências** of the later Epic with the prefix `(cross-epic, not auto-created)` and do **not** emit a checkbox.

Do not invent dependencies. Pre-check only the proposals you're confident about; leave anything ambiguous unchecked.

#### 5c. Generate QA test plan Sub-tasks (one per Story)

Only run this step if `team.yaml` defines a `qa` stack entry. If `qa` is not present, skip this step entirely.

For each derived user story (across all milestones), generate a QA test plan Sub-task:

- **Summary**: `[qa] Test plan: <Story summary>` — using the same summary derivation as other Sub-tasks.
- **Description**: a list of key test scenarios derived from:
  1. The Story's **acceptance criteria** (from step 3) — each criterion maps to one or more test cases.
  2. The tech spec's **QA Validation/Test Plans** section — any test scenarios explicitly described for this Story's scope.
  3. **Edge cases** inferred from the tech spec prose (error paths, timeouts, race conditions, graceful degradation).
  The description should be concrete enough for a QA engineer to write test cases from, but not prescriptive about test tooling or framework.
- **Stack**: always `qa`.
- **Assignee**: the `assignee` field from the `qa` stack entry in `team.yaml`, if non-empty. Looked up via Jira user search (display name → accountId) on Phase 2 creation.

QA Sub-tasks are appended to the `### Sub-tasks` subsection of each Epic, after the regular engineering Sub-tasks. They follow the same `- [TBD] [qa] Test plan: <summary> — parent [TBD] (story #N)` format as other Sub-tasks, so they participate in the same Phase 2 creation, idempotent re-sync, and orphan detection flows.

QA Sub-tasks do **not** appear in the coverage matrix — the `qa` column is omitted from the matrix because QA coverage is guaranteed (one per Story, always generated). They also do not appear in **Pendências PM** (no gap tracking needed).

#### 6. Write `plano-<slug>.md` with placeholder keys

In the current working directory, write a file named `plano-<slug>.md`. The file format is identical to the final artifact produced in Phase 2 (see the template below), with one difference: every Jira key is still unknown at this point, so write the literal placeholder `[TBD]` everywhere a key would appear. The user will see those placeholders in the file; they are not asked to fill them in — Phase 2 replaces them with the real keys after creating the issues in Jira.

If a `plano-<slug>.md` already exists in the cwd (for example because the user picked "new" at the invocation check while another plan file is still present), **do not overwrite it**. Pick a non-colliding slug by appending `-2`, `-3`, etc., until the filename is free. Echo the chosen filename to the user.

Template (with `[TBD]` placeholders):

```
# <PRD title>

Source PRD: <prd_identifier>
Source tech spec: <tech_spec_identifier>
Snapshot: <ISO 8601 timestamp captured in step 2>
Generated: <ISO 8601 timestamp at write time>
Jira project: <project_key>
Status: pending review

## Epic 1 — <Milestone 1 name>

Depends on: (none)

### Stories

1. [TBD] As a <actor>, I want <feature>, so that <benefit>
   PRD: <section_id> — "<short quoted excerpt>"
   Acceptance Criteria:
   - <criterion 1>
   - <criterion 2>
   - ...
2. [TBD] As a <actor>, I want <feature>, so that <benefit>
   PRD: <section_id> — "<short quoted excerpt>"
   Acceptance Criteria:
   - <criterion 1>
   - ...
...

### Matriz de cobertura

| História | <stack_1> | <stack_2> | ... | <stack_N> |
|---|---|---|---|---|
| [TBD] <feature_1> | ✅ <citation> | ✅ <citation> | ❓ | ❓ |
| [TBD] <feature_2> | ❓ | ✅ <citation> | ✅ <citation> | ❓ |
...

### Sub-tasks

- [TBD] [<stack_1>] <Story summary_1> — parent [TBD] (story #1) — cites <tech spec section>
- [TBD] [<stack_2>] <Story summary_1> — parent [TBD] (story #1) — cites <tech spec section>
- [TBD] [qa] Test plan: <Story summary_1> — parent [TBD] (story #1)
- [TBD] [<stack_2>] <Story summary_2> — parent [TBD] (story #2) — cites <tech spec section>
- [TBD] [<stack_3>] <Story summary_2> — parent [TBD] (story #2) — cites <tech spec section>
- [TBD] [qa] Test plan: <Story summary_2> — parent [TBD] (story #2)
...

### Pendências PM

- [ ] story #1 × <stack_3> — gap or N/A?
- [ ] story #1 × <stack_4> — gap or N/A?
- [ ] story #2 × <stack_1> — gap or N/A?
- [ ] story #2 × <stack_4> — gap or N/A?
...

### Dependências

- [x] [TBD] (<stack_1>, story #1) blocks [TBD] (<stack_2>, story #1) — multi-stack split
- [x] [TBD] (<stack_2>, story #2) blocks [TBD] (<stack_3>, story #2) — multi-stack split
- [x] [TBD] (<stack_2>, story #1) blocks [TBD] (<stack_2>, story #3) — "story #3 builds on the UI from story #1" (tech spec §<n>)
- [ ] [TBD] (<stack_1>, story #2) blocks [TBD] (<stack_2>, story #2) — "ambiguous reference in tech spec §<n>"

## Epic 2 — <Milestone 2 name>

Depends on: Epic 1

### Stories

1. [TBD] As a <actor>, I want <feature>, so that <benefit>
...

### Matriz de cobertura

| História | <stack_1> | <stack_2> | ... | <stack_N> |
|---|---|---|---|---|
| [TBD] <feature_X> | ✅ <citation> | ❓ | ❓ | ❓ |
...

### Sub-tasks

- [TBD] [<stack_1>] <Story summary_X> — parent [TBD] (story #1) — cites <tech spec section>
...

### Pendências PM

- [ ] story #1 × <stack_2> — gap or N/A?
- [ ] story #1 × <stack_3> — gap or N/A?
- [ ] story #1 × <stack_4> — gap or N/A?
...

### Dependências

(none)
```

Notes on the file format:

- The top-level `Status: pending review` line is the marker that this file is a Phase-1 artifact waiting for Phase 2. Phase 2 rewrites it to `Status: applied` when creation completes. There is exactly one `Status:` line in the whole file — it is **not** repeated per Epic.
- Each milestone gets its own `## Epic N — <name>` section. Epic numbering is 1-based and follows the order milestones appear in the PRD. The Epic **name** comes from the milestone name detected in step 3.
- The `Depends on:` line directly under each Epic heading lists the Epics this one depends on, by Epic number or name (e.g., `Depends on: Epic 1`, `Depends on: Epic 1, Epic 2`, `Depends on: (none)`). The list is informational at the Epic level — the skill does not create Jira "blocks" links between Epics — but it is persisted in the file and copied into each Epic's Jira description in Phase 2. The user may edit it during review or set it to `(none)`.
- Inside each Epic section, stories are numbered 1-based **within that Epic**. The same story number can appear in two different Epics — `(story #1)` in Sub-tasks, Pendências PM, and Dependências always refers to story #1 of the **same enclosing Epic**.
- In the Sub-tasks subsection under an Epic, the `(story #N)` annotation after each `parent [TBD]` records which story (by 1-based position in the **enclosing Epic's** Stories subsection) the sub-task belongs to. The user may delete that annotation if they prefer; Phase 2 will then fall back to matching the parent by Story summary text within the same Epic.
- The matrix columns are the stack names from `team.yaml`, in the order they appear there. Each Epic has its own matrix listing only the stories in that Epic. Cells use `✅ <citation>` when covered, `❓` when uncovered (gap candidate), and stay literally empty after the user resolves a `❓` as N/A in Phase 2's rewrite. Do not write `—` or `n/a`.
- The matrix is informational. The **Stories** subsection drives Story creation in Phase 2; the **Sub-tasks** subsection drives the per-(story × stack) Sub-task creation; the **Pendências PM** subsection drives gap-resolution Sub-task creation; the **Dependências** subsection drives Jira link creation. Edits to the matrix alone do not change what gets created — only edits to those four subsections do.
- **Pendências PM**: one line per `❓` cell, in the format `- [ ] story #N × <stack> — gap or N/A?`. The user resolves each line by either checking it (`- [x]` → Phase 2 creates a `pm`-stack Sub-task to track the missing work) or deleting the line entirely (→ Phase 2 blanks the matrix cell). Lines left unchecked and undeleted are treated as **unresolved** — Phase 2 leaves the matrix cell as `❓`, does not create a Sub-task, and flags the unresolved gap in the chat summary so the user knows there is a pending decision. The user may also edit the line text (e.g., to add context like `gap or N/A? — waiting on legal sign-off`) without changing the checkbox state; Phase 2 preserves the edited text on the rewrite.
- **Dependências**: one line per proposed Sub-task ↔ Sub-task link, in the format `- [<state>] [TBD] (<stack_a>, story #N) blocks [TBD] (<stack_b>, story #M) — <rationale>`. The first `[TBD]` is the **blocker** Sub-task (the issue with the outward `blocks` link); the second `[TBD]` is the **blocked** Sub-task (the issue with the inward `blocked_by` link). Both Sub-tasks must live in the **same Epic** — cross-Epic dependencies are not auto-created (see step 5b). The user manages each line by:
  - **Approving as-is**: leave the line checked (`- [x]`) — Phase 2 creates the link.
  - **Rejecting**: uncheck (`- [ ]`) or delete the line entirely — Phase 2 does not create the link.
  - **Editing the direction**: swap the two `(<stack>, story #N)` references on either side of `blocks`. Phase 2 honors the edited direction.
  - **Editing the targets**: change the stack tag or the `story #N` index. Phase 2 re-resolves to the matching Sub-task within the same Epic.
  - **Adding a new dependency**: append a new line in the same format, with `[TBD]` placeholders, and check the box. Phase 2 will resolve the references and create the link.
  - The rationale (text after ` — `) is informational and not parsed; the user may edit it freely.
- **Moving a story between Epics**: the user cuts the story line from one Epic's `### Stories` and pastes it into another Epic's `### Stories`. They also move the corresponding row in `### Matriz de cobertura`, **all** matching lines in `### Sub-tasks`, **all** matching lines in `### Pendências PM`, and **any** lines in `### Dependências` that reference that story (including dependency lines whose other side is no longer in the same Epic — those become orphaned). Phase 2 will then create the story under the destination Epic. If the user moves the story but forgets to move its sub-tasks / pendências / dependências, those lines become orphaned (their `(story #N)` reference does not resolve in the same Epic) — Phase 2 will warn and skip them rather than reassign them silently across Epics.
- **Renaming an Epic**: edit the text after ` — ` on the `## Epic N — <name>` line. Do not change the `N` — Epic numbers are positional. If the user renumbers Epics by hand, Phase 2 still treats them in the order they appear in the file.
- **Deleting an Epic**: delete the entire `## Epic N — <name>` heading and its `Depends on`, `### Stories`, `### Matriz de cobertura`, `### Sub-tasks`, `### Pendências PM`, and `### Dependências` subsections. Phase 2 will not create that Epic, its Stories, its Sub-tasks, its gap Sub-tasks, or any of its links.
- **Adding a new Epic**: append a new `## Epic N — <name>` section at the end of the file with the next integer in sequence. Use `[TBD]` for the keys on every line inside. Include a `Depends on:` line (use `(none)` if there are no dependencies) and the six subsections (`### Stories`, `### Matriz de cobertura`, `### Sub-tasks`, `### Pendências PM`, `### Dependências`). If the file already has `## Orphans` and `## Histórico de execuções` sections at the bottom, place the new Epic **before** them — those two sections always remain at the very end of the file.
- **`## Orphans`** (managed by Phase 2, regenerated each run): added by step 12 when step 8a discovers Jira issues whose keys are no longer in the file. The block is regenerated fresh on every run: orphans the user has already closed / deleted in Jira disappear automatically on the next run. The user may delete the entire `## Orphans` block manually — step 8a will re-create it if there are still orphans to report. Do **not** edit lines inside the block by hand; the format is parser-friendly and is overwritten on each run.
- **`## Histórico de execuções`** (managed by Phase 2, append-only): created by step 12 the first time Phase 2 completes, and grown by one line on every subsequent run. Each line is parser-friendly: `- <timestamp> — <kind>: created E.../S.../T.../L...; updated E.../S.../T...; orphans S.../T.../L...; failures N`. The user may edit the section heading text or annotate individual lines (e.g., adding `# notes about this run` after a line), but should not delete prior lines — the log is the audit trail for the plan's evolution.

#### 7. Pause for review

After writing the plan file, print in chat exactly one message and **stop**. Do not continue to Phase 2 in the same turn — the user must have a chance to open the file in their editor first. Use this wording (paraphrasable, but the meaning must be preserved):

> Plan written to `plano-<slug>.md`. Resolve the gaps in **Pendências PM**, review the proposed links in **Dependências**, edit anything else you need to, and type `ok` when ready.

Do not also print a preview of the matrix, the gaps, or the dependencies in chat — the file *is* the preview. The user reviews directly in their editor.

### Pause (human checkpoint)

The skill is now idle. The user edits `plano-<slug>.md` outside of chat (in their editor, on disk) and returns to chat to indicate readiness. The next user message ends the pause:

- Replies that count as **approval** (resume Phase 2): `ok`, `OK`, `okay`, `go`, `proceed`, `continue`, `aprovado`, `pode ir`, or any obvious paraphrase of approval. When in doubt, ask once to confirm rather than guessing.
- Replies that count as **cancel** (stop, leave the file in place): `cancel`, `abort`, `stop`, `cancela`, `parar`.
- Anything else (questions, follow-up requests, asking the skill itself to edit the file): respond and continue waiting. Do not advance to Phase 2 until the user explicitly approves.

If the user cancels, leave `plano-<slug>.md` on disk untouched (still `Status: pending review`). A future `/spec-to-jira` invocation will detect it via the invocation check and offer to continue it.

### Phase 2 — create or idempotently re-sync Jira issues and links from the plan file

Phase 2 has one entry point but two behaviors, decided per item on read:

- **Create** — the line has no real Jira key (placeholder `[TBD]` or no bracketed key at all). Phase 2 calls `createJiraIssue` (or `createIssueLink` for dependencies) and writes the returned key back into the file. This is the first-run path.
- **Update** — the line carries a real Jira key, and the line's mutable fields (summary, description, labels, parent / Epic Link, link relations) **diverge** from the current Jira state. Phase 2 calls `editJiraIssue` exactly once to bring Jira back in sync with the file. Dependency lines instead toggle the underlying Jira link (create it if missing, leave it alone if already present — there is no "update" of a link, only presence/absence).
- **Skip** — the line carries a real Jira key, and the line's mutable fields **match** Jira. Phase 2 performs zero Jira writes for that item.
- **Orphan** — a Jira issue that is reachable from a parent recorded in the file (an Epic's Story descendants, a Story's Sub-task descendants) carries a key that does **not** appear anywhere in the file. Phase 2 records the orphan in the run summary, in `## Histórico de execuções`, and as an `Orphans:` block in the rewritten file. It does **not** delete or transition the Jira issue.

Mutable fields and the `status` / `assignee` rule:

- Only **summary, description, labels, parent / Epic Link, and the presence of link relations** are ever written by Phase 2 on existing Jira issues. Components are written on create but not changed on update (the skill does not re-attach / re-detach components on re-runs).
- **`status` and `assignee` are never touched on existing issues**, under any circumstance — not on create (Jira's default applies), not on update, not on orphan handling. If a teammate changed the assignee or moved a Story to "In Progress" in Jira between runs, the next re-run leaves both fields alone.
- The skill does not three-way-merge: if a teammate edited a Story title directly in Jira and the `.md` was untouched, the next re-run will overwrite Jira back to whatever the `.md` says. The `.md` is the source of truth.

#### 8. Re-read the plan file from disk

Read `plano-<slug>.md` fresh from disk with the Read tool. **Do not** rely on in-memory state from Phase 1 — the user has had a chance to edit the file, and any divergence between in-memory state and the file on disk must resolve in favor of the file. The same rule applies when Phase 2 was entered via the invocation check on a pre-existing plan file.

Parse:

- **PRD title** — the H1 line at the top of the file.
- **Metadata header** — `Source PRD:`, `Source tech spec:`, `Snapshot:`, `Generated:`, `Jira project:`. The `Jira project` value must match `project_key` in `./.spec-to-jira.yaml`; if it does not, stop and tell the user the file targets a different project than the cwd is configured for.
- **Status** — `pending review` selects the **first-run flow** (a `Histórico de execuções` entry is appended at the end). `applied` selects the **idempotent re-sync flow** described in this Phase 2 intro and in the per-step "update path" subsections below. No re-confirmation prompt is needed for `applied` — the operation is safe to repeat. Any other value (typo, partial edit) is malformed and stops the run with a clear error.
- **Histórico de execuções** — read every existing `- <timestamp> — <kind>: …` line in the `## Histórico de execuções` section at the bottom of the file, if the section exists. Keep them verbatim — step 12 appends one new line at the end of the section; it never rewrites or removes older entries. If the section does not exist, step 12 creates it. The log is informational only; nothing in steps 9–11b parses it for decision-making.
- **Epics** — every `## Epic N — <name>` heading, in the order they appear in the file. For each Epic, parse:
  - The Epic's **name** — the text after ` — ` on the heading line.
  - The Epic's **Jira key** — for a fresh plan the heading has no key; for a partially-completed prior run the heading reads `## Epic N — <name> [<KEY>]`. Record the key when present, treat its absence as "needs creation".
  - The **Depends on** line directly under the heading — a comma-separated list of Epic numbers / names, or `(none)`. Record it verbatim for the Jira description in step 9 and for the rewrite in step 12. The skill does not create Jira links between Epics.
  - The Epic's **Stories** — every `<n>. [<key>] <story sentence>` line in the Epic's `### Stories` subsection, in the order they appear. Each line becomes one Story belonging to this Epic. If a line already carries a real key, capture it for parenting and skip creation.
  - The Epic's **Sub-tasks** — every `- [<key>] [<stack>] <summary> — parent [<parent_key>] (story #N) — cites <citation>` line in the Epic's `### Sub-tasks` subsection. Parse the `[<stack>]` tag to look up the corresponding stack in `team.yaml`. Resolve the parent Story key in this order: explicit Story key when present, story index `N` within the **same Epic**, parent summary text matched against the Stories of the **same Epic**. If none of those resolve to a Story in the same Epic, the sub-task is **orphaned** — flag it but do not reassign across Epics (see step 11).
  - The Epic's **Pendências PM** — every `- [<state>] story #N × <stack> — <text>` line in the Epic's `### Pendências PM` subsection. The `<state>` is `x` (checked → real gap → create `pm`-stack Sub-task), space (unchecked → unresolved → leave `❓` in matrix, no creation, flag in summary), or any other character (treat as unresolved and warn). Resolve `story #N` against the **Stories** subsection of the same Epic — if it does not resolve, flag the line as orphaned and skip. The `<stack>` is the column the gap covers; it should match a stack in `team.yaml`. The trailing `<text>` is the user's free-form context (e.g., "waiting on legal") and is preserved on the rewrite.
  - The Epic's **Dependências** — every `- [<state>] [<key_a>] (<stack_a>, story #M) blocks [<key_b>] (<stack_b>, story #N) — <rationale>` line in the Epic's `### Dependências` subsection. The `<state>` is `x` (checked → create the Jira link in step 11b), space (unchecked → skip), or any other character (treat as unchecked and warn). Resolve `[<key_a>]` and `[<key_b>]` to Sub-tasks **within the same Epic** in this order: explicit Sub-task key when present, then `(<stack>, story #N)` reference. Both sides must resolve within the same Epic; if either side does not, the line is **orphaned** and Phase 2 skips it (see step 11b). The `<rationale>` text is preserved on the rewrite but otherwise unused.
  - The Epic's **Matriz de cobertura** — informational only, not parsed for issue creation. Phase 2 rewrites it in step 12 to reflect the real Story keys, the user's gap resolutions (gap `❓` → `❌` for missing PM coverage when a Sub-task creation failed, blank for N/A, or `✅ pm sub-task <PM_KEY>` for an accepted gap that was created), and any failures.

If a line in an Epic's Stories subsection was deleted by the user, do not create that Story — and skip every Sub-task in the same Epic whose parent annotation points at the deleted story, every Pendências PM line that references the deleted story, and every Dependências line that references a Sub-task of the deleted story. If a line in a Sub-tasks subsection was deleted, do not create that Sub-task. If a line in a Pendências PM subsection was deleted, treat it as **N/A** — blank the corresponding matrix cell in step 12. If a line in a Dependências subsection was deleted, treat it as a rejected suggestion — do not create the link. If an entire `## Epic N — <name>` section (heading + subsections) was deleted, skip the Epic, all its Stories, all its Sub-tasks, all its Pendências PM lines, and all its Dependências lines. **This is how the user vetoes items during review**: just delete the line (or the whole Epic section).

If a line was edited (story title rewritten, sub-task title rewritten, gap line annotated, dependency direction flipped, citation reworded, Epic renamed, dependency list changed), use the **edited** text when creating the Jira issue or link. The file is authoritative.

If the file is malformed (missing H1, broken metadata header, zero `## Epic N — <name>` sections, or an Epic section that has Sub-task lines but no `### Stories` subsection), stop and tell the user exactly which section is malformed. Do not guess. A missing `### Matriz de cobertura`, `### Pendências PM`, or `### Dependências` is **not** malformed — those subsections are optional and their absence means "no gaps to resolve" / "no dependencies to create". They are regenerated as needed in step 12 from the Stories and Sub-tasks subsections of the same Epic.

After parsing, build an in-memory map of **every Jira key referenced by the file** — Epic keys on `## Epic N — <name> [<KEY>]` headings, Story keys on Stories lines, Sub-task keys on Sub-tasks and accepted Pendências PM lines, and link endpoint keys on Dependências lines. This set is the **canonical inventory** for orphan detection in step 8a (only meaningful on `Status: applied` re-runs — on a `pending review` first run the set is typically empty and step 8a is a no-op).

#### 8a. Discover Jira orphans (re-run only)

Only run this step when `Status:` is `applied`. On a first run there is nothing to discover — every line is a create.

For each Epic in the file that carries a real Jira key:

1. Use the Atlassian MCP's `searchJiraIssuesUsingJql` to list every Story whose Epic Link / parent points at this Epic key. Use a JQL like `"Epic Link" = <EPIC_KEY> OR parent = <EPIC_KEY>` (try the first form; if the project does not expose the `Epic Link` field, retry with `parent = <EPIC_KEY>`).
2. For each returned Story key, check the canonical inventory. If the key is **not** in the inventory, record it as a **Story orphan** — capture the Story's current Jira summary so step 12 can echo it in the `Orphans:` block and step 13 can show it in chat.
3. For each Story key that **is** in the inventory, call `searchJiraIssuesUsingJql` with `parent = <STORY_KEY>` to list its Sub-tasks. For each Sub-task key not in the inventory, record it as a **Sub-task orphan** with its current Jira summary.
4. For each existing Jira `blocks` link between two Sub-tasks **both of which** are still in the inventory, but where no Dependências line in the file describes that link (regardless of checkbox state), record it as a **Link orphan**. Only consider links whose endpoints are both in the inventory — links to Sub-tasks that are themselves orphans are reported via the Sub-task orphan, not duplicated as a Link orphan.

Do **not** call the Atlassian MCP for an Epic whose heading has no real key — that Epic is in "create" mode and has no Jira state to compare against.

If the MCP search fails for an Epic (e.g., transient API error), report it in step 13 as `orphan discovery failed for <EPIC_KEY>` and continue with the next Epic — the run still completes; the file's existing items are still skip / update / create as usual.

#### 9. Create or update one Epic per milestone

Iterate the `## Epic N — <name>` sections in the order they appear in the file. For each Epic still present (i.e., not deleted by the user), classify on the heading's Jira key:

- **No key on the heading** → **create**. Call the Atlassian MCP's `createJiraIssue` with:
  - `cloudId`: `cloud_id` from `.spec-to-jira.yaml`.
  - `projectKey`: `project_key` from `.spec-to-jira.yaml`.
  - `issueTypeName`: `issue_types.epic` from `.spec-to-jira.yaml`.
  - **Summary**: the name on the `## Epic N — <name>` heading — i.e., whatever the user left there during review (renames are honored).
  - **Description**: a short pointer back to `prd_identifier`, a pointer to `tech_spec_identifier`, the snapshot timestamp, a one-paragraph summary derived from the PRD section for that milestone, and the Epic's `Depends on:` list verbatim (so a reviewer can see the declared dependencies even though they are not Jira link issues yet).
  Capture the returned Jira key (e.g., `PROJ-1234`); step 12 writes it onto the heading.
- **Key on the heading** → fetch the Epic from Jira via `getJiraIssue` and compare:
  - **Mutable fields (summary, description, labels) all match the file** → **skip**. Record this as `Epic skipped (no divergence)` for the run summary.
  - **At least one mutable field diverges from the file** → **update**. Call `editJiraIssue` exactly once with only the diverged fields. The Epic's `Depends on:` list is re-rendered into the description block on update, so a teammate that edits `Depends on:` in the file will see the new list show up in Jira. Components are not changed on update. `status` and `assignee` are never sent — even when the user changed nothing else, those fields are excluded from the payload.
  Use the existing Epic key as the parent for that Epic's Stories.

If Epic creation **or update** fails for one milestone, do not abort the whole run — record the failure (it will be surfaced in step 12 as `❌ <reason>` on the heading and in step 13) and continue with the next Epic. Stories, Sub-tasks, Pendências PM Sub-tasks, and Dependências links belonging to a failed Epic are still attempted in steps 10, 11, and 11b only when the Epic's key is known (created on a prior run, or successfully created this run); if the Epic itself never got a key, skip its descendants entirely for this run.

#### 10. Create or update one Story per Stories line, parented to its Epic

For each Epic in turn, iterate every line still present in that Epic's `### Stories` subsection. Classify on the line's Jira key:

- **No key on the line** → **create**. Call `createJiraIssue` with `issueTypeName` set to `issue_types.story` from `.spec-to-jira.yaml`, linked to **that Epic's** key from step 9. Use whichever mechanism the project supports for parenting a Story to an Epic — the `Epic Link` custom field on classic projects, or the `parent` field on next-gen / team-managed projects. If the first attempt fails because the field is not available, retry with the other mechanism before giving up.
  - **Summary**: derived from the **user-edited** story sentence in the file — extract the `<feature>` clause from the `As a <actor>, I want <feature>, so that <benefit>` pattern when present; otherwise use the full edited sentence as the summary.
  - **Description**: structured as:
    1. **PRD Reference** section — the PRD section ID(s), a short quoted excerpt of the relevant requirement text, and a link back to the PRD source document (using `prd_identifier`). When multiple PRD sections apply, list all of them. This allows reviewers to trace the Story back to the driving requirement.
    2. A short paragraph describing the Story's scope and purpose (the full story sentence plus any extra context the user kept in the file).
    3. **Acceptance Criteria** section — a bulleted checklist of concrete, testable conditions derived in Phase 1 step 3. Each criterion is a single verifiable statement. The user may have edited, added, or removed criteria during the review pause — use whatever the file contains.
  - **Assignee** (create only): if `team.yaml` has a non-empty `assignees.story_assignee`, look up the display name via the Jira user search API (`/rest/api/3/user/search?query=<name>`) to resolve the `accountId`, and set the `assignee` field on creation. If the lookup fails (user not found), skip the assignment silently and continue — do not block creation. Assignees are **only set on create**, never on updates or re-runs.
  A user who moved a story between Epics (cut from Epic 1's `### Stories`, pasted into Epic 2's `### Stories`) before approval — and whose line had **no** key when they moved it — will see the Story created under Epic 2's key here. The file's structure drives parenting, not any memory of the original Phase 1 assignment.
- **Key on the line** → fetch the Story from Jira via `getJiraIssue` and compare:
  - **Mutable fields (summary, description, labels) all match the file** → **skip**. Use the existing key for sub-task parenting.
  - **At least one mutable field diverges** → **update**. Call `editJiraIssue` exactly once with only the diverged fields. `status` and `assignee` are never sent.
  - **Real-key Story line moved between Epics** (the Story's Epic Link / parent in Jira does not match the Epic that now contains the line in the file): the skill does **not** re-parent existing Stories across Epics — that is out of scope. Treat the line's mutable fields normally (skip or update them as above), but **also** flag the divergence in the run summary and in `## Histórico de execuções` so the user knows the parenting is out of sync. The user can move the Story in Jira manually.

Capture the Jira key for each Story (returned by `createJiraIssue` or already on the line) and remember which Epic it belongs to (used by step 11 for parent resolution and orphan detection).

#### 11. Create or update Sub-tasks (covered cells + accepted PM gaps), parented to their Story

For each Epic in turn, process Sub-tasks in this order: first every line still present in `### Sub-tasks`, then every **checked** line in `### Pendências PM` (the accepted gaps). Classify each line on its Jira key.

For a **`### Sub-tasks` line**, the **mutable fields** are:

- **Summary**: the text on the sub-task line between the `[<stack>]` tag and the ` — parent ` separator. This is the user-editable title — use whatever the user wrote, do not regenerate it from the Story summary or the stack `title_prefix`.
- **Labels**: include the matching stack's `label` value from `team.yaml` — e.g. `stack:cloud`. Do not add other labels.
- **Description**: state which Story this Sub-task implements (by Story key), cite the relevant tech spec section as written on the line, and include `tech_spec_identifier` so a reviewer can locate the original tech spec document.

For a **checked `### Pendências PM` line** (an **accepted gap**), the **mutable fields** are:

- **Stack**: always the virtual `pm` stack. The Sub-task's labels come from the `pm` entry in `team.yaml`.
- **Summary**: `gap: <story summary> — <missing stack> coverage` (e.g., `gap: Onboarding flow — data coverage`). The user may have edited the gap line text (e.g., added "waiting on legal sign-off") — append the user's free-form note to the description, not to the summary, so the summary stays scannable.
- **Description**: state which Story this gap belongs to (by Story key), state the missing stack column (`<stack>`), include the user's free-form note from the gap line, and include `tech_spec_identifier`. The description should make clear that this is a **PM tracking issue for missing stack coverage**, not a regular implementation Sub-task.

Classification per line:

- **No key on the line** → **create**. Use `createJiraIssue` with `issueTypeName` set to `issue_types.sub_task` from `.spec-to-jira.yaml`, parented to the resolved Story key from step 10 (parent resolution is scoped to the **same Epic** — see step 8 for the order). On create, also attach the **Component** if the matching stack has a non-empty `component` field in `team.yaml` (if the project does not have that component configured, skip the component silently and continue with the Sub-task creation). Components are **not** changed on later updates.
  - **Assignee** (create only): resolve the assignee for this Sub-task using the following priority order:
    1. If `team.yaml` has a non-empty `assignees.stack_overrides.<stack>` for this Sub-task's stack, use that display name.
    2. Otherwise, if the stack entry in `team.yaml` has a non-empty `assignee` field (e.g., the `qa` stack's `assignee`), use that.
    3. Otherwise, no assignee is set.
    Look up the display name via the Jira user search API (`/rest/api/3/user/search?query=<name>`) to resolve the `accountId`. Cache user lookups within the run to avoid repeated API calls. If the lookup fails (user not found), skip the assignment silently and continue. Assignees are **only set on create**, never on updates or re-runs.
- **Key on the line** → fetch the Sub-task from Jira via `getJiraIssue` and compare:
  - **Mutable fields (summary, description, labels) all match the file** → **skip**. Use the existing key for dependency resolution in step 11b.
  - **At least one mutable field diverges** → **update**. Call `editJiraIssue` exactly once with only the diverged fields. The parent reference is **not** re-sent on update — once a Sub-task is in Jira, the skill does not re-parent it. `status` and `assignee` are never sent.

If a Sub-tasks line's parent annotation does not resolve to any Story line in the same Epic (an **orphan** — typically because the user moved the parent story to another Epic but forgot to move the sub-task), warn the user in the chat summary, append ` — ⚠️ orphan parent` to the line during the step 12 rewrite, and skip creation.

If a Pendências PM line's `story #N` reference does not resolve in the same Epic, also flag as orphan, append ` — ⚠️ orphan parent`, and skip creation.

For Pendências PM lines that are **unchecked** (and not deleted): do not create a Sub-task, leave the `❓` in the matrix during step 12's rewrite, and surface the unresolved gap in the chat summary so the user knows there is still an open decision.

Iterate Sub-tasks in this order: outer loop = Epics in file order; inner loop within each Epic = Sub-tasks lines first, then accepted Pendências PM lines (so that gap-tracking Sub-tasks come after the regular ones in the same Story). Capture the Jira key for each Sub-task (returned by `createJiraIssue` or already on the line) and remember which Sub-task each `(stack, story #N)` reference resolves to within each Epic — step 11b uses that mapping to verify or create links.

#### 11b. Create or verify Jira `blocks` / `blocked by` links from approved Dependências

Only run this step if `link_types.blocks` and `link_types.blocked_by` are non-empty in `.spec-to-jira.yaml`. If either is empty, skip the step and warn the user in the chat summary that no links were created because the project has no "blocks" link type configured.

Links have no "update" path — a Jira link either exists between two issues with a given relation, or it does not. The idempotent operation for links is therefore **verify-then-create-if-missing**.

For each Epic in turn, iterate every **checked** line in that Epic's `### Dependências` subsection. For each line:

- Resolve the **blocker** Sub-task: the Sub-task referenced by `(<stack_a>, story #M)` on the left of `blocks`, scoped to this Epic. Use the explicit `[<key_a>]` if it carries a real key (resumed plan); otherwise look up the mapping built in step 11.
- Resolve the **blocked** Sub-task: the Sub-task referenced by `(<stack_b>, story #N)` on the right of `blocks`, scoped to this Epic. Same resolution order.
- If both sides resolve to **real** Jira keys (not placeholders), first verify whether the link already exists in Jira:
  - Call `getJiraIssue` on the blocker Sub-task and inspect its `issuelinks` array (or call `getJiraIssueRemoteIssueLinks` if the MCP requires a dedicated endpoint).
  - If an outward `blocks` link to the blocked Sub-task's key is already present → **skip**. Treat as a successful idempotent op for the run summary (counts under "skipped, no divergence").
  - If no such link is present → **create**. Call the Atlassian MCP's `createIssueLink` with:
    - `cloudId`: from `.spec-to-jira.yaml`.
    - `inwardIssueKey`: the **blocked** Sub-task's key.
    - `outwardIssueKey`: the **blocker** Sub-task's key.
    - `linkTypeName`: `link_types.blocks` from `.spec-to-jira.yaml` (the outward direction name; the MCP picks up the inward name `link_types.blocked_by` from the link type definition).
- If at least one side is still a `[TBD]` placeholder (e.g., the user added the line with placeholders just for this re-run) but the placeholder resolves to a Sub-task that was just created in step 11, use the newly created key for that side and proceed as above. Verification can be skipped when at least one side was created this run — by definition the link did not exist a moment ago.
- If either side does not resolve to a Sub-task within the same Epic, the line is **orphaned**: append ` — ⚠️ orphan reference` to the line during the step 12 rewrite, skip the link, and flag it in the chat summary. Do not auto-fix by reaching across Epics.
- For unchecked lines (`- [ ]` or any other unrecognized state) and deleted lines: do not create a link. **Importantly, the skill does NOT remove an existing Jira link when the user unchecks or deletes a previously-applied Dependências line.** That is in the same family as deletion and is out of scope — the existing link stays in Jira, and step 8a's orphan-detection sweep will list it as a **Link orphan** so the user can remove it manually.

Capture which links were verified-as-present, which were created, and which failed; all three lists feed into step 12's rewrite and step 13's summary.

If the same dependency appears twice (duplicate line, or two lines that resolve to the same blocker/blocked pair), perform the verify-or-create only once; flag the duplicate in the summary so the user knows one was skipped.

Iterate links in file order — outer loop = Epics; inner loop = Dependências lines within each Epic.

#### 12. Update the plan file with the run's outcome

Rewrite `plano-<slug>.md` in place to reflect what just happened. The rewrite is the same shape for first-run and re-run — the only difference is which marks need to be set / refreshed and what the appended log entry says.

**Key replacements (first-run path mostly; harmless on re-runs since the keys are already there):**

- On each `## Epic N — <name>` heading whose Epic was just **created**, append ` [<EPIC_KEY>]` so the heading reads `## Epic N — <name> [<EPIC_KEY>]`. On re-runs the heading already carries `[<EPIC_KEY>]`; leave it untouched. If an Epic creation / update failed this run, leave its heading otherwise intact and append ` — ❌ <reason>` to the heading line; skip the per-Epic Story / Sub-task / link replacements for that Epic and leave their `[TBD]` placeholders (if any) in place. On a re-run, an Epic that was updated does **not** receive any extra marker — a clean update is invisible in the file (only the `Histórico de execuções` entry records it).
- Replace each `[TBD]` placeholder on Story and Sub-task lines with the real Jira key returned this run. Lines that already carry a real key are left as-is for **skip** outcomes and updated only in the rewrite block below for **update** outcomes.
- Replace each `[TBD]` placeholder on Dependências lines (both the blocker and the blocked side) with the real Sub-task keys created in step 11. Lines that already carry real keys on both sides stay as-is.

**Per-Epic matrix rewrite:**

For each Epic, rewrite its matrix:

- The first column (`| [TBD] <feature> |` becomes `| <STORY_KEY> <feature> |`).
- Cells for **covered Sub-tasks** that were successfully created **or skipped / updated as idempotent re-runs** stay as `✅ <citation>`.
- Cells for **`❓` gaps** that the user **checked** in Pendências PM are rewritten to `✅ pm sub-task <PM_KEY>` (using the `pm` Sub-task key created in step 11), making the matrix self-explanatory: the gap was acknowledged and is now tracked.
- Cells for **`❓` gaps** that the user **deleted** from Pendências PM (resolved as N/A) are blanked — leave the cell empty between the pipes.
- Cells for **`❓` gaps** that the user left unresolved (line still present and unchecked) stay as `❓`.
- Cells for sub-tasks that **failed to create** become `❌ <reason>` instead of `✅ <citation>` (or `✅ pm sub-task ...`). Cells for sub-tasks that **failed to update** stay as `✅ <citation>` but the Sub-task line gets `❌ <reason>` appended (see below).

**Sub-tasks, Pendências PM, Dependências per Epic:**

- Rewrite `parent [TBD]` references in each Epic's Sub-tasks subsection so they point at the created Story keys within the same Epic. Already-resolved parents stay as-is.
- For each Sub-task line: if it was **updated** this run, append ` — ✏️ updated` (informational, removed on the next rewrite if not re-updated). If it **failed to update or create**, append ` — ❌ <reason>` and leave any placeholder in place. **Skip** outcomes get no marker.
- For the Pendências PM subsection of each Epic:
  - Lines whose checkbox was checked and whose `pm` Sub-task was created this run: replace `[ ]` with `[x]` if not already, and append ` → [<PM_KEY>] ✅` to the line so the user can see the created Sub-task key.
  - Lines whose checkbox was checked and whose `pm` Sub-task was updated this run: append ` — ✏️ updated`.
  - Lines whose checkbox was checked but the `pm` Sub-task failed to create or update: append ` → ❌ <reason>`.
  - Lines that were left unchecked (unresolved): leave the line as-is.
  - Lines the user deleted: do not regenerate them.
- For the Dependências subsection of each Epic:
  - Lines whose checkbox was checked and whose link was created this run: keep the `[x]` and append ` → ✅` to the line. If the MCP returned a link ID, append ` ✅ link <link_id>` instead.
  - Lines whose checkbox was checked and whose link was **verified-as-already-present** during a re-run: keep the `[x]` and append ` → ✅` (or leave any prior `→ ✅` mark intact). The state is the same as "link exists in Jira", so no distinct marker is needed.
  - Lines whose checkbox was checked but the link failed to create / verify: append ` → ❌ <reason>`.
  - Lines that were left unchecked: leave them as-is.
  - Lines the user deleted: do not regenerate them.
  - Orphaned lines (one or both sides did not resolve in the same Epic): append ` — ⚠️ orphan reference`.

**Status and Orphans block:**

- Change the top-level `Status:` header from `pending review` to `applied` on a first run. On a re-run, `Status:` is already `applied`; leave it.
- If step 8a discovered any orphans (Stories, Sub-tasks, or Links that exist in Jira but are absent from the file), append (or refresh, if already present) a `## Orphans` section directly above `## Histórico de execuções`. Format:
  ```
  ## Orphans

  - Story orphan: PROJ-201 "Old story title" — line removed from this file on a prior run; close or move in Jira manually
  - Sub-task orphan: PROJ-305 "[cloud] Old subtask" — parent Story PROJ-202 still present in file
  - Link orphan: PROJ-310 blocks PROJ-311 — neither endpoint has a matching Dependências line; remove in Jira manually
  ```
  Regenerate the section fresh each run from step 8a's findings — orphans the user has already cleaned up in Jira disappear from the section on the next run automatically. If no orphans were discovered, omit the `## Orphans` section entirely (and remove it if a prior run left it behind).
- If the project has no "blocks" link type (link types empty in `.spec-to-jira.yaml`), append ` — ⚠️ skipped: no blocks link type` to every checked Dependências line instead of creating links, and surface the global warning in step 13.

**Append a `## Histórico de execuções` entry:**

After all of the above, append exactly one line at the end of the `## Histórico de execuções` section at the bottom of the file. Create the section if it does not exist. Format:

```
- <timestamp> — <kind>: created E<created_epics>/S<created_stories>/T<created_subtasks>/L<created_links>; updated E<updated_epics>/S<updated_stories>/T<updated_subtasks>; orphans S<orphan_stories>/T<orphan_subtasks>/L<orphan_links>; failures <total_failures>
```

Where:
- `<timestamp>` is an ISO 8601 UTC timestamp captured at the start of step 12 (e.g., `2026-05-11T15:42:03Z`).
- `<kind>` is `first-run` when this run flipped `Status:` from `pending review` to `applied`, otherwise `re-run`.
- The counts are exact integers — zero when the count is zero. Do not omit the zero counts; the parser-friendly format makes the log diffable across runs.
- Sub-task counts are summed across both `### Sub-tasks` lines and accepted `### Pendências PM` lines (i.e., everything that became a Jira Sub-task).
- Link counts cover Dependências links only.

Example entries for the three demoable scenarios from the acceptance criteria:

```
- 2026-05-11T14:00:00Z — first-run: created E2/S5/T12/L4; updated E0/S0/T0; orphans S0/T0/L0; failures 0
- 2026-05-12T09:30:00Z — re-run: created E0/S0/T0/L0; updated E0/S0/T0; orphans S0/T0/L0; failures 0  # unchanged .md → zero writes
- 2026-05-13T16:15:00Z — re-run: created E0/S0/T0/L0; updated E0/S1/T0; orphans S0/T0/L0; failures 0  # one Story title edited → exactly one update
- 2026-05-14T11:45:00Z — re-run: created E0/S0/T0/L0; updated E0/S0/T0; orphans S0/T1/L0; failures 0  # one Sub-task line deleted → orphan reported, nothing deleted
```

Do not rewrite or remove any prior entry — the section is append-only. The user can manually edit history if they want, but the skill never touches existing lines.

Do not change anything else in the file — preserve every user edit to titles, citations, dependencies, gap rationales, Epic order, and story order. Specifically, on a re-run the file should be byte-identical to its pre-run state except for: any `❌` / `✏️ updated` markers added by this run, any prior such markers refreshed or removed when the underlying state changed, the `## Orphans` block, and the new `## Histórico de execuções` line.

#### 13. Summarize in chat

Print a concise summary, grouped per Epic. For each Epic: its key + name, and per-issue-type counts split as **created / updated / skipped** under it (Stories, Sub-tasks, Links). For first-runs the updated / skipped counts will be zero on every Epic; for re-runs they carry the real signal of what changed. Include the keys of any items that were **created** or **updated** this run (not the ones that were skipped — they would drown the summary on stable re-runs). After the per-Epic breakdown, include the total counts across all Epics and the path to the updated `plano-<slug>.md`. Include a direct link to each Epic in Jira if the MCP response includes one or if `cloud_url` is set in `.spec-to-jira.yaml`.

The chat summary **must end with the same single line that step 12 just appended to `## Histórico de execuções`**, prefixed with `Log:`. This is the acceptance criterion's "chat summary matches the `.md` log" — the user can paste either into a ticket and they will carry identical counts.

Flag explicitly in the summary:

- Any **Epic that failed to create or update** (and the resulting skipped Stories / Sub-tasks / links under it).
- Any **orphaned sub-task line** in the file (parent annotation does not resolve within the same Epic).
- Any **orphaned dependency line** (blocker or blocked side does not resolve within the same Epic).
- Any **Jira orphan** discovered in step 8a — list the orphan key plus its current Jira summary so the user can act on it manually. Group as `Story orphans:`, `Sub-task orphans:`, `Link orphans:`. The skill never deletes; the user decides whether to close / delete in Jira.
- Any **unresolved gap** in Pendências PM (line still present and unchecked, no decision recorded).
- Any **accepted gap** that produced a `pm` Sub-task **this run** — list the Sub-task keys explicitly so the user can find them in Jira. (Accepted gaps that were created on a prior run and skipped this run do not need to be re-listed.)
- Any **rejected dependency** the skill had pre-checked but the user unchecked or deleted (informational, so the user remembers their own decision).
- Any **skipped link creation** because `.spec-to-jira.yaml` has no `blocks` link type configured.
- Any **real-key Story line the user moved between Epics** — note that Jira and file are now out of sync because the skill does not re-parent existing Stories.
- Any **orphan-discovery failure** from step 8a (transient MCP search error for one Epic) — the user can re-run later to pick those up.

If `.spec-to-jira.yaml` was newly written by the auto-detect flow on this run, mention it explicitly so the user knows to review and commit the file.

## Failure handling

- If the Atlassian MCP is not configured or authentication fails — at auto-detect time, at Jira-key fetch time, at Epic creation, at Sub-task creation, at link creation, or anywhere else — stop and tell the user how to authenticate. Do not write a partial `.spec-to-jira.yaml`, and do not flip `Status:` in the plan file.
- If a **Jira issue key** input refers to a missing or inaccessible issue, stop and tell the user the issue key was not found. Do **not** fall back to paste-or-path — the Atlassian MCP is needed later anyway, so masking its failure here only delays the same error.
- If a **Google Docs URL** input cannot be fetched (Drive MCP missing, auth failure, access denied, empty body), trigger the paste-or-path fallback described in "Input resolution". Do not abort.
- If a **local `.md` path** input does not exist or is not a `.md` file, stop and tell the user. Do not auto-fall-back — the user typed a path, not a URL, so paste-or-path would be surprising here.
- If `team.yaml` is missing, write the default file and continue. Never abort the run because of a missing `team.yaml` — the default is always usable.
- If the auto-detected project has no Sub-task issue type (or the equivalent), stop and tell the user before writing `.spec-to-jira.yaml`. This skill requires the three-level hierarchy.
- If the auto-detected project has no "blocks" link type, write `.spec-to-jira.yaml` with both `link_types` fields empty and continue — Phase 2 will skip link creation and warn the user in the summary, but everything else still works.
- If the user **deletes** `plano-<slug>.md` during the pause, Phase 2 cannot proceed — the plan is gone. When the user types `ok`, tell them the file is missing and stop. The user can re-invoke the skill to start over.
- If the plan file is **malformed** when Phase 2 re-reads it (missing H1, broken metadata header, empty Stories section, mismatched `Jira project` value), stop and tell the user exactly which section is malformed. Do not guess. A missing `### Pendências PM` or `### Dependências` is not malformed — those subsections are optional.
- If the plan file reads `Status: applied`, re-running is an **idempotent re-sync**, not a duplicate-creating push. Phase 2 classifies each line as skip / update / create / orphan and only writes what is necessary; an unchanged file produces zero Jira writes. No re-confirmation prompt is needed. See "Phase 2 — create or idempotently re-sync …" and step 8a for the full flow. If `Status:` reads anything other than `pending review` or `applied` (typo, partial edit), stop and tell the user the file is malformed.
- If Epic creation **or update** fails for one milestone, do not abort the run — leave the Epic heading otherwise intact, append ` — ❌ <reason>` to it, skip Story / Sub-task / link creation for that Epic, and continue with the next Epic. The skill never deletes already-created Epics to "match" the file state.
- If Story creation **or update** fails partway through, do not roll back created issues (the skill never deletes). Continue to Sub-task processing for whatever Stories did succeed within the same Epic. Update `plano-<slug>.md` with whatever was created / updated so the user can see the partial state, and clearly mark which user stories failed (append ` — ❌ <reason>` to the failed line; leave any `[TBD]` placeholder in place when the failure was a create, or leave the existing key in place when the failure was an update) so the user can fix and re-run.
- If Sub-task creation **or update** fails for a particular line, log it inline (append ` — ❌ <reason>` to the line; for create failures leave the `[TBD]` placeholder in place and write `❌ <reason>` in the corresponding matrix cell instead of `✅ <citation>` or `✅ pm sub-task ...`; for update failures keep the existing key and the existing matrix cell content) and move on. Do not abort the whole run for a single Sub-task failure.
- If a `getJiraIssue` divergence check fails for an existing key (e.g., the issue was deleted in Jira between runs, or the MCP returned a transient error), treat the line as **failed**: append ` — ❌ <reason>` to the line, do not attempt the update, do not re-create, and surface the failure in step 13. The user can re-run after fixing the underlying issue. Do not silently skip — silent skips would mask a real Jira deletion.
- If step 8a's orphan-discovery JQL search fails for a particular Epic, report `orphan discovery failed for <EPIC_KEY>` in step 13 and continue. The current run's create / update / skip behavior is unaffected — only the orphan column is unreliable for that Epic.
- If a sub-task is **orphaned** (its parent annotation does not resolve to any Story in the same Epic), append ` — ⚠️ orphan parent` to the line, leave the `[TBD]` placeholder in place, skip creation, and surface the orphan in the chat summary. Do not auto-fix by reassigning across Epics.
- If a Pendências PM line's `story #N` reference does not resolve in the same Epic, also flag as orphan, append ` — ⚠️ orphan parent`, skip creation, and surface in the summary.
- If a Pendências PM line is left unchecked (the user did not decide), do not create a Sub-task, leave the `❓` in the matrix on the rewrite, and flag it in the summary as an unresolved gap so the user knows there is a pending decision.
- If a Dependências line's blocker or blocked reference does not resolve to a Sub-task in the same Epic, the line is orphaned: append ` — ⚠️ orphan reference`, skip the link, and surface in the summary. Do not reach across Epics.
- If link creation fails for a particular checked Dependências line (Jira returned an error, link type rejected, etc.), append ` → ❌ <reason>` to the line, leave the `[x]` checkbox in place so the user knows it was attempted, and flag the failure in the summary. Do not abort the whole run for a single link failure.
- If `.spec-to-jira.yaml` has no `blocks` link type configured, skip step 11b entirely, append ` — ⚠️ skipped: no blocks link type` to every checked Dependências line during the step 12 rewrite, and surface the global warning in step 13.
- If the same Dependências link appears twice (duplicate line or duplicate after resolution), create it only once and flag the duplicate in the summary.

## Out of scope

- **Deleting Jira issues** when the corresponding line is removed from the `.md`. Orphaned Jira issues (key exists in Jira, line removed from file) are reported at the end of the run in the `## Orphans` block and the chat summary; the user closes, deletes, or re-links them in Jira directly. The skill never deletes Jira data automatically.
- **Touching `status` or `assignee`** on any **existing** Jira issue. These fields are managed by the team in Jira, not by the skill. The `assignee` field is set **only on initial creation** (when `team.yaml` defines `assignees.story_assignee` or `assignees.stack_overrides` or a stack-level `assignee`), and is never updated or cleared on subsequent re-runs. Only the fields explicitly listed as "mutable" in Phase 2 (summary, description, labels, parent / Epic Link on create, and link relations) are ever written by the skill on existing issues. Components are written on create only.
- **Three-way merging** of edits that originated outside the skill. If a teammate edits a Story title directly in Jira and the `.md` was untouched between runs, the next idempotent re-run will overwrite Jira back to whatever the `.md` says (the `.md` is the source of truth). The skill does not detect or warn about external Jira edits, beyond surfacing them implicitly through the per-field divergence check.
- **Removing existing Jira links** when the user unchecks or deletes a previously-applied Dependências line. The existing link stays in Jira; step 8a surfaces it as a `Link orphan` so the user can remove it manually.
- **Re-parenting Stories or Sub-tasks across Epics** when the file already has real keys. The first run honors the file's structure; once a Story or Sub-task is in Jira, moving its line to a different Epic in the file does not move the Jira issue. Phase 2 surfaces the divergence in the summary but does not act on it.
- **Time-travel / undo**. The `Histórico de execuções` log is informational only — there is no way to "replay" a previous run's state from the log. Use Jira's own history for that.
- Tech specs organized per user story. Only tech specs organized per stack are supported.
- **Creating Jira `blocks` links between Epics**. The per-Epic `Depends on:` list is persisted in the plan file and copied into each Epic's Jira description for visibility, and the link type names are persisted in `.spec-to-jira.yaml` and used for Sub-task ↔ Sub-task links, but the skill does **not** create any Jira link between Epics. A future version will turn the Epic-level `Depends on:` into real links.
- **Cross-Epic Sub-task ↔ Sub-task links**. The Dependências subsection only operates within a single Epic; a tech spec match that crosses milestones is recorded as a free-text note (prefixed `(cross-epic, not auto-created)`) on the later Epic and not turned into a Jira link. A future version will support cross-Epic Sub-task links.
- Auto-refresh of the PRD or tech spec content. Once the source documents are captured in Phase 1 step 2, the skill operates on that snapshot. If the source document changes later, the user must re-run the skill to pick up the change. Idempotent re-runs on the same `plano-<slug>.md` do **not** re-fetch the PRD or tech spec.
- Live syncing between the plan file and Jira after Phase 2 completes — edits to the plan file only propagate to Jira on the **next** `/spec-to-jira` invocation that picks up this plan. There is no daemon or watcher.
