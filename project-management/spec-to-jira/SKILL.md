---
name: spec-to-jira
description: Take a PRD and a Type-1 tech spec — each supplied as a local `.md` path, a Google Docs URL, a Jira issue key, or pasted content — detect milestone boundaries in the PRD, derive user stories per milestone, perform an inverted scan across stack sections, write a `plano-<slug>.md` plan file in the cwd structured as `## Epic N — <name>` sections (each with its own stories, matrix, sub-tasks, and dependencies), and pause for human review. When the user edits the file and types `ok`, the skill re-reads the `.md` from disk as the source of truth and creates one Epic per milestone in Jira via the Atlassian MCP, plus one Story per user story under the right Epic and one Sub-task per covered (story × stack) cell. Milestones are detected from explicit `## Milestone X` headings when present, inferred from PRD structure otherwise. Stack vocabulary lives in `~/.config/spec-to-jira/team.yaml`; per-cwd Jira target lives in `./.spec-to-jira.yaml` (auto-detected via MCP on first run). On invocation, if a `plano-*.md` already exists in the cwd, the skill asks whether to continue that plan or create a new one. Records source identifiers, a snapshot timestamp, Epic keys per milestone, Story keys, and a coverage matrix per Epic. Invoking with no arguments triggers interactive prompts for both sources. Use when user wants to bootstrap a multi-milestone Jira hierarchy with stack-split sub-tasks from a PRD and a tech spec, with a human checkpoint between planning and Jira creation.
---

# Spec to Jira (slice 4 — multi-milestone, N Epics per PRD)

Take a PRD and a Type-1 tech spec — each accepted as a local `.md` path, a Google Docs URL, a Jira issue key, or content pasted directly into chat — detect milestone boundaries in the PRD, derive a list of user stories **per milestone**, perform an inverted scan of the tech spec to build a coverage matrix (user story × stack) **per milestone**, write a `plano-<slug>.md` plan file in the current working directory structured as `## Epic N — <name>` sections, and **pause** for the user to review and edit it. When the user types `ok`, the skill re-reads `plano-<slug>.md` from disk and treats it as the source of truth: it creates one Epic per milestone, plus the Stories and Sub-tasks under the right Epic, in Jira via the Atlassian MCP exactly as the file now describes them, including any edits the user made during the pause — including stories moved between Epics or whole Epics renamed, added, or deleted.

This slice extends slice 3 by detecting milestone boundaries in the PRD and producing one Epic per milestone in Jira, with the plan file structured as `## Epic N — <name>` sections. Detection prefers explicit `## Milestone X` headings in the PRD; when those are absent, the skill infers milestone groupings from PRD structure and prose. The plan file's per-Epic layout lets the user move user stories between Epics, rename Epics, add new Epics, or delete an Epic entirely by editing the file before approval — the change propagates to the Jira issues that get created. Idempotent re-runs and creation of Jira "blocks" links between Epics remain out of scope.

## Invocation

On every invocation, **before reading any source input**, the skill scans the current working directory for files matching `plano-*.md`. The presence of such a file is the signal that a prior run paused for review.

- **No matches**: proceed directly to Phase 1 (input resolution → matrix → write plan).
- **One or more matches**: list them in chat (filename and last-modified time) and ask the user:
  > Found <N> existing plan file(s): <list>. Continue with one of these, or create a new plan?
  - If the user picks one of the existing files, **skip Phase 1** and jump straight to Phase 2 with that file as the input. Do not re-read the PRD or tech spec — the plan file is now the source of truth. Do not re-prompt for sources.
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

