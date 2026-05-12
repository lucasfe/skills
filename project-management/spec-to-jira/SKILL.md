---
name: spec-to-jira
description: Read a local PRD `.md` plus a Type-1 tech spec `.md`, derive user stories, perform an inverted scan across stack sections (cloud, web, data), and create one Epic + one Story per user story + one Sub-task per covered (story × stack) cell in Jira via the Atlassian MCP. Records the Epic key, Story keys, and a "Matriz de cobertura" table in a `plano-<slug>.md` file. Use when user wants to bootstrap a Jira hierarchy with stack-split sub-tasks from a PRD and a tech spec.
---

# Spec to Jira (slice 2 — tech spec + coverage matrix + sub-tasks per stack)

Take a local PRD Markdown file and a local Type-1 tech spec Markdown file, derive a list of user stories from the PRD, perform an inverted scan of the tech spec to build a coverage matrix (user story × stack), and create the corresponding Epic + Stories + Sub-tasks in Jira via the Atlassian MCP. Record the created keys and the coverage matrix in a `plano-<slug>.md` file in the current working directory.

This slice extends slice 1 with tech spec ingestion, the coverage matrix, and one Sub-task per covered cell. Stack labels and title prefixes are **hardcoded** in this slice — configuration via `team.yaml` arrives in a later slice. Review-pause workflow, multi-milestone handling, Google Docs input, Jira-key input, and idempotent re-runs remain out of scope.

## Inputs

- **PRD path** (required): a local `.md` file passed as the **first** argument: `/spec-to-jira <path-to-prd.md> <path-to-tech-spec.md>`. If no first argument is provided, ask the user for the PRD path once and proceed.
- **Tech spec path** (required): a local `.md` file passed as the **second** argument. If no second argument is provided, ask the user for the tech spec path once and proceed. If only one path is given and it is ambiguous, ask the user to clarify which file is the PRD and which is the tech spec.
- **Jira project key** (required): always ask the user for this in chat at the start of every run. Do not read it from any config file — there is no persisted config in this slice.

## Stack vocabulary (hardcoded in this slice)

Exactly three stacks are recognized, in this order, every run:

| Stack | Label | Title prefix |
|---|---|---|
| Cloud | `stack:cloud` | `[cloud]` |
| Web | `stack:web` | `[web]` |
| Data | `stack:data` | `[data]` |

The matrix columns and Sub-task creation iterate over these three stacks in this fixed order. Do not invent additional stacks even if the tech spec mentions other concerns (e.g., mobile, infra). If the user needs a different set, they will configure it in slice 7.

## Process

### 1. Resolve the PRD and the tech spec

- Read the PRD `.md` file with the Read tool. If the path does not exist or is not a `.md` file, stop and tell the user.
- Read the tech spec `.md` file with the Read tool. If the path does not exist or is not a `.md` file, stop and tell the user.
- Pick a slug for the plan filename from the PRD's first H1 title (lowercased, kebab-case, ASCII-only). If there is no H1, derive the slug from the PRD's filename.

### 2. Ask for the Jira project key

Ask the user, in chat, for the Jira project key (e.g., `PROJ`). Do not proceed without it.

### 3. Derive user stories from the PRD

Read the PRD and derive a list of well-defined user stories. Prefer stories that are already written in `As a <actor>, I want <feature>, so that <benefit>` form; if the PRD lacks them, synthesize the list from the problem statement, solution, and acceptance criteria sections. Each story must be a single, independently-meaningful unit of user value — do not stitch multiple goals into one story, and do not split a single goal across multiple stories.

### 4. Identify the tech spec's stack sections (Type-1 organization)

The tech spec is assumed to be **Type-1 organized**: top-level sections grouped per stack, each section describing the implementation work needed for that stack across the whole product. Map headings in the tech spec to the three hardcoded stacks:

- **Cloud**: backend, services, APIs, jobs, infra, deployment, observability that lives server-side.
- **Web**: client-side application, UI, components, routing, client state, browser concerns.
- **Data**: schema, migrations, ETL/ELT, pipelines, analytics, data contracts, warehousing.

Capture, per stack, the list of headings (with their heading text and an anchor or section number) that belong to that stack. Headings that do not clearly belong to one of the three stacks are ignored in the inverted scan — they will not appear in the matrix. If a tech spec heading legitimately spans two stacks, count it under both.

If the tech spec has none of the three stacks represented at all (e.g., it is empty or unrelated), stop and tell the user — there is nothing to scan.

### 5. Inverted scan → coverage matrix

For each derived user story, scan **all three stack sections** of the tech spec and identify the specific tech spec section(s) that describe implementation work needed to deliver that story. The scan is **inverted**: the user story drives the lookup, and the tech spec is the lookup target.

The output is a coverage matrix with one row per user story and three columns (cloud, web, data). Each cell is either:

- **Covered**: a non-empty list of tech spec section citations (heading text + anchor or section number) that describe the relevant implementation work for that story in that stack.
- **Empty**: no tech spec section in that stack describes work needed for this story. Leave the cell blank in the matrix and create no Sub-task for that cell.

A story can be covered in zero, one, two, or all three stacks. Do not invent coverage to fill cells — empty cells are valid and informative.

### 6. Confirm before writing to Jira

Print:

