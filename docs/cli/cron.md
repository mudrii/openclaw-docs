---
summary: "CLI reference for `openclaw cron` scheduling commands"
read_when:
  - Inspecting cron jobs from scripts
  - Reading cron run state from JSON output
title: "`openclaw cron`"
---

# `openclaw cron`

Use `openclaw cron` to create, inspect, and maintain scheduled Gateway jobs.
See [Scheduled Tasks](/automation/cron-jobs) for the full scheduler model.

## JSON Status

As of `v2026.5.7`, `openclaw cron list --json` and
`openclaw cron show <job-id> --json` include a computed top-level `status`.

Possible values:

| Status | Meaning |
| --- | --- |
| `disabled` | The job is disabled. |
| `running` | The Gateway currently owns an active run for the job. |
| `ok` | The latest durable run completed successfully. |
| `error` | The latest durable run failed or timed out. |
| `skipped` | The latest durable run was skipped. |
| `idle` | No active run or terminal durable state is available. |

