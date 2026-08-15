# Getting started — step by step

This is deliberately slow. Each step is small enough to verify before moving
to the next, and nothing here turns on automation yet — that's the last step,
not the first.

## Step 1 — Set up a dedicated Jira service account

Don't run this under your personal Atlassian login.

- Create (or ask your Atlassian admin to create) a service account scoped
  **only** to the TFAE project — not TA, not TFHFUNNEL.
- Generate an API token for that account.
- Confirm the account can: read issues, comment, transition status, and
  create subtasks — and nothing more (no admin rights, no delete).

**Why first:** everything downstream writes to Jira. If something misfires
later, you want the blast radius contained to one project and one
narrowly-scoped account, not your own credentials.

## Step 2 — Confirm TFAE's actual workflow and issue types

Before any automation touches real tickets:

- List every status in TFAE's workflow and confirm which ones the plan
  will use (To Do, In Progress, Code Review, Ready To Test, Testing,
  Ready for Production).
- Confirm a "Subtask" issue type exists and can be created under a Story in
  this project — this is what QA will use to file bugs later.
- Confirm task-level issues are distinguishable from stories cleanly (by
  issue type), since the conductor's queries will rely on that distinction.

**Why:** this is where a wrong assumption would otherwise surface silently —
better to check it by hand now than discover the conductor is sweeping up
the wrong issue type mid-cycle.

## Step 3 — Get a Cursor API key and confirm one real response

- Generate a Cursor API key from the Cursor Dashboard.
- Manually create a single test Cursor Cloud Agent (via curl or Cursor's own
  interface, not through this pipeline) against a real repo.
- Inspect the actual response shape when that agent finishes — specifically,
  what field says it's done, and what field holds the PR URL.

**Why:** this is the single riskiest unknown in the whole plan. Guessing
these field names would mean this step could go unnoticed later. Confirming
it by hand now removes the guesswork before it's load-bearing.

## Step 4 — Add a kill switch, before anything else runs on a schedule

- Decide the mechanism: a Jira label (e.g. `automation-paused`) is simplest —
  something that can be checked by a single lookup and toggled by anyone
  with Jira access, no code change required.
- Confirm it can be set and unset by hand and that checking for it is cheap
  (one query, not a search across the whole project).

**Why here, not last:** every step after this one involves something running
automatically. You want the off-switch to exist and be tested *before* the
first automated action, not added after something's already gone sideways.

## Step 5 — Dispatch one task, by hand, and watch it end-to-end

- Pick one real, low-stakes TFAE task sitting in "To Do."
- Manually trigger a single dispatch (not on a schedule) — create the scoped
  Cursor agent, comment the agent reference on the Jira issue, move it to
  "In Progress."
- Watch it through to a finished PR. Manually check the PR was linked back
  correctly and the issue moved to "Code Review."

**Why:** this is the smallest possible end-to-end proof — one task, fully
observed, with no scheduling or concurrency involved yet. If anything about
steps 1–3 was wrong, this is where it surfaces cheaply.

## Step 6 — Manually verify one story, using your own judgment

- Once a story's tasks are sitting in "Code Review" or "Ready To Test," walk
  through it yourself: check out the PR(s), confirm the acceptance criteria
  are actually met.
- Either move the story to "Ready for Production" yourself, or file a bug
  subtask yourself, exactly as the automated QA step would eventually do.

**Why:** doing this by hand first means you'll notice if the acceptance
criteria are too vague to verify automatically, or if a task's scope was
unclear — problems worth catching before any automation inherits them.

## Step 7 — Turn on the conductor, capped and slow

Only after steps 1–6 have each been confirmed once by hand:

- Set the concurrency cap (2–3 agents in flight, system-wide).
- Set the schedule (every 20–30 minutes is a reasonable start — slower is
  fine, there's no rush).
- Confirm the kill switch from Step 4 is checked first, every cycle, before
  anything else runs.
- Let it run against a small number of real tasks and watch the first few
  cycles closely rather than walking away immediately.

**QA stays manual for now** — the conductor dispatches and syncs, but you
still decide when a story is ready and run QA yourself, per Step 6. That's a
deliberate choice, not a placeholder to remove quickly — see the writeup for
why.

## Step 8 — Only after the above is boring

Once the capped, manual-QA version has run long enough that it feels
unremarkable rather than nerve-wracking, that's the signal to consider (not
necessarily do) any of the following, one at a time:

- Automating the QA step itself, once you trust the judgment involved.
- Real file-overlap detection, to safely raise the concurrency cap.
- Webhook-based Cursor status updates instead of polling.

None of these are urgent. The slow version is a complete, working system on
its own — these are optimizations for later, not missing pieces now.
