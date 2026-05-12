---
name: spec-to-jira
description: Take a PRD and a Type-1 tech spec — each supplied as a local `.md` path, a Google Docs URL, a Jira issue key, or pasted content — derive user stories, perform an inverted scan across stack sections, and create one Epic + one Story per user story + one Sub-task per covered (story × stack) cell in Jira via the Atlassian MCP. Stack vocabulary lives in `~/.config/spec-to-jira/team.yaml`; per-cwd Jira target lives in `./.spec-to-jira.yaml` (auto-detected via MCP on first run). Records source identifiers, a snapshot timestamp, the Epic key, Story keys, and a "Matriz de cobertura" table in a `plano-<slug>.md` file. Invoking with no arguments triggers interactive prompts for both sources. Use when user wants to bootstrap a Jira hierarchy with stack-split sub-tasks from a PRD and a tech spec.
---

# Spec to Jira (slice 8 — multi-format input)

Take a PRD and a Type-1 tech spec — each accepted as a local `.md` path, a Google Docs URL, a Jira issue key, or content pasted directly into chat — derive a list of user stories from the PRD, perform an inverted scan of the tech spec to build a coverage matrix (user story × stack), and create the corresponding Epic + Stories + Sub-tasks in Jira via the Atlassian MCP. Stack definitions and the Jira target are read from YAML configuration files. Record the source identifiers, a snapshot timestamp, the created keys, and the coverage matrix in a `plano-<slug>.md` file in the current working directory.

This slice extends slice 7 by accepting input in formats beyond local `.md`: Google Docs URLs (fetched via the Google Drive MCP, with a paste-or-path fallback when the MCP is unavailable), Jira issue keys (fetched via the Atlassian MCP), pasted-in-chat content, and a no-argument interactive mode that prompts for both sources. PRD and tech spec sources are independent — they can be mixed (e.g., PRD as a Drive URL, tech spec as a Jira key). The resolved source identifiers are written into the metadata header of `plano-<slug>.md` so the artifact is traceable back to the originals. Review-pause workflow, multi-milestone handling, and idempotent re-runs remain out of scope.

## Inputs

The skill accepts the PRD and the tech spec independently — each can be supplied in any of the supported formats, and the two formats do not need to match.

- **PRD source** (required): the **first** positional argument: `/spec-to-jira <prd-source> <tech-spec-source>`. May be a local `.md` path, a Google Docs URL (any `https://docs.google.com/document/...` link), or a Jira issue key (e.g., `PROJ-123`). If the argument is omitted, the skill enters interactive mode for the PRD (see "Input resolution" below).
- **Tech spec source** (required): the **second** positional argument. Same supported formats as the PRD. If omitted, the skill enters interactive mode for the tech spec. If only one argument is given and its role is ambiguous, ask the user to clarify which is the PRD and which is the tech spec.
- **No-argument interactive mode**: invoking `/spec-to-jira` with **no arguments** enters interactive mode for both sources. For each source, ask the user to either paste content, give a local `.md` path, give a Google Docs URL, or give a Jira issue key. PRD comes first, tech spec second.
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

### 1. Load configuration

- Read `~/.config/spec-to-jira/team.yaml`. If missing, write the default file (cloud / web / data / pm) and continue with that default. Echo the path to the user so they know where to edit it.
- Read `./.spec-to-jira.yaml`. If missing, run the auto-detect flow (see above) and write the file. If present, read it silently — do not prompt the user for any value it already carries.

### 2. Resolve the PRD and the tech spec

- Run the "Input resolution" algorithm above on the PRD source argument (or the interactive prompt response when no argument was given). The result is `(prd_content, prd_identifier)`.
- Run the same algorithm on the tech spec source argument. The result is `(tech_spec_content, tech_spec_identifier)`.
- Capture a snapshot timestamp at this point as an ISO 8601 string (e.g., `2026-05-11T14:23:07Z`). This is the value that goes into the plan file's metadata header so a reader can tell when the source documents were captured.
- Pick a slug for the plan filename from the PRD's first H1 title (lowercased, kebab-case, ASCII-only). If there is no H1, derive the slug from `prd_identifier`: take the basename for a path, the document ID segment for a Google Docs URL, the lowercased issue key for a Jira key, or `pasted` for pasted content. Still kebab-case and ASCII-only.

### 3. Derive user stories from the PRD

Read the PRD and derive a list of well-defined user stories. Prefer stories that are already written in `As a <actor>, I want <feature>, so that <benefit>` form; if the PRD lacks them, synthesize the list from the problem statement, solution, and acceptance criteria sections. Each story must be a single, independently-meaningful unit of user value — do not stitch multiple goals into one story, and do not split a single goal across multiple stories.

### 4. Identify the tech spec's stack sections (Type-1 organization)

The tech spec is assumed to be **Type-1 organized**: top-level sections grouped per stack, each section describing the implementation work needed for that stack across the whole product. Map headings in the tech spec to the stacks defined in `team.yaml`:

