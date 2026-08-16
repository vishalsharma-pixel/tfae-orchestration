# Story & task template — for ideation → Jira creation

Use this template whenever turning an idea into a real TFAE Story + Tasks.
The fields here aren't arbitrary — each one exists because the conductor
Routine's Step 3 ("judge whether the task is clear enough to dispatch
safely") checks for exactly this information before deciding whether to
send a task to a Cursor agent or skip it and flag it for you. A
well-filled template here means fewer tasks get stuck in "To Do" with a
"this is unclear" comment later.

## How to use this

Write your rough idea in plain language first — don't try to fill the
template in your head. Then hand the rough idea to a chat session that has
the Atlassian (Rovo) connector available, along with this template, and
ask it to:
1. Draft the Story and Task breakdown using the template below
2. Show you the draft before creating anything
3. Once approved, create the Story in TFAE, then create each Task as a
   child linked to that Story

---

## Story template

```
Title: [Short, action-oriented — what this delivers, not how]

Summary:
[2-4 sentences: what is this, and why does it matter right now?]

Acceptance criteria (story-level):
- [ ] Testable, pass/fail statement — not a vague goal
- [ ] Another one, if there's more than one condition for "done"

Out of scope:
- [Explicitly state what this story does NOT cover — prevents scope
  creep and prevents QA from failing the story for something it was
  never meant to do]

Related context:
- [Links to designs, other tickets, docs — anything a task under this
  story might need to reference]
```

**Why each field matters:**
- **Acceptance criteria** is the single most important field. The Routine's
  QA step (manual, per `docs/getting-started.md` Step 6) checks work against
  this — if it's vague, QA can't verify pass/fail cleanly, which is exactly
  the "too unclear to verify" trap `max_bugs_filed_per_qa_pass` was built to
  catch.
- **Out of scope** prevents a task later referencing "context that doesn't
  exist in the ticket" — one of the Routine's explicit reasons to refuse
  dispatch.

---

## Task template (one per child Task under the Story)

```
Title: [Specific, single concern — not "and" in the title]

Description:
[What needs to change, specifically enough that someone unfamiliar with
the story could start without asking a clarifying question first.]

Acceptance criteria (task-level):
- [ ] Specific, checkable — this is what the Cursor agent's prompt gets
      built from directly (see routines/conductor-routine-prompt.md Step 4)

Files/areas likely touched (if known):
- [e.g. "frontend/checkout", "api/orders" — doesn't need to be exact,
  but helps the one-task-per-story-per-cycle guardrail reason about
  whether two tasks under the same story might conflict]

Depends on:
- [Another task in this story it must follow, or "none" if independent]
```

**Why each field matters:**
- **Single concern per task** is what keeps tasks small enough to dispatch
  as one Cursor agent safely — a task with "and" in the title is often two
  tasks that should be split, since the Routine dispatches one task at a
  time per story per cycle anyway.
- **Files/areas touched** feeds directly into the parallel-dispatch safety
  question flagged throughout this build (`max_dispatch_per_story_per_cycle`
  in guardrails.yaml is a blunt substitute for real file-overlap detection
  — this field is the one piece of manual input that makes that substitute
  more informed).
- **Depends on** matters because the Routine currently has no automatic
  sequencing logic — if Task B genuinely needs Task A's PR merged first,
  that has to be handled by you (leaving Task B out of "To Do" until ready),
  not assumed by the pipeline.

---

## A filled example

```
STORY
Title: Add CSV export to order history page

Summary: Customers have asked for a way to export their order history for
expense reports. Add a button on the order history page that downloads
the visible orders as a CSV.

Acceptance criteria:
- [ ] "Export CSV" button visible on the order history page
- [ ] Clicking it downloads a .csv file with columns: date, order ID,
      total, status
- [ ] Only exports orders currently visible (respects existing filters)

Out of scope:
- Does not need a scheduled/automatic export
- Does not need to support any format other than CSV

Related context:
- Order history page: frontend/pages/orders
```

```
TASK 1
Title: Add CSV export button and download handler to order history page

Description: Add a button labeled "Export CSV" to the order history page.
On click, generate a CSV client-side from the currently displayed/filtered
order data and trigger a browser download.

Acceptance criteria:
- [ ] Button appears in the existing page header area, matching current
      button styling
- [ ] Downloaded file is named orders-export-YYYY-MM-DD.csv
- [ ] Columns match: date, order ID, total, status
- [ ] Exported rows match whatever filter is currently applied on screen

Files/areas likely touched:
- frontend/pages/orders

Depends on: none
```

Notice the task's acceptance criteria are strictly more specific than the
story's — that specificity is exactly what Step 4 of the conductor prompt
needs to compose a good Cursor prompt without guessing.
