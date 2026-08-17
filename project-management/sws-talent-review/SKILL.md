---
name: sws-talent-review
description: Turn a person's Work Summary doc plus your feedback into a balanced, LP-grounded performance summary; one person per round, appended to a combined markdown file. Use when writing team performance/talent-review summaries from Work Summary docs (Google Docs or pasted text).
---

# SWS Talent Review

Turn one person's **Work Summary** (a Google Doc or pasted text) plus the
manager's feedback into a balanced, evidence-backed performance summary, then
append it to a single combined markdown file for the review cycle.

Work **one person per round**. Never batch. Never write to disk until the
manager approves the summary in chat.

## Flow

Run these steps in order for each person. Ask the questions **one at a time**
and wait for the answer before moving on.

### Step 1 — Get the Work Summary

Ask: **"Paste the Work Summary — raw text, a Google Docs link/ID, or any
textual format is fine."**

- **If given a Google Docs link or ID**, extract the document ID and fetch it
  with `gws`, then flatten it to plain text:
  ```bash
  gws docs documents get --params '{"documentId": "<DOC_ID>"}' --format json 2>/dev/null | python3 -c '
  import json,sys
  d=json.load(sys.stdin)
  def walk(content):
      out=[]
      for el in content:
          p=el.get("paragraph")
          if p:
              line="".join(e.get("textRun",{}).get("content","") for e in p.get("elements",[]))
              out.append(line.rstrip("\n"))
          t=el.get("table")
          if t:
              for row in t.get("tableRows",[]):
                  for cell in row.get("tableCells",[]):
                      out.extend(walk(cell.get("content",[])))
      return out
  print("\n".join(walk(d["body"]["content"])))
  '
  ```
  The doc ID is the long token in `/document/d/<DOC_ID>/edit`.
- **If `gws` fails or is not authenticated**, fall back to asking the manager to
  paste the doc text directly.
- **If given raw text**, use it as-is.

The doc is typically a "`<YEAR> Work Summary`" template with four sections:
Q1 customer impact, Q2 AI usage, Q3 learnings, Q4 growing others — with Amazon
Leadership Principle tags. **Infer the period and role from the doc title**
(e.g. "2026 H1", "SDE III", "FEE III", "QAE"). Confirm only if ambiguous.

### Step 2 — Get the feedback

Ask: **"What feedback do you want to fold in? Strengths, growth areas, specific
incidents, peer comments — anything not already in the doc."**

Accept free-form text. Do not auto-search Slack or other systems; this feedback
is judgment-heavy and manager-provided.

### Step 3 — Get the trending direction (sets tone)

Ask: **"How is this person trending — improving, or needing to improve?"**

This calibrates tone and the weight given to growth areas. **Never state the
rating or trend explicitly in the output** — it only shapes the writing.

- **Improving / strong** → celebratory-but-balanced; strengths lead, growth
  areas are genuine but lighter.
- **Needing to improve** → honest and clear about gaps, still respectful and
  constructive; growth areas are more prominent. Do not soften to the point of
  dishonesty and do not name the rating.

### Step 4 — Generate, iterate, then persist

1. Write the summary using the **Output template** and **Style rules** below.
2. Present it in chat. Iterate on the manager's edits — expect several passes.
3. **Only after the manager approves**, append it to the combined file (see
   **Persistence**).
4. Offer to start another person, reusing the same combined file and period.

## Output template

```
**<Name> — Performance Summary (<period>)**

<Opening paragraph>: one or two sentences, "Over the past six months, <Name>...",
naming the 4–6 headline projects/deliverables.

<Narrative paragraph>: the highest-impact project in depth — concrete evidence:
ticket IDs, metrics, architecture decisions, dates.

**<Theme / LP>.** A strength section with evidence. Section headers are bold and
tied to an Amazon Leadership Principle or a role theme (e.g. "Dive Deep and
root-causing.", "Invent and Simplify.", "Elevating the team.", "Ownership and
Earn Trust."). Add as many strength sections as the material supports.

**Growth areas.** (or a specific dev-area header like "Communication and
presence (an area of development).") Constructive and honest. Reinforce the
person's own stated learnings from the doc where they exist; ground critiques in
the role guidelines / Leadership Principles where relevant.

<Closing paragraph>: "Looking ahead, <Name>'s biggest opportunities are...",
then a one-line characterization of the person.
```

The number of thematic strength sections flexes with the available material.
The prominence of growth areas flexes with the trending direction from Step 3.

## Style rules

- **No em dashes.** Use commas, parentheses, or periods instead. This is the
  single most common backslide — do not use `—` anywhere in the summary body.
- **Never state the rating or trending direction.** It only shapes tone.
- **Balanced.** Real positives plus genuine growth areas (the doc template
  itself asks for a balanced record).
- **Concrete evidence** pulled from the doc: ticket numbers, metrics, project
  names, dates. Do not invent figures.
- **Ground critiques** in the role guidelines or Amazon Leadership Principles
  where relevant (e.g. an L6 is expected to resolve ambiguity).
- Section headers are **bold, tied to an LP or theme**.
- **Pronouns:** use they/them or the person's stated pronouns; never guess
  gender from a name.

## Persistence

The combined file holds every person's summary for one review cycle.

1. **Discover the folder.** Scan the working directory (and one level down) for
   an existing folder that already contains performance-summary `.md` files or
   is named like `summaries/`, `performance/`, `reviews/`, or `work-summaries/`.
   Suggest the best candidate and **ask for confirmation**. If none exists,
   propose creating `./performance-summaries/` and confirm before creating it.
   ("Repo" here just means the current working directory.)
2. **File name:** `<year>-<period>-performance-summaries.md`
   (e.g. `2026-H1-performance-summaries.md`), derived from the inferred period.
   Reuse the same file for every person in the session.
3. **Write only after approval** (Step 4). Never auto-write on first generation.
4. **Idempotent append:** if a section for this person already exists in the
   file (match on their `<Name> — Performance Summary` header), **confirm with
   the manager before replacing it in place**. Otherwise append a new section
   separated by `---`.

## Notes

- One person per round; never batch multiple people into one generation.
- Steps 1–3 are asked one at a time. The heavy iteration happens in Step 4,
  in chat, before anything is written to disk.