- The proposed Epic title (use the PRD's H1 or filename slug as the source).
- The numbered list of derived user stories.
- A preview of the coverage matrix: one row per story, with a short citation list per covered cell (heading text is enough at this point).

Ask the user to confirm before any Jira write happens. Iterate until the user approves. Once approved, do not ask again in the same run.

### 7. Create the Epic

Use the Atlassian MCP to discover the cloud ID (`getAccessibleAtlassianResources`) and then create exactly one Epic in the user's specified Jira project (`createJiraIssue` with the project's Epic issue type). The Epic's summary is the PRD title; the description is a short pointer back to the source PRD path, a pointer to the tech spec path, and a one-paragraph summary derived from the PRD.

Capture the returned Jira key (e.g., `PROJ-1234`).

### 8. Create one Story per user story

For each derived user story, create a Story in the same project (`createJiraIssue`) and link it to the Epic. Use whichever mechanism the project supports for parenting a Story to an Epic — the `Epic Link` custom field on classic projects, or the `parent` field on next-gen / team-managed projects. If the first attempt fails because the field is not available, retry with the other mechanism before giving up.

The Story summary is the user story's `<feature>` clause; the description is the full `As a … I want … so that …` sentence plus any extra context from the PRD that clarifies the story.

Capture the returned Jira key for each Story.

### 9. Create one Sub-task per covered (story × stack) cell

For each non-empty cell in the coverage matrix, create one Sub-task in the same project (`createJiraIssue` with the project's Sub-task issue type), parented to the corresponding Story (`parent` field set to the Story key). For each Sub-task:

- **Summary**: `<title prefix> <Story summary>` — e.g. `[cloud] Allow user to reset password`. The prefix is the hardcoded prefix for that stack (see the table above).
- **Labels**: include the hardcoded stack label — e.g. `stack:cloud`. Do not add other labels in this slice.
- **Description**: state which Story this Sub-task implements (by Story key) and cite the relevant tech spec section(s) for this (story, stack) cell. Use the heading text plus an anchor or section number so a reviewer can navigate back to the source. Also include the source tech spec path.

Iterate stacks in the fixed order (cloud → web → data) and stories in the order they appear in the matrix, so the resulting Sub-tasks are created in a predictable sequence.

Capture the returned Jira key for each Sub-task. Do not create Sub-tasks for empty cells.

### 10. Write `plano-<slug>.md`

In the current working directory, write a file named `plano-<slug>.md` with the following structure:

```
# <PRD title>

Source PRD: <path-to-prd.md>
Source tech spec: <path-to-tech-spec.md>
Generated: <ISO 8601 timestamp>
Jira project: <PROJECT_KEY>

## Epic

- [<EPIC_KEY>] <Epic summary>

## Stories

1. [<STORY_KEY_1>] As a <actor>, I want <feature>, so that <benefit>
2. [<STORY_KEY_2>] As a <actor>, I want <feature>, so that <benefit>
...

## Matriz de cobertura

| História | cloud | web | data |
|---|---|---|---|
| [<STORY_KEY_1>] <feature> | ✅ <citation> | ✅ <citation> |  |
| [<STORY_KEY_2>] <feature> |  | ✅ <citation> | ✅ <citation> |
...

## Sub-tasks

- [<SUBTASK_KEY_1>] [cloud] <Story summary> — parent [<STORY_KEY_1>] — cites <tech spec section>
- [<SUBTASK_KEY_2>] [web] <Story summary> — parent [<STORY_KEY_1>] — cites <tech spec section>
- [<SUBTASK_KEY_3>] [web] <Story summary> — parent [<STORY_KEY_2>] — cites <tech spec section>
- [<SUBTASK_KEY_4>] [data] <Story summary> — parent [<STORY_KEY_2>] — cites <tech spec section>
...
```

Notes on the matrix:

- Cells use `✅ <citation>` when covered, with the citation being the tech spec heading text (or section number) you used. If a cell is covered by multiple tech spec sections, separate them with `; ` inside the same cell.
- Empty cells stay literally empty between the pipes — do not write `—` or `n/a`.
- The matrix lives at the Epic level (one matrix per Epic). Since this slice always produces exactly one Epic, there is exactly one matrix per `plano-<slug>.md`.

If a `plano-<slug>.md` already exists in the cwd, do not overwrite silently — tell the user and ask whether to overwrite or pick a new slug.

### 11. Summarize in chat

Print a concise summary: the Epic key, the count of created stories, the count of created sub-tasks (broken down per stack), the list of story keys, the list of sub-task keys, and the path to the written `plano-<slug>.md`. Include a direct link to the Epic in Jira if the MCP response includes one.

## Failure handling

- If the Atlassian MCP is not configured or authentication fails, stop and tell the user how to authenticate. Do not write a partial `plano-<slug>.md`.
- If Story creation fails partway through, do not roll back created issues (this slice never deletes). Continue to Sub-task creation for whatever Stories did succeed. Write `plano-<slug>.md` with whatever was created so the user can see the partial state, and clearly mark which user stories failed so the user can re-run manually.
- If Sub-task creation fails for a particular cell, log it inline in the matrix (write `❌ <reason>` in the cell instead of `✅ <citation>`) and move on. Do not abort the whole run for a single Sub-task failure.

## Out of scope for this slice

- Review-pause workflow (editing the `.md` and approving with `ok`).
- Config files (`team.yaml`, `.spec-to-jira.yaml`) — stack vocabulary stays hardcoded here.
- Multi-milestone handling — this slice always creates exactly one Epic.
- Google Docs URLs as input.
- Jira issue keys as input.
- Idempotent re-runs — re-running on the same PRD + tech spec will create duplicate Jira issues. The user is responsible for not doing that yet.
- Tech specs organized per user story (Type-2). Only Type-1 (per stack) is supported in this slice.
- Stacks beyond the three hardcoded ones (cloud, web, data).
