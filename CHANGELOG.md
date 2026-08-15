# Changelog

All notable changes to the orchestration config are logged here, even solo
changes — this is the audit trail for a system running on full personal
Jira permissions rather than a scoped service account.

## [Unreleased]

### Resolved
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
