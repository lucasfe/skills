---
name: spec-to-jira
description: Walking skeleton — read a local PRD `.md`, derive user stories, and create one Epic plus one Story per user story in a Jira project via the Atlassian MCP, then write a `plano-<slug>.md` recording the keys. Use when user wants to bootstrap a Jira hierarchy from a PRD file.
---

# Spec to Jira (slice 1 — walking skeleton)

Take a local PRD Markdown file, derive a list of user stories from it, and create one Epic plus one Story per user story in Jira via the Atlassian MCP. Record the created keys in a `plano-<slug>.md` file in the current working directory.

This is the minimum end-to-end path of the skill. Tech spec ingestion, sub-tasks, coverage matrices, review pauses, config files, multi-milestone handling, Google Docs input, and Jira-key input are out of scope for this slice — keep the flow lean.

## Inputs

- **PRD path** (required): a local `.md` file passed as the first argument: `/spec-to-jira <path-to-prd.md>`. If no argument is provided, ask the user for the path once and proceed.
- **Jira project key** (required): always ask the user for this in chat at the start of every run. Do not read it from any config file — there is no persisted config in this slice.

## Process

### 1. Resolve the PRD

- Read the `.md` file at the provided path with the Read tool. If the path does not exist or is not a `.md` file, stop and tell the user.
- Pick a slug for the plan filename from the PRD's first H1 title (lowercased, kebab-case, ASCII-only). If there is no H1, derive the slug from the PRD's filename.

### 2. Ask for the Jira project key

Ask the user, in chat, for the Jira project key (e.g., `PROJ`). Do not proceed without it.

### 3. Derive user stories

Read the PRD and derive a list of well-defined user stories. Prefer stories that are already written in `As a <actor>, I want <feature>, so that <benefit>` form; if the PRD lacks them, synthesize the list from the problem statement, solution, and acceptance criteria sections. Each story must be a single, independently-meaningful unit of user value — do not stitch multiple goals into one story, and do not split a single goal across multiple stories.

### 4. Confirm before writing to Jira

Print the proposed Epic title (use the PRD's H1 or filename slug as the source) and the numbered list of derived user stories. Ask the user to confirm before any Jira write happens. Iterate until the user approves. Once approved, do not ask again in the same run.

### 5. Create the Epic

Use the Atlassian MCP to discover the cloud ID (`getAccessibleAtlassianResources`) and then create exactly one Epic in the user's specified Jira project (`createJiraIssue` with the project's Epic issue type). The Epic's summary is the PRD title; the description is a short pointer back to the source PRD path and a one-paragraph summary derived from the PRD.

Capture the returned Jira key (e.g., `PROJ-1234`).

### 6. Create one Story per user story

For each derived user story, create a Story in the same project (`createJiraIssue`) and link it to the Epic. Use whichever mechanism the project supports for parenting a Story to an Epic — the `Epic Link` custom field on classic projects, or the `parent` field on next-gen / team-managed projects. If the first attempt fails because the field is not available, retry with the other mechanism before giving up.

The Story summary is the user story's `<feature>` clause; the description is the full `As a … I want … so that …` sentence plus any extra context from the PRD that clarifies the story.

Capture the returned Jira key for each Story.

### 7. Write `plano-<slug>.md`

In the current working directory, write a file named `plano-<slug>.md` with the following structure:

```
# <PRD title>

Source PRD: <path-to-prd.md>
Generated: <ISO 8601 timestamp>
Jira project: <PROJECT_KEY>

## Epic

- [<EPIC_KEY>] <Epic summary>

## Stories

1. [<STORY_KEY_1>] As a <actor>, I want <feature>, so that <benefit>
2. [<STORY_KEY_2>] As a <actor>, I want <feature>, so that <benefit>
...
```

If a `plano-<slug>.md` already exists in the cwd, do not overwrite silently — tell the user and ask whether to overwrite or pick a new slug.

### 8. Summarize in chat

Print a concise summary: the Epic key, the count of created stories, the list of story keys, and the path to the written `plano-<slug>.md`. Include a direct link to the Epic in Jira if the MCP response includes one.

## Failure handling

- If the Atlassian MCP is not configured or authentication fails, stop and tell the user how to authenticate. Do not write a partial `plano-<slug>.md`.
- If Story creation fails partway through, do not roll back created issues (this slice never deletes). Write `plano-<slug>.md` with whatever was created so the user can see the partial state, and clearly mark which user stories failed so the user can re-run manually.

## Out of scope for this slice

- Tech spec ingestion and coverage matrix.
- Sub-tasks of any kind (no stack split, no PM virtual stack).
- Review-pause workflow (editing the `.md` and approving with `ok`).
- Config files (`team.yaml`, `.spec-to-jira.yaml`).
- Multi-milestone handling — this slice always creates exactly one Epic.
- Google Docs URLs as input.
- Jira issue keys as input.
- Idempotent re-runs — re-running on the same PRD will create duplicate Jira issues. The user is responsible for not doing that yet.