- **cloud**: backend, services, APIs, jobs, infra, deployment, observability that lives server-side.
- **web**: client-side application, UI, components, routing, client state, browser concerns.
- **data**: schema, migrations, ETL/ELT, pipelines, analytics, data contracts, warehousing.
- **pm** (virtual): rollout plan, comms, milestone tracking, stakeholder coordination, training, documentation hand-off.
- Any other custom stacks the user has added in `team.yaml`: use the stack name and label as the mapping signal, plus any context the user has documented in the tech spec.

Capture, per stack, the list of headings (with their heading text and an anchor or section number) that belong to that stack. Headings that do not clearly belong to any configured stack are ignored in the inverted scan — they will not appear in the matrix. If a tech spec heading legitimately spans two stacks, count it under both.

If none of the configured stacks are represented in the tech spec at all (e.g., the tech spec is empty or unrelated), stop and tell the user — there is nothing to scan.

### 5. Inverted scan → coverage matrix

For each derived user story, scan **all configured stack sections** of the tech spec and identify the specific tech spec section(s) that describe implementation work needed to deliver that story. The scan is **inverted**: the user story drives the lookup, and the tech spec is the lookup target.

The output is a coverage matrix with one row per user story and one column per stack defined in `team.yaml` (in the order they appear there). Each cell is either:

- **Covered**: a non-empty list of tech spec section citations (heading text + anchor or section number) that describe the relevant implementation work for that story in that stack.
- **Empty**: no tech spec section in that stack describes work needed for this story. Leave the cell blank in the matrix and create no Sub-task for that cell.

A story can be covered in zero, one, several, or all stacks. Do not invent coverage to fill cells — empty cells are valid and informative.

### 6. Confirm before writing to Jira

Print:

- The proposed Epic title (use the PRD's H1 or filename slug as the source).
- The numbered list of derived user stories.
- A preview of the coverage matrix: one row per story, with a short citation list per covered cell (heading text is enough at this point). Use the stack names from `team.yaml` as column headers.

Ask the user to confirm before any Jira write happens. Iterate until the user approves. Once approved, do not ask again in the same run.

### 7. Create the Epic

Use the Atlassian MCP to create exactly one Epic in the configured project. Call `createJiraIssue` with:

- `cloudId`: `cloud_id` from `.spec-to-jira.yaml`.
- `projectKey`: `project_key` from `.spec-to-jira.yaml`.
- `issueTypeName`: `issue_types.epic` from `.spec-to-jira.yaml`.

The Epic's summary is the PRD title; the description is a short pointer back to `prd_identifier`, a pointer to `tech_spec_identifier`, the snapshot timestamp, and a one-paragraph summary derived from the PRD.

Capture the returned Jira key (e.g., `PROJ-1234`).

### 8. Create one Story per user story

For each derived user story, create a Story in the same project (`createJiraIssue` with `issueTypeName` set to `issue_types.story` from `.spec-to-jira.yaml`) and link it to the Epic. Use whichever mechanism the project supports for parenting a Story to an Epic — the `Epic Link` custom field on classic projects, or the `parent` field on next-gen / team-managed projects. If the first attempt fails because the field is not available, retry with the other mechanism before giving up.

The Story summary is the user story's `<feature>` clause; the description is the full `As a … I want … so that …` sentence plus any extra context from the PRD that clarifies the story.

Capture the returned Jira key for each Story.

### 9. Create one Sub-task per covered (story × stack) cell

For each non-empty cell in the coverage matrix, create one Sub-task in the same project (`createJiraIssue` with `issueTypeName` set to `issue_types.sub_task` from `.spec-to-jira.yaml`), parented to the corresponding Story (`parent` field set to the Story key). For each Sub-task:

- **Summary**: `<title_prefix> <Story summary>` — e.g. `[cloud] Allow user to reset password`. The prefix comes from the matching stack's `title_prefix` field in `team.yaml`.
- **Labels**: include the matching stack's `label` value from `team.yaml` — e.g. `stack:cloud`. Do not add other labels in this slice.
- **Component**: if the matching stack has a non-empty `component` field in `team.yaml`, attach it as a Jira component. If the project does not have that component configured, skip the component silently and continue with the Sub-task creation.
- **Description**: state which Story this Sub-task implements (by Story key) and cite the relevant tech spec section(s) for this (story, stack) cell. Use the heading text plus an anchor or section number so a reviewer can navigate back to the source. Also include `tech_spec_identifier` so a reviewer can locate the original tech spec document regardless of format.

Iterate stacks in the order they appear in `team.yaml` and stories in the order they appear in the matrix, so the resulting Sub-tasks are created in a predictable sequence.

Capture the returned Jira key for each Sub-task. Do not create Sub-tasks for empty cells.

### 10. Write `plano-<slug>.md`

In the current working directory, write a file named `plano-<slug>.md` with the following structure. The lines under the title are the **metadata header** — it lists the source identifiers (in whatever format the user supplied), the snapshot timestamp captured in step 2, and the Jira project so the artifact is fully traceable back to its inputs.

```
# <PRD title>

Source PRD: <prd_identifier>
Source tech spec: <tech_spec_identifier>
Snapshot: <ISO 8601 timestamp captured in step 2>
Generated: <ISO 8601 timestamp at write time>
Jira project: <project_key>

## Epic

- [<EPIC_KEY>] <Epic summary>

## Stories

1. [<STORY_KEY_1>] As a <actor>, I want <feature>, so that <benefit>
2. [<STORY_KEY_2>] As a <actor>, I want <feature>, so that <benefit>
...

## Matriz de cobertura

| História | <stack_1> | <stack_2> | ... | <stack_N> |
|---|---|---|---|---|
| [<STORY_KEY_1>] <feature> | ✅ <citation> | ✅ <citation> |  |  |
| [<STORY_KEY_2>] <feature> |  | ✅ <citation> | ✅ <citation> |  |
...

## Sub-tasks

- [<SUBTASK_KEY_1>] [<stack_1>] <Story summary> — parent [<STORY_KEY_1>] — cites <tech spec section>
- [<SUBTASK_KEY_2>] [<stack_2>] <Story summary> — parent [<STORY_KEY_1>] — cites <tech spec section>
...
```

Notes on the matrix:

- The columns are the stack names from `team.yaml`, in the order they appear there. If the user has added or removed stacks, the matrix layout follows.
- Cells use `✅ <citation>` when covered, with the citation being the tech spec heading text (or section number) you used. If a cell is covered by multiple tech spec sections, separate them with `; ` inside the same cell.
- Empty cells stay literally empty between the pipes — do not write `—` or `n/a`.
- The matrix lives at the Epic level (one matrix per Epic). Since this slice always produces exactly one Epic, there is exactly one matrix per `plano-<slug>.md`.

If a `plano-<slug>.md` already exists in the cwd, do not overwrite silently — tell the user and ask whether to overwrite or pick a new slug.

### 11. Summarize in chat

Print a concise summary: the Epic key, the count of created stories, the count of created sub-tasks (broken down per stack), the list of story keys, the list of sub-task keys, and the path to the written `plano-<slug>.md`. Include a direct link to the Epic in Jira if the MCP response includes one or if `cloud_url` is set in `.spec-to-jira.yaml`.

If `.spec-to-jira.yaml` was newly written by the auto-detect flow on this run, mention it explicitly so the user knows to review and commit the file.

## Failure handling

- If the Atlassian MCP is not configured or authentication fails — at auto-detect time, at Jira-key fetch time, at Epic creation, or anywhere else — stop and tell the user how to authenticate. Do not write a partial `.spec-to-jira.yaml` or a partial `plano-<slug>.md`.
- If a **Jira issue key** input refers to a missing or inaccessible issue, stop and tell the user the issue key was not found. Do **not** fall back to paste-or-path — the Atlassian MCP is needed later anyway, so masking its failure here only delays the same error.
- If a **Google Docs URL** input cannot be fetched (Drive MCP missing, auth failure, access denied, empty body), trigger the paste-or-path fallback described in "Input resolution". Do not abort.
- If a **local `.md` path** input does not exist or is not a `.md` file, stop and tell the user. Do not auto-fall-back — the user typed a path, not a URL, so paste-or-path would be surprising here.
- If `team.yaml` is missing, write the default file and continue. Never abort the run because of a missing `team.yaml` — the default is always usable.
- If the auto-detected project has no Sub-task issue type (or the equivalent), stop and tell the user before writing `.spec-to-jira.yaml`. This skill requires the three-level hierarchy.
- If Story creation fails partway through, do not roll back created issues (this slice never deletes). Continue to Sub-task creation for whatever Stories did succeed. Write `plano-<slug>.md` with whatever was created so the user can see the partial state, and clearly mark which user stories failed so the user can re-run manually.
- If Sub-task creation fails for a particular cell, log it inline in the matrix (write `❌ <reason>` in the cell instead of `✅ <citation>`) and move on. Do not abort the whole run for a single Sub-task failure.

## Out of scope for this slice

- Review-pause workflow (editing the `.md` and approving with `ok`).
- Multi-milestone handling — this slice always creates exactly one Epic.
- Idempotent re-runs — re-running on the same PRD + tech spec will create duplicate Jira issues. The user is responsible for not doing that yet. (The snapshot timestamp in the metadata header makes duplicate runs easier to spot manually, but the skill does not yet deduplicate.)
- Tech specs organized per user story (Type-2). Only Type-1 (per stack) is supported in this slice.
- Creating "blocks" / "is blocked by" links between issues. The link type names are persisted in `.spec-to-jira.yaml` so future slices can use them, but this slice does not create any link.
- Auto-refresh: once the PRD or tech spec content is captured in step 2, the skill operates on that snapshot. If the source document changes later, the user must re-run the skill to pick up the change.