Stack definitions used by the coverage matrix and Sub-task creation. Read once per run. If the file is missing, the skill writes a default file with `cloud`, `web`, `data`, and the virtual `pm` stack, then continues using that default.

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
```

Notes on `team.yaml`:

- The matrix columns and the Sub-task iteration follow the order in which stacks appear in `team.yaml`. Reordering the list reorders the matrix.
- The `pm` stack is **virtual**: it represents project-management work (rollout coordination, comms, milestone tracking) rather than an engineering stack. It uses the same Sub-task mechanic — the inverted scan can map "pm" coverage to tech-spec sections describing rollout / comms / coordination work, or leave the cell empty for stories with no pm component.
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
- `link_types` carry the inward / outward names for the "blocks" relationship. This slice does **not** yet create those links, but persisting the names now means future slices can use them without re-detecting.
- `cloud_url` is optional but recommended — useful for human-readable summaries and direct links in `plano-<slug>.md`.
- Once the file exists, the skill reads it silently. Do not prompt the user for any value already in the file.

### Auto-detect (first run in a new cwd)

When `./.spec-to-jira.yaml` is missing, the skill runs the following sequence **before** reading the PRD or the tech spec:

1. Call `getAccessibleAtlassianResources` (Atlassian MCP) to list the cloud resources the user has access to. If exactly one is returned, use it without asking. Otherwise, show the user the list (id + name + url) and ask which to use.
2. Call `getVisibleJiraProjects` on the chosen cloud ID. Show the user the list (project key + name) and ask which to use.
3. Call `getJiraProjectIssueTypesMetadata` (falling back to `getJiraIssueTypeMetaWithFields` if needed) for the chosen project. Map the returned issue type names to the `epic`, `story`, and `sub_task` slots. If the project does not expose an Epic, a Story, or a Sub-task equivalent, stop and tell the user — this skill needs all three.
4. Call `getIssueLinkTypes` to fetch the project's link type names. Find the "blocks" relationship and record the inward name as `blocked_by` and the outward name as `blocks`. If the project has no "blocks" link type, leave both fields empty in the file and continue — the skill does not need them in this slice.
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

The skill runs in two phases separated by a human review pause. Phase 1 builds the plan file; the pause lets the user edit it; Phase 2 creates the Jira issues from whatever the file now contains. If the user picked an existing `plano-*.md` at the invocation check, **skip directly to Phase 2** with that file. Otherwise, run Phase 1 first.

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

Also capture, per milestone, any **dependencies** declared in the PRD — e.g. "Milestone 2 depends on Milestone 1", "blocked by the auth rollout in Phase 1", "requires X to ship first". Record them as a list of milestone numbers / names that the current milestone depends on. If a stated dependency points at a milestone not present in the PRD, keep it as a free-text note; do not invent dependencies. The user can edit the dependency list during review. This slice does **not** create the corresponding Jira "blocks" links — the dependency list is informational and persisted in the plan file and in each Epic's Jira description for a future slice (see "Out of scope").

#### 4. Identify the tech spec's stack sections (Type-1 organization)

The tech spec is assumed to be **Type-1 organized**: top-level sections grouped per stack, each section describing the implementation work needed for that stack across the whole product. Map headings in the tech spec to the stacks defined in `team.yaml`:

- **cloud**: backend, services, APIs, jobs, infra, deployment, observability that lives server-side.
- **web**: client-side application, UI, components, routing, client state, browser concerns.
- **data**: schema, migrations, ETL/ELT, pipelines, analytics, data contracts, warehousing.
- **pm** (virtual): rollout plan, comms, milestone tracking, stakeholder coordination, training, documentation hand-off.
- Any other custom stacks the user has added in `team.yaml`: use the stack name and label as the mapping signal, plus any context the user has documented in the tech spec.

Capture, per stack, the list of headings (with their heading text and an anchor or section number) that belong to that stack. Headings that do not clearly belong to any configured stack are ignored in the inverted scan — they will not appear in the matrix. If a tech spec heading legitimately spans two stacks, count it under both.

If none of the configured stacks are represented in the tech spec at all (e.g., the tech spec is empty or unrelated), stop and tell the user — there is nothing to scan.

#### 5. Inverted scan → coverage matrix per milestone

For each derived user story (across **all** milestones), scan **all configured stack sections** of the tech spec and identify the specific tech spec section(s) that describe implementation work needed to deliver that story. The scan is **inverted**: the user story drives the lookup, and the tech spec is the lookup target. The tech spec itself does not need to be split per milestone — the same tech spec sections may legitimately cover stories from multiple milestones.

The output is **one coverage matrix per milestone**, with one row per user story in that milestone and one column per stack defined in `team.yaml` (in the order they appear there). Each cell is either:

- **Covered**: a non-empty list of tech spec section citations (heading text + anchor or section number) that describe the relevant implementation work for that story in that stack.
- **Empty**: no tech spec section in that stack describes work needed for this story. Leave the cell blank in the matrix and create no Sub-task for that cell.

A story can be covered in zero, one, several, or all stacks. Do not invent coverage to fill cells — empty cells are valid and informative.

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
2. [TBD] As a <actor>, I want <feature>, so that <benefit>
...

### Matriz de cobertura

| História | <stack_1> | <stack_2> | ... | <stack_N> |
|---|---|---|---|---|
| [TBD] <feature_1> | ✅ <citation> | ✅ <citation> |  |  |
| [TBD] <feature_2> |  | ✅ <citation> | ✅ <citation> |  |
...

### Sub-tasks

- [TBD] [<stack_1>] <Story summary_1> — parent [TBD] (story #1) — cites <tech spec section>
- [TBD] [<stack_2>] <Story summary_1> — parent [TBD] (story #1) — cites <tech spec section>
...

## Epic 2 — <Milestone 2 name>

Depends on: Epic 1

### Stories

1. [TBD] As a <actor>, I want <feature>, so that <benefit>
...

### Matriz de cobertura

| História | <stack_1> | <stack_2> | ... | <stack_N> |
|---|---|---|---|---|
| [TBD] <feature_X> | ✅ <citation> |  |  |  |
...

### Sub-tasks

- [TBD] [<stack_1>] <Story summary_X> — parent [TBD] (story #1) — cites <tech spec section>
...
```

