# orchestration

Config and docs for the Jira (TFAE) → Cursor → manual QA pipeline. This repo
is the control surface — no script anywhere in the automation should define
a limit, a query, or a status mapping independently of what's here.

## Why this repo exists

Access to Jira is via a personal Rovo/Atlassian connection, not a scoped
service account — meaning the automation runs with full personal
permissions across every project, not just TFAE. Since the account itself
isn't a boundary, this repo is: every limit, every query, and every scope
restriction lives here in version control, so a change to any of them is a
visible, reviewable diff — never a silent edit inside a script.

## Structure

```
config/
  guardrails.yaml     the numbers: concurrency caps, QA cycle limit,
                       staleness timeout, kill switch, project_key
  jql.yaml             every JQL query used anywhere, named + documented
  workflow-map.yaml     TFAE's real statuses mapped to pipeline concepts
docs/
  writeup.md            what the pipeline does and why, guardrail by guardrail
  getting-started.md    the 8-step slow rollout guide
CHANGELOG.md            human-readable log of what changed and why
```

## The one rule

**`config/guardrails.yaml`'s `project_key` value is the real safety boundary
in this system.** Every JQL query in `config/jql.yaml` must interpolate it —
none may hardcode a project key separately. Any change to `project_key`
should be treated as the highest-risk edit possible in this repo, since it's
standing in for the project-scoped permissions a dedicated service account
would otherwise provide.

## Before changing anything in config/

- `guardrails.yaml`: know what you're loosening. Every value here exists
  because of a specific failure mode — see the table in `docs/writeup.md`.
- `jql.yaml`: a query change alters what the conductor considers "ready" —
  test it manually against real TFAE data before relying on it.
- `workflow-map.yaml`: only edit if TFAE's actual Jira workflow changes.
  Confirm the new status names by querying TFAE directly, don't guess.

Log every config change in `CHANGELOG.md`, even solo — it's the audit trail
for a system that has full personal permissions behind it.
