# Changelog

All notable changes to the orchestration config are logged here, even solo
changes — this is the audit trail for a system running on full personal
Jira permissions rather than a scoped service account.

## [Unreleased]

### Changed
- **Reverted to native GitHub repo cloning, dropping Glean as a
  dependency.** Confirmed the Claude Code Routine can access
  `github.com/vishalsharma-pixel/tfae-orchestration` directly — Step 1 of
  `routines/conductor-routine-prompt.md` now reads config from this
  Routine's natively-cloned repository instead of fetching via Glean from
  Bitbucket. Required connectors simplified back down to just Atlassian
  (Rovo). This repo needs to actually be pushed to that GitHub URL — still
  outstanding as of this change (see Known open items).
- Rewrote `routines/conductor-routine-prompt.md` around four explicit
  steps: (1) load config repo (now: native clone) and check the kill
  story/task details, folding in the concurrency cap check; (3) confirms
  the cloud agent setup is actually ready (target repo, dedup, task
  clarity) before generating anything; (4) composes the Cursor prompt and
  dispatches — marked as a draft to refine after the first few real runs.

### Fixed
- `conductor_interval_minutes` changed from 20 to 60 — confirmed Claude Code
  Routines' minimum schedule interval is 1 hour, so the original value was
  never achievable. Config now matches platform reality.