Notes on the file format:

- The top-level `Status: pending review` line is the marker that this file is a Phase-1 artifact waiting for Phase 2. Phase 2 rewrites it to `Status: applied` when creation completes. There is exactly one `Status:` line in the whole file — it is **not** repeated per Epic.
- Each milestone gets its own `## Epic N — <name>` section. Epic numbering is 1-based and follows the order milestones appear in the PRD. The Epic **name** comes from the milestone name detected in step 3.
- The `Depends on:` line directly under each Epic heading lists the Epics this one depends on, by Epic number or name (e.g., `Depends on: Epic 1`, `Depends on: Epic 1, Epic 2`, `Depends on: (none)`). The list is informational — this slice does not create Jira "blocks" links — but it is persisted in the file and copied into each Epic's Jira description in Phase 2. The user may edit it during review or set it to `(none)`.
- Inside each Epic section, stories are numbered 1-based **within that Epic**. The same story number can appear in two different Epics — `(story #1)` in Sub-tasks always refers to story #1 of the **same enclosing Epic**.
- In the Sub-tasks subsection under an Epic, the `(story #N)` annotation after each `parent [TBD]` records which story (by 1-based position in the **enclosing Epic's** Stories subsection) the sub-task belongs to. The user may delete that annotation if they prefer; Phase 2 will then fall back to matching the parent by Story summary text within the same Epic.
- The matrix columns are the stack names from `team.yaml`, in the order they appear there. Each Epic has its own matrix listing only the stories in that Epic.
- Cells use `✅ <citation>` when covered, with the citation being the tech spec heading text (or section number) you used. If a cell is covered by multiple tech spec sections, separate them with `; ` inside the same cell. Empty cells stay literally empty between the pipes — do not write `—` or `n/a`.
- The matrix is informational. The **Stories** subsection drives Story creation in Phase 2; the **Sub-tasks** subsection drives Sub-task creation. Edits to the matrix alone do not change what gets created in Jira — only edits to the Stories and Sub-tasks subsections do. If the user wants to add a sub-task that is not currently in the list, they add a line to the right Epic's Sub-tasks subsection.
- **Moving a story between Epics**: the user cuts the story line from one Epic's `### Stories` and pastes it into another Epic's `### Stories`. They also move the corresponding row in `### Matriz de cobertura` and **all** matching lines in `### Sub-tasks` of the same Epic. Phase 2 will then create the story under the destination Epic. If the user moves the story but forgets to move its sub-tasks, those sub-tasks become orphaned (their `parent [TBD] (story #N)` annotation does not match a story in the same Epic) — Phase 2 will warn and skip them rather than reassign them silently across Epics.
- **Renaming an Epic**: edit the text after ` — ` on the `## Epic N — <name>` line. Do not change the `N` — Epic numbers are positional. If the user renumbers Epics by hand, Phase 2 still treats them in the order they appear in the file.
- **Deleting an Epic**: delete the entire `## Epic N — <name>` heading and its `Depends on`, `### Stories`, `### Matriz de cobertura`, and `### Sub-tasks` subsections. Phase 2 will not create that Epic, its Stories, or its Sub-tasks.
- **Adding a new Epic**: append a new `## Epic N — <name>` section at the end of the file with the next integer in sequence. Use `[TBD]` for the keys on every line inside. Include a `Depends on:` line (use `(none)` if there are no dependencies) and the four subsections (`### Stories`, `### Matriz de cobertura`, `### Sub-tasks`).

#### 7. Pause for review

After writing the plan file, print in chat exactly one message and **stop**. Do not continue to Phase 2 in the same turn — the user must have a chance to open the file in their editor first. Use this wording (paraphrasable, but the meaning must be preserved):

> Plan written to `plano-<slug>.md`. Edit the file, type `ok` when ready.

Do not also print a preview of the matrix or the stories in chat — the file *is* the preview. The user reviews directly in their editor.

### Pause (human checkpoint)

The skill is now idle. The user edits `plano-<slug>.md` outside of chat (in their editor, on disk) and returns to chat to indicate readiness. The next user message ends the pause:

- Replies that count as **approval** (resume Phase 2): `ok`, `OK`, `okay`, `go`, `proceed`, `continue`, `aprovado`, `pode ir`, or any obvious paraphrase of approval. When in doubt, ask once to confirm rather than guessing.
- Replies that count as **cancel** (stop, leave the file in place): `cancel`, `abort`, `stop`, `cancela`, `parar`.
- Anything else (questions, follow-up requests, asking the skill itself to edit the file): respond and continue waiting. Do not advance to Phase 2 until the user explicitly approves.

If the user cancels, leave `plano-<slug>.md` on disk untouched (still `Status: pending review`). A future `/spec-to-jira` invocation will detect it via the invocation check and offer to continue it.

### Phase 2 — create Jira issues from the plan file

#### 8. Re-read the plan file from disk

Read `plano-<slug>.md` fresh from disk with the Read tool. **Do not** rely on in-memory state from Phase 1 — the user has had a chance to edit the file, and any divergence between in-memory state and the file on disk must resolve in favor of the file. The same rule applies when Phase 2 was entered via the invocation check on a pre-existing plan file.

Parse:

- **PRD title** — the H1 line at the top of the file.
- **Metadata header** — `Source PRD:`, `Source tech spec:`, `Snapshot:`, `Generated:`, `Jira project:`. The `Jira project` value must match `project_key` in `./.spec-to-jira.yaml`; if it does not, stop and tell the user the file targets a different project than the cwd is configured for.
- **Status** — should read `pending review`. If it reads `applied`, the plan has already been pushed to Jira on a prior run; warn the user that re-running will create duplicate issues and ask for an explicit re-confirmation before proceeding.
- **Epics** — every `## Epic N — <name>` heading, in the order they appear in the file. For each Epic, parse:
  - The Epic's **name** — the text after ` — ` on the heading line.
  - The Epic's **Jira key** — for a fresh plan the heading has no key; for a partially-completed prior run the heading reads `## Epic N — <name> [<KEY>]`. Record the key when present, treat its absence as "needs creation".
  - The **Depends on** line directly under the heading — a comma-separated list of Epic numbers / names, or `(none)`. Record it verbatim for the Jira description in step 9 and for the rewrite in step 12. This slice does not create Jira links.
  - The Epic's **Stories** — every `<n>. [<key>] <story sentence>` line in the Epic's `### Stories` subsection, in the order they appear. Each line becomes one Story belonging to this Epic. If a line already carries a real key, capture it for parenting and skip creation.
  - The Epic's **Sub-tasks** — every `- [<key>] [<stack>] <summary> — parent [<parent_key>] (story #N) — cites <citation>` line in the Epic's `### Sub-tasks` subsection. Parse the `[<stack>]` tag to look up the corresponding stack in `team.yaml`. Resolve the parent Story key in this order: explicit Story key when present, story index `N` within the **same Epic**, parent summary text matched against the Stories of the **same Epic**. If none of those resolve to a Story in the same Epic, the sub-task is **orphaned** — flag it but do not reassign across Epics (see step 11).
  - The Epic's **Matriz de cobertura** — informational only, not parsed for issue creation. Phase 2 rewrites it in step 12 to reflect the real Story keys.

If a line in an Epic's Stories subsection was deleted by the user, do not create that Story — and skip every Sub-task in the same Epic whose parent annotation points at the deleted story. If a line in a Sub-tasks subsection was deleted, do not create that Sub-task. If an entire `## Epic N — <name>` section (heading + subsections) was deleted, skip the Epic, all its Stories, and all its Sub-tasks. **This is how the user vetoes items during review**: just delete the line (or the whole Epic section).

If a line was edited (story title rewritten, sub-task title rewritten, citation reworded, Epic renamed, dependency list changed), use the **edited** text when creating the Jira issue. The file is authoritative.

If the file is malformed (missing H1, broken metadata header, zero `## Epic N — <name>` sections, or an Epic section that has Sub-task lines but no `### Stories` subsection), stop and tell the user exactly which section is malformed. Do not guess. A missing `### Matriz de cobertura` is **not** malformed — it is regenerated in step 12 from the Stories and Sub-tasks subsections of the same Epic.

#### 9. Create one Epic per milestone

Iterate the `## Epic N — <name>` sections in the order they appear in the file. For each Epic still present (i.e., not deleted by the user), use the Atlassian MCP to create one Epic in the configured project. Call `createJiraIssue` with:

- `cloudId`: `cloud_id` from `.spec-to-jira.yaml`.
- `projectKey`: `project_key` from `.spec-to-jira.yaml`.
- `issueTypeName`: `issue_types.epic` from `.spec-to-jira.yaml`.

For each Epic:

- **Summary**: the name on the `## Epic N — <name>` heading — i.e., whatever the user left there during review (renames are honored).
- **Description**: a short pointer back to `prd_identifier`, a pointer to `tech_spec_identifier`, the snapshot timestamp, a one-paragraph summary derived from the PRD section for that milestone, and the Epic's `Depends on:` list verbatim (so a reviewer can see the declared dependencies even though they are not Jira link issues yet).

If an Epic heading already carries a real Jira key (resumed plan), skip creation and use the existing key as the parent for that Epic's Stories.

Capture the returned Jira key for each Epic (e.g., `PROJ-1234`).

If Epic creation fails for one milestone, do not abort the whole run — record the failure (it will be surfaced in step 12) and continue with the next Epic. Stories and Sub-tasks belonging to a failed Epic are skipped in steps 10 and 11.

#### 10. Create one Story per Stories line, parented to its Epic

For each Epic in turn, iterate every line still present in that Epic's `### Stories` subsection and create one Story in the same project (`createJiraIssue` with `issueTypeName` set to `issue_types.story` from `.spec-to-jira.yaml`), linked to **that Epic's** key from step 9. Use whichever mechanism the project supports for parenting a Story to an Epic — the `Epic Link` custom field on classic projects, or the `parent` field on next-gen / team-managed projects. If the first attempt fails because the field is not available, retry with the other mechanism before giving up.

A user who moved a story between Epics (cut from Epic 1's `### Stories`, paste into Epic 2's `### Stories`) before approval will see the Story created under Epic 2's key here — the file's structure drives parenting, not any memory of the original Phase 1 assignment.

The Story summary is derived from the **user-edited** story sentence in the file — extract the `<feature>` clause from the `As a <actor>, I want <feature>, so that <benefit>` pattern when present; otherwise use the full edited sentence as the summary. The description is the full story sentence as written in the file plus any extra context the user kept in the file.

If a Stories line already carries a real Jira key (resumed plan), skip creation and remember the existing key for sub-task parenting. The Story's Epic parent is **not** re-validated for resumed lines — once a Story is in Jira, this slice does not move it between Epics. If the user moved a real-key Story line between Epics in the file, the file and Jira are now out of sync; flag it for the summary in step 13 and leave the Jira parenting alone.

Capture the returned Jira key for each Story and remember which Epic it belongs to (used by step 11 for orphan detection).

#### 11. Create one Sub-task per Sub-tasks line, parented to its Story

For each Epic in turn, iterate every line still present in that Epic's `### Sub-tasks` subsection and create one Sub-task in the same project (`createJiraIssue` with `issueTypeName` set to `issue_types.sub_task` from `.spec-to-jira.yaml`), parented to the resolved Story key from step 10 — where parent resolution is scoped to the **same Epic** (see step 8 for the resolution order). For each Sub-task:

- **Summary**: the text on the sub-task line between the `[<stack>]` tag and the ` — parent ` separator. This is the user-editable title — use whatever the user wrote, do not regenerate it from the Story summary or the stack `title_prefix`. The `title_prefix` only seeded the line in Phase 1; once written to disk, the user owns it.
- **Labels**: include the matching stack's `label` value from `team.yaml` — e.g. `stack:cloud`. Do not add other labels in this slice.
- **Component**: if the matching stack has a non-empty `component` field in `team.yaml`, attach it as a Jira component. If the project does not have that component configured, skip the component silently and continue with the Sub-task creation.
- **Description**: state which Story this Sub-task implements (by Story key) and cite the relevant tech spec section as written on the line. Also include `tech_spec_identifier` so a reviewer can locate the original tech spec document regardless of format.

If a sub-task line already carries a real Jira key (resumed plan), skip creation and remember the existing key.

If a sub-task's parent annotation does not resolve to any Story line in the same Epic (an **orphan** — typically because the user moved the parent story to another Epic but forgot to move the sub-task), warn the user in the chat summary, append ` — ⚠️ orphan parent` to the line during the step 12 rewrite, and skip creation. Do not reassign the sub-task across Epics on the user's behalf.

Iterate Sub-tasks in this order: outer loop = Epics in file order; inner loop = Sub-tasks in line order within each Epic. Capture the returned Jira key for each Sub-task.

#### 12. Update the plan file with the real Jira keys

Rewrite `plano-<slug>.md` in place to reflect the created Jira issues. Specifically:

- On each `## Epic N — <name>` heading whose Epic was created, append ` [<EPIC_KEY>]` so the heading reads `## Epic N — <name> [<EPIC_KEY>]`. If an Epic failed to create, leave its heading untouched and append ` — ❌ <reason>` to the heading line; skip the per-Epic Story / Sub-task replacements for that Epic and leave their `[TBD]` placeholders in place.
- Replace each `[TBD]` placeholder on Story and Sub-task lines with the real Jira key.
- For each Epic, rewrite its matrix's first column (`| [TBD] <feature> |` becomes `| <STORY_KEY> <feature> |`) and the `parent [TBD]` references in its Sub-tasks subsection so they point at the created Story keys within the same Epic.
- Change the top-level `Status:` header from `pending review` to `applied`.
- If a particular Story or Sub-task failed to create, leave its `[TBD]` placeholder in place and append ` — ❌ <reason>` to that line so the user can see which items failed and re-run them later. Use `❌ <reason>` inside matrix cells for failed sub-tasks (instead of `✅ <citation>`).
- For orphaned sub-tasks (parent in another Epic), keep the `[TBD]` placeholder and append ` — ⚠️ orphan parent` per step 11.

Do not change anything else in the file — preserve every user edit to titles, citations, dependencies, Epic order, and story order.

#### 13. Summarize in chat

Print a concise summary, grouped per Epic. For each Epic: its key + name, the count of created stories under it, the count of created sub-tasks under it (broken down per stack), the list of created story keys, and the list of created sub-task keys. After the per-Epic breakdown, include the total counts across all Epics and the path to the updated `plano-<slug>.md`. Include a direct link to each Epic in Jira if the MCP response includes one or if `cloud_url` is set in `.spec-to-jira.yaml`.

Flag explicitly in the summary:

- Any **Epic that failed to create** (and the resulting skipped Stories / Sub-tasks under it).
- Any **orphaned sub-task** (parent annotation does not resolve within the same Epic).
- Any **real-key Story line the user moved between Epics** — note that Jira and file are now out of sync because this slice does not re-parent existing Stories.

If `.spec-to-jira.yaml` was newly written by the auto-detect flow on this run, mention it explicitly so the user knows to review and commit the file.

## Failure handling

- If the Atlassian MCP is not configured or authentication fails — at auto-detect time, at Jira-key fetch time, at Epic creation, or anywhere else — stop and tell the user how to authenticate. Do not write a partial `.spec-to-jira.yaml`, and do not flip `Status:` in the plan file.
- If a **Jira issue key** input refers to a missing or inaccessible issue, stop and tell the user the issue key was not found. Do **not** fall back to paste-or-path — the Atlassian MCP is needed later anyway, so masking its failure here only delays the same error.
- If a **Google Docs URL** input cannot be fetched (Drive MCP missing, auth failure, access denied, empty body), trigger the paste-or-path fallback described in "Input resolution". Do not abort.
- If a **local `.md` path** input does not exist or is not a `.md` file, stop and tell the user. Do not auto-fall-back — the user typed a path, not a URL, so paste-or-path would be surprising here.
- If `team.yaml` is missing, write the default file and continue. Never abort the run because of a missing `team.yaml` — the default is always usable.
- If the auto-detected project has no Sub-task issue type (or the equivalent), stop and tell the user before writing `.spec-to-jira.yaml`. This skill requires the three-level hierarchy.
- If the user **deletes** `plano-<slug>.md` during the pause, Phase 2 cannot proceed — the plan is gone. When the user types `ok`, tell them the file is missing and stop. The user can re-invoke the skill to start over.
- If the plan file is **malformed** when Phase 2 re-reads it (missing H1, broken metadata header, empty Stories section, mismatched `Jira project` value), stop and tell the user exactly which section is malformed. Do not guess.
- If the plan file reads `Status: applied`, the plan was already pushed to Jira on a prior run. Warn the user that re-running will create duplicate issues and require an explicit re-confirmation before proceeding. The skill does not deduplicate.
- If Epic creation fails for one milestone, do not abort the run — leave the Epic heading untouched, append ` — ❌ <reason>` to it, skip Story and Sub-task creation for that Epic, and continue with the next Epic. This slice never deletes already-created Epics to "match" the file state.
- If Story creation fails partway through, do not roll back created issues (this slice never deletes). Continue to Sub-task creation for whatever Stories did succeed within the same Epic. Update `plano-<slug>.md` with whatever was created so the user can see the partial state, and clearly mark which user stories failed (append ` — ❌ <reason>` to the failed line and leave its `[TBD]` placeholder in place) so the user can fix and re-run.
- If Sub-task creation fails for a particular line, log it inline (append ` — ❌ <reason>` to the line, leave the `[TBD]` placeholder in place, and write `❌ <reason>` in the corresponding matrix cell instead of `✅ <citation>`) and move on. Do not abort the whole run for a single Sub-task failure.
- If a sub-task is **orphaned** (its parent annotation does not resolve to any Story in the same Epic), append ` — ⚠️ orphan parent` to the line, leave the `[TBD]` placeholder in place, skip creation, and surface the orphan in the chat summary. Do not auto-fix by reassigning across Epics.

## Out of scope for this slice

- Full idempotent re-runs — re-running on an `applied` plan will warn the user but otherwise create duplicate Jira issues if the user confirms. The only resume case this slice handles natively is a paused plan where some lines / Epic headings already carry real keys (e.g., one Epic was created on a prior turn but its Stories were not) — Phase 2 detects real keys per heading and per line and skips creation for those items, but still creates everything else.
- Tech specs organized per user story (Type-2). Only Type-1 (per stack) is supported in this slice.
- Creating "blocks" / "is blocked by" links between Epics. The per-Epic `Depends on:` list is persisted in the plan file and copied into each Epic's Jira description for visibility, and the link type names are persisted in `.spec-to-jira.yaml`, but this slice does **not** create any Jira link issue. A future slice will turn `Depends on:` into real links.
- Re-parenting Stories or Sub-tasks across Epics when the file already has real keys. The first run honors the file's structure; once a Story is in Jira, moving its line to a different Epic in the file does not move the Jira issue. Phase 2 surfaces this divergence in the summary but does not act on it.
- Auto-refresh: once the PRD or tech spec content is captured in Phase 1 step 2, the skill operates on that snapshot. If the source document changes later, the user must re-run the skill to pick up the change.
- Live syncing between the plan file and Jira after Phase 2 completes — edits to the plan file after `Status:` flips to `applied` do not propagate back to Jira. The user would need a fresh run.
