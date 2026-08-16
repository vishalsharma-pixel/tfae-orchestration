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

### Resolved
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