### Added
- **Pre-flight GET checks added around every Cursor API interaction.**
  Step 4 (dispatch) now does `GET /v1/me` to confirm the API key and
  connectivity work *before* the `POST /v1/agents` create call — stops
  and flags rather than firing a create blind. Step 2 (sync) now treats
  a failed `GET /v1/agents/<id>` as its own explicit failure mode (leave
  the task as-is, don't guess a status) rather than assuming success.
  This is a second independent safety layer alongside Step 3's
  Jira-comment dedup check — consistent with the "never rely on one
  check for something this expensive" principle from `docs/writeup.md`'s
  guardrails table.
- `routines/conductor-routine-prompt.md` — the conductor is now a Claude
  Code Cloud Routine's system prompt, not a Node script. Reuses your
  existing Rovo connection (same permissions/scope tradeoffs as documented
  throughout this repo). Fetches `config/*.yaml` fresh from this repo every
  cycle, so a config change takes effect on the next scheduled run with no
  redeploy. Numeric guardrails (concurrency cap, dispatch batch size, kill
  switch) remain hard mechanical gates in the prompt — judgment is scoped
  specifically to reading task content and composing the Cursor prompt,
  never to whether a guardrail applies.

### Added
- **`config/repo-map.yaml`** — resolves per-task Cursor targets instead of
  one hardcoded repo. Added after the conductor's first real multi-repo
  cycle (TFAE-5 through TFAE-8) blocked correctly rather than guessing,
  exposing that a single hardcoded target doesn't scale. Keyed by Jira's
  structured **Component** field (deliberately not a label — Component is
  a visible, structured choice at ticket-creation time). Supports two
  dispatch modes: `new_agent` (the only mode that existed before) and
  `follow_up_existing_agent` (continues an already-existing agent/branch
  instead of creating a new one — needed for TFAE-8's case). A task with
  no Component, or an unmapped one, stops and flags rather than guessing
  a target — same hard-gate philosophy as everything else in this build.
  `docs/story-task-template.md` updated to require Component on every task.

### Known open item (surfaced by this fix, not yet resolved)
- **Story/Task hierarchy mismatch**: TFAE-6/7/8 are linked to TFAE-5 via
  "Relates" issue links, not the native `parent` field — because `Story`
  and `Task` are both `hierarchyLevel: 0` in this project (siblings, not
  parent/child; only `Subtask` can natively hold `parent` to a Story).
  This means `find_child_tasks_of_story`'s `parent = {story_key}` query
  currently finds zero children for TFAE-5, even though TFAE-7/TFAE-8 are
  genuinely ready to dispatch. Two fix paths identified, neither applied
  yet pending a decision: (1) convert future child work items to issue
  type `Subtask` with a real `parent` field (aligns with how
  `workflow-map.yaml` already treats QA bug tickets — the cleaner, more
  native fix, but requires converting existing live tickets via Jira's
  "Move" wizard, not a simple API field edit), or (2) redesign the
  lookup around "Relates" links instead of `parent` (avoids touching
  existing tickets, but issue links aren't cleanly filterable in plain
  JQL, requiring per-story `issuelinks` fetches instead of a clean query
  — harder to keep bounded at scale).
- **`node-monolith-api`'s Cursor environment name** in `repo-map.yaml`
  ("Promotion Node and Python Partners") is taken from the blocked
  cycle's report, not independently confirmed against Cursor's dashboard
  — same caution as the earlier "TF SuperAdmin Full Stack" confirmation.
- **`seller-v2-new-campaigns`'s existing agent reference** is still just a
  human-readable name ("Seller V2 - New Campaigns"), not a confirmed
  `bc-<uuid>`. Step 3 is written to stop and flag rather than guess an ID
  from this name — needs the real ID before TFAE-8 can dispatch.

### Resolved
- **`seller-v2-new-campaigns` existing agent confirmed**:
  `bc-02735464-f4e9-419f-88eb-0af659371ebc`, branch
  `vishal/creator-campaign-listing-detail-flow-1ebc` — verified against the
  real branch name (not guessed from the human-readable "Seller V2 - New
  Campaigns" label) before being recorded in `repo-map.yaml`. This was the
  last blocker on TFAE-11's dispatch path; `dispatch_mode:
  follow_up_existing_agent` can now proceed for real.
- **TFAE-9/10/11 confirmed working correctly**: real native `Subtask` type
  under TFAE-5's `parent` field (not "Relates" links) — `find_child_tasks_of_story`'s
  `parent =` query finds these. TFAE-9 correctly carried over its real
  progress (Code Review) rather than resetting — avoided a duplicate
  dispatch on already-completed work.
- **Corrected `repo-map.yaml`'s field assumption**: the "Component" field
  is actually `customfield_10321`, a FREE-TEXT custom field, not Jira's
  native Components dropdown as originally assumed. Real reliability
  difference — free text has no typo protection. Confirmed working on
  TFAE-10 (`node-monolith-api`) and TFAE-11 (`seller-v2-new-campaigns`),
  both exact matches. All references updated to cite `customfield_10321`
  explicitly.

### Open question raised (not yet answered)
- **Unplanned field discovered**: `customfield_10287`, labeled "Agent
  Sessions," exists on these issues (currently empty on TFAE-10/11) but
  was never part of this build's design. Could potentially replace the
  comment-regex-based `agentId=`/`runId=` tracking in Step 2 with a real
  structured field — worth deciding whether to adopt it, once its
  origin/intended use is confirmed.

### Resolved
- **Real dispatch-queue scale fix.** The conductor previously scanned
  "To Do" project-wide (`find_undispatched_tasks`), which would have been
  genuinely dangerous once TFAE holds hundreds/thousands of cards — no
  boundary existed on what automation could touch. Added an explicit
  opt-in gate: a new `find_agent_eligible_stories` query returns only
  stories a human has moved to "Ready for Agent," and only tasks under
  those stories (via `find_child_tasks_of_story`) are ever dispatch
  candidates. An empty eligible-story list now means "genuinely nothing
  opted in" — the conductor stops rather than falling back to a broader
  scan.
- **Four new Jira statuses added to the workflow design**: `Ready for
  Agent` (manual entry gate, story-only), `Agent In Progress` (system,
  story-only, fires on first child dispatch), `Manual Intervention Needed`
  (system, BOTH story and task level — task-level for dispatch/execution
  failures, story-level whenever any child needs it), `PR By Agent`
  (system, story-only, fires once all children reach Code Review or
  later). Full state machine and transition rules in
  `config/workflow-map.yaml`; Jira admin setup steps added to
  `docs/getting-started.md` under "Scaling setup."
- **Story status rollup added** — computed fresh every cycle from actual
  child-task state, never remembered from a prior decision. Priority
  order: Manual Intervention Needed > PR By Agent > Agent In Progress >
  Ready for Agent. A story recovers from Manual Intervention Needed
  automatically once every flagged child task is cleared by a human — no
  separate manual action needed on the story.
- Task-level "too unclear to dispatch" (Step 3) and Cursor run
  ERROR/CANCELLED/EXPIRED (Step 2) now transition the task to "Manual
  Intervention Needed" as a real status, not just a Jira comment while
  silently sitting in "To Do."
- Notification policy updated to include story rollup changes to Manual
  Intervention Needed or PR By Agent as notification-worthy events.
- **Explicit notification policy added — closes a real gap.** The Routine
  previously had a generic "log a summary" instruction with no explicit
  rule for when to actively notify versus just log silently. In practice
  it notified on some blocking-gate cycles by its own judgment, but nothing
  said a finished PR (which needs a human's review) was equally
  notification-worthy — an inconsistency, since a completed task needing
  review is at least as actionable as a stuck cycle. Added an explicit list
  of notification-worthy events (PR ready for review, run
  finished-without-PR, run errored/cancelled/expired, duplicate-agent
  holds, genuine gate failures, unclear tasks skipped) versus routine
  non-events that should just log quietly (capacity full, empty queue).
- **Idempotent dispatch added, closing a real gap surfaced by the first
  live dispatch.** During the Routine's first actual dispatch, a
  connection drop occurred between sending the create-agent call and
  receiving its response — it happened to recover safely by checking
  rather than assuming, but nothing previously guaranteed that. Step 4 now
  generates a client-supplied `agentId` (`bc-<uuid>`) before every create
  call; if a retry is needed after an uncertain response, it reuses the
  same ID rather than generating a new one, so Cursor's API returns
  `409 agent_id_conflict` on the retry instead of silently creating a
  duplicate agent for the same task.
- **Confirmed real Cursor API field names, replacing all guessed
  placeholders** (`agent.status`/`prUrl` were never real fields — the
  actual shape is `run.status` on `GET /v1/agents/{id}/runs/{runId}`, with
  `FINISHED`/`ERROR`/`CANCELLED`/`EXPIRED`/`RUNNING`/`CREATING` values, and
  PR links live nested at `run.git.branches[].prUrl`, not a flat field).
  `routines/conductor-routine-prompt.md` Steps 2 and 4 updated to match
  the confirmed OpenAPI schema exactly.
- **Cursor environment confirmed**: `TF SuperAdmin Full Stack` — the named
  Cursor Cloud Agent environment tied to `superadmin-react-portal`'s repo,
  install script, secrets, and Build snapshots. Step 4's dispatch call now
  uses `env: {type: "cloud", name: "TF SuperAdmin Full Stack"}` instead of
  a raw `repos` array (the two are mutually exclusive on Cursor's API) —
  this ties every dispatch to the pre-built Build rather than booting a
  bare environment from scratch each time.
- **Target Bitbucket repo confirmed**:
  `trulyfree-marketplace/superadmin-react-portal` (the frontend Admin
  portal). `routines/conductor-routine-prompt.md` Step 3 updated from
  "NOT YET DECIDED" to this confirmed path — this was the last blocking
  gate; the Routine's first clean full cycle (config load → kill switch →
  sync → capacity → queue check → this gate) ran successfully and stopped
  here exactly as designed before this fix landed, confirming every prior
  piece of the pipeline works end to end.
- Decided the open "Testing vs Ready To Test after failed QA" question:
  story stays in "Testing" on a failed QA pass (does not bounce back to
  "Ready To Test"). Keeps board position stable while the bug subtask moves
  through the normal dispatch queue. `workflow-map.yaml`'s
  `transition_triggers` updated to state this as the one convention, not a
  decision left open.
- `automation-paused` label seeded into TFAE's label autocomplete (kill
  switch mechanism, see guardrails.yaml `kill_switch_label`).
- `QA Cycle Count` custom field created in Jira: Number field, scoped to
  TFAE + Story issue type only via field context. Confirmed field ID
  `customfield_10320` by querying TFAE-2 directly (field appears and is
  null/blank, as expected for a fresh story). `jql.yaml`'s
  `find_stories_over_qa_cap` now uses the real `cf[10320]` syntax instead
  of a placeholder field name.
- Added `needs_human_review_label` to `guardrails.yaml`, distinct from
  On Hold — On Hold means "human paused this," the new label means "the
  automation stopped itself because QA Cycle Count hit the cap."
- Added explicit `transition_triggers` to `workflow-map.yaml` — every
  status change now has a documented exact trigger, not just a status name,
  so a future Jira rename or workflow edit has something concrete to check
  against.

### Added
- Initial repo structure: `config/guardrails.yaml`, `config/jql.yaml`,
  `config/workflow-map.yaml`, `docs/writeup.md`, `docs/getting-started.md`.
- Guardrails set: `max_concurrent_agents: 3`, `dispatch_batch_size: 1`,
  `max_qa_cycles_per_story: 3`, `staleness_timeout_hours: 4`,
  `kill_switch_label: automation-paused`, `conductor_interval_minutes: 20`.
- `project_key: TFAE` established as the system's real scope boundary, since
  access is via personal Rovo connection rather than a scoped service
  account.

### Known open items (not yet resolved, tracked deliberately)
- No automated QA Routine yet — QA is manual per `docs/getting-started.md`
  Step 6, by design, not as a placeholder.
- No file-overlap detection for parallel dispatch — currently substituted
  with `max_dispatch_per_story_per_cycle: 1` in `guardrails.yaml`.
