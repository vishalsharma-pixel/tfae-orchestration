# Jira → Cursor → Claude QA: revised plan (slow, guarded, manual QA)

## What changed from the original plan

The first version of this plan optimized for a fully autonomous loop: dispatch
everything undispatched, sync continuously, auto-trigger QA the moment a
story looked ready, and let bugs recycle through the pipeline with no human
in the loop until a story reached "Ready for Production."

That version had real failure modes — an unbounded bug loop, no cap on how
many Cursor agents could run at once, a QA step that could file bugs
autonomously with no review, and no way to pause it short of editing code.

This version trades speed for visibility. The goal isn't to remove humans
from the loop — it's to let Cursor do the mechanical work (writing code,
opening PRs) while every decision that matters (what to build, whether it's
actually done) still goes through you.

## Core philosophy

**Small, visible, reversible.** At any moment, you should be able to look at
the system and know exactly what it's doing — not infer it from logs. That
means:

- A hard cap on how many things can be happening at once (2–3 Cursor agents,
  system-wide, not per story).
- The conductor's job each cycle is narrow: sync what's running, and if
  there's a free slot, dispatch **one thing**, then stop. It does not try to
  drain the whole queue in one pass.
- QA is **not automated yet**. You trigger it manually against a specific
  story when you've decided it's ready — the system doesn't get to decide
  that on its own.
- Every loop in the system has a hard ceiling. The bug-loop is capped at 3 QA
  cycles per story before it stops recycling and asks for a human instead.
- There's a single kill switch that stops everything, checked first thing
  every cycle.

## What the system actually does

1. **You pick tasks and leave them in Jira's "To Do" column.** Nothing about
   what work exists is automated — that's still your call.

2. **The conductor wakes up on a schedule** (e.g. every 20–30 min) and does
   exactly three things, in order:
   - Checks the kill switch. If it's set, stop — do nothing else this cycle.
   - Syncs any tasks currently "In Progress": checks if their Cursor agent
     finished, and if so, links the PR in a Jira comment and moves the task
     to "Code Review."
   - If there's a free slot (fewer than 2–3 agents currently running), picks
     the next task off "To Do" and dispatches it to a new Cursor agent.

3. **Cursor does the actual coding work** — checks out the task's scope,
   writes the change, opens a PR. It does not merge anything itself.

4. **PRs sit in "Code Review" until you're ready.** Nothing auto-advances
   them. You can review a PR whenever suits you — this is a natural place to
   look things over before anything is called "done."

5. **When you decide a story's ready, you manually trigger QA** against that
   one story. QA's job is narrow: check the linked PR(s) against the story's
   acceptance criteria, and either:
   - Mark the story verified (moves to "Ready for Production"), or
   - File a bug as a new subtask under the story, with a clear description
     of what's wrong — which lands back in "To Do" for the next conductor
     cycle to pick up.

6. **The bug loop is bounded.** If a story hits 3 failed QA cycles without
   passing, the system stops recycling it automatically and flags it for you
   to look at directly — rather than quietly grinding the same story forever.

## Why manual QA, for now

QA is the highest-judgment, highest-blast-radius step in the whole pipeline —
it's the one place where the system would be creating new work
(bug subtasks) based on its own read of whether something is actually
correct. Automating that from day one means trusting an unproven judgment
call with no review. Keeping it manual for now means you get everything
valuable about the pipeline (Cursor doing the grunt work, Jira staying in
sync) without handing that specific decision to an unsupervised loop before
you've seen it work reliably by hand.

## Guardrails, and why each one exists

| Guardrail | What it prevents |
|---|---|
| Hard cap: 2–3 Cursor agents in flight, system-wide | Runaway spend from a bad query or a missed dedupe check fanning out dozens of agents at once |
| Conductor dispatches one task per free slot, then stops | The conductor trying to drain an entire backlog in one cycle, making the system's behavior hard to predict or watch |
| Manual QA trigger (not automatic) | The system autonomously deciding "this is done" or filing bugs with no review |
| Cap: 3 QA cycles per story before manual escalation | An infinite bug loop that never terminates and keeps spending on the same story forever |
| Kill switch checked first, every cycle | No way to pause the system short of touching code — always want one lever that stops everything immediately |
| Staleness check on in-flight agents | A stuck or errored agent silently polling forever, invisibly consuming a slot that never frees up |
| Dedupe check before dispatch | The same task getting dispatched twice (e.g. a manual test run overlapping the scheduled conductor), creating duplicate agents and conflicting PRs |

## What's deliberately *not* built yet

- **No automated QA Routine.** QA is you, manually, using the same Jira
  read/write access already set up — not a separate autonomous agent.
- **No webhook-based Cursor status updates.** The conductor polls on its own
  schedule rather than reacting to push notifications — slower, but the
  failure mode (a missed webhook) is different and this keeps the system's
  behavior easier to reason about while it's new.
- **No parallel-agent conflict detection.** The 2–3 agent cap combined with
  one-task-per-cycle dispatch is a blunt but effective substitute for real
  file-overlap detection, which isn't built yet.

These aren't omissions to fix immediately — they're the next things to
consider *once* the slower version has run long enough to trust.
