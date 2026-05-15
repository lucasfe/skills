---
name: sec-consult
description: Generate a security consult ticket from a tech spec, PRD, or project reference. Accepts local files, Jira ticket IDs, or Confluence URLs as input. Use when the user needs to create a security consult for a project.
---

# Security Consult Generator

Generate a security consult ticket text from a project reference document (tech spec, PRD, HLD, etc.).

## Arguments (at least one required)

Arguments are space-separated. Each argument is one of:
- A **Jira ticket ID** matching `[A-Z]+-\d+` (e.g., `PROJ-123`) — fetched via the `/jira` skill
- A **Confluence URL** containing `confluence` or `wiki` in the hostname — fetched via the `/confluence` skill
- A **local file path** (e.g., `docs/spec.md`) — read directly

Multiple arguments can be mixed to combine context from different sources.

## Process

### Step 1 — Gather context

For each argument:
- **Jira ticket ID**: Invoke the `/jira` skill to fetch the ticket summary, description, and comments.
- **Confluence URL**: Invoke the `/confluence` skill to fetch the page content.
- **File path**: Read the file directly.

If any fetch fails, warn the user but continue with whatever context was successfully gathered.

### Step 2 — Analyze the project

From the gathered documents, extract:
- **What is being built**: New services, pipelines, infrastructure components
- **Why**: Business motivation, customer commitments, deadlines
- **Architecture**: Technologies involved (databases, caches, APIs, third-party integrations, message queues)
- **Data involved**: What data is processed, stored, or exposed. What data is new vs already existing.
- **External integrations**: Third-party APIs, webhook consumers, partner-facing surfaces
- **Access control changes**: New roles, permissions, endpoints, manual overrides
- **User input surfaces**: Any new paths where user-controlled data enters the system

### Step 3 — Generate the security consult text

Generate the consult using the exact template below. Fill every section based on the analysis from Step 2. Be specific and technical — avoid vague statements. If information is not available for a section, note what's missing rather than guessing.

<sec-consult-template>

**Q: Does this project involve creating a new service (either in eero Tools or Amazon Builder Tools)? If so, an ASR will be required unconditionally. If yes, what type of service? Provide details on the new service.**

(Describe new services, pipelines, or infrastructure. Include the technology stack, what the service does, and how it interacts with other systems. If no new service, state that clearly.)

**Q: What is the summary of this project? Explain "why we are doing this project" and a high-level explanation of what eero will be changing about its systems/features/processes/data stores.**

(First paragraph: motivation and business drivers. Second section: bulleted list of key technical changes, max 5 lines. Each bullet should be self-contained and specific.)

**Q: What are the security implications of this feature/project?**

(Bulleted list of security-relevant areas. For each, describe what is changing and why it matters from a security perspective. Cover: new data exposure, new endpoints, third-party integrations, access control changes, user input surfaces. Explicitly state if there are NO new user-facing input surfaces.)

**Q: Link to your completed classification request**

(Reference existing classification if known, or note that one needs to be created. Describe what data is involved to help scope the classification.)

Please make sure to add release date. The status should remain on Triage. Once Triaged, the security team will move it to the appropriate status.

**Project Leads:**

Tech Lead: *(fill in)*

SDM: *(fill in)*

PM: *(fill in)*

**Release Date:** *(fill in based on documents, or mark as TBD)*

**Documents & Links:**

*(list all source documents with links)*

**Priority:** *(P0/P1/P2 with justification — mention ASR requirement, deadlines, or scope that drives the priority)*

</sec-consult-template>

### Step 4 — Generate the Jira issue summary

After the consult text, generate a Jira-ready summary:

**Title format:** `Security Consult: <Project Name> — <key capabilities in ~10 words>`

**Description:** A concise paragraph (3-5 sentences) summarizing the project scope and listing the key areas requiring security review as bullets. Include release timeline and priority at the end.

### Step 5 — Present to the user

Display both outputs clearly separated:

1. The full security consult text (under a `## Security Consult` header)
2. The Jira issue summary (under a `## Jira Issue` header)

Tell the user to review, fill in the placeholder fields (Project Leads, classification link, etc.), and copy when ready.
