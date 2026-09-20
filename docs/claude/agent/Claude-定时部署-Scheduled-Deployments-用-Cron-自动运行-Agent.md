---
title: Claude 定时部署（Scheduled Deployments）：用 Cron 自动运行 Agent
url: https://platform.claude.com/docs/en/managed-agents/scheduled-deployments
source_type: web
folder: claude/agent
author: null
tags:
- Scheduled Deployments
- Claude API
- Cron
- Managed Agents
- 自动化调度
summary: 用 Cron 表达式定时触发 Claude Agent 会话，结合预算、运行记录与生命周期管理实现无人值守自动化任务。
fetched_at: '2026-09-20T07:48:36.135726+00:00'
---

Create and manage deployments with the Claude API: run an agent on a recurring cron schedule and inspect its run history.

A **scheduled deployment** allows an [agent](https://platform.claude.com/docs/en/managed-agents/agent-setup) to start [sessions](https://platform.claude.com/docs/en/managed-agents/sessions) autonomously, enabling task completion over a predictable cadence. You create and manage deployments with the Deployments API, part of the Claude API.

For the launch context and examples of what teams run on schedules, see [scheduled deployments and vaults in Claude Managed Agents](https://claude.com/blog/whats-new-in-claude-managed-agents) on the blog.

## Create a scheduled deployment

When creating a deployment, you pass the [session configurations](https://platform.claude.com/docs/en/managed-agents/sessions) required for execution, in addition to a `schedule`.

*   Deployments require [agent configuration](https://platform.claude.com/docs/en/managed-agents/agent-setup) and [environment configuration](https://platform.claude.com/docs/en/managed-agents/environments), and optionally accept [files](https://platform.claude.com/docs/en/managed-agents/files), [GitHub](https://platform.claude.com/docs/en/managed-agents/github), [memory stores](https://platform.claude.com/docs/en/managed-agents/memory), and [vaults](https://platform.claude.com/docs/en/managed-agents/vaults). A deployment that targets a [self-hosted environment](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes#use-memory-stores) can attach memory stores; `file` and `github_repository` resources require a cloud environment. The Claude Console deployment form does not currently offer memory stores for self-hosted environments; attach them through the API or an SDK instead.
*   Deployments also require at least one initial event, a `user.message` or `user.define_outcome`, that starts each session's work.
*   In the `schedule`, you define a cron `expression` and a `timezone`. Maximum granularity supported is at the minute level.

The response includes a deployment object with a populated `schedule.upcoming_runs_at` with the next upcoming fire times, to confirm your schedule was set correctly.

The upcoming run timestamps reflect the exact schedule configured. However, to distribute load, actual execution applies jitter of up to 15% of the interval between runs, with a minimum of 5 seconds and a maximum of 9 minutes.

A maximum of **1,000 scheduled deployments** is supported per organization. Contact Anthropic support if you need more.

See the [Create Deployment reference](https://platform.claude.com/docs/en/api/beta/deployments/create) for full parameters and response schema.

### Cron and timezone semantics

*   **Expression:** Standard POSIX cron (`minute hour day-of-month month day-of-week`). You can generate and validate these cron expressions in the [Claude Console](https://platform.claude.com/workspaces/default/deployments).
*   **Timezone:** IANA timezone identifier (for example, `"America/Los_Angeles"`).
*   **DST:** Cron schedules use literal wall-clock matching, so `"0 20 * * *"` in `America/New_York` fires at 8:00 PM local time regardless of whether EST or EDT is in effect.

### Set a budget on each run

Pass the optional `budget` object when you create or update the deployment. It takes the same shape as a [session budget](https://platform.claude.com/docs/en/managed-agents/budgets). The deployment copies the cap onto each session it starts, so the budget bounds every run separately rather than acting as a cumulative ceiling across runs: a deployment with a `"2000"` cap can spend up to about $20 on every run.

A session started by the deployment behaves exactly like any other budgeted session: it pauses with `budget_reached` when its own list cost [reaches the cap](https://platform.claude.com/docs/en/managed-agents/budgets#when-a-session-reaches-its-budget). Changing the deployment's budget applies to runs started afterward; a session already running keeps the cap it started with, which you can [change through the session itself](https://platform.claude.com/docs/en/managed-agents/session-operations#updating-the-session-budget). Unlike a session budget, a deployment's budget can be removed with `"budget": null` and set again later.

The following example sets a budget on an existing deployment:

## Deployment runs

Deployments can fail to trigger for a variety of reasons: for example, if the `environment` resource has been archived, or if session creation is rate-limited. Each attempt at executing a deployment generates a **deployment run** record, allowing you to track successes and failures independent of the session lifecycle.

Successful deployments generate active sessions, and a successful deployment run contains the associated `session_id`. To follow a session's lifecycle, track the session events through the [event stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming) or [webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks). Deployment lifecycle changes and the outcome of each scheduled run are also delivered as webhook events, listed in the Deployment events and Deployment run events tabs of [Supported event types](https://platform.claude.com/docs/en/managed-agents/webhooks#supported-event-types).

List all deployment runs for a deployment as follows:

You can additionally filter on deployment runs with errors:

A failed run includes an `error` with a `type` describing why session creation was rejected (for example, `environment_archived_error`, `agent_archived_error`, or `session_rate_limited_error`). See the [List Deployment Runs reference](https://platform.claude.com/docs/en/api/beta/deployment_runs/list) for all filter parameters and the response schema.

To retrieve a single run by ID, call [`GET /v1/deployment_runs/{deployment_run_id}`](https://platform.claude.com/docs/en/api/beta/deployment_runs/retrieve). A [`deployment_run` webhook event](https://platform.claude.com/docs/en/managed-agents/webhooks#supported-event-types) carries the run ID as its `data.id`.

## Managing deployment lifecycle

Each lifecycle change emits a [webhook event](https://platform.claude.com/docs/en/managed-agents/webhooks#supported-event-types), so you can react to a paused, unpaused, or archived deployment without polling; see the Deployment events tab.

**Pause** suppresses scheduled triggers on a go-forward basis; running sessions from a prior deployment run continue to execute. Manual runs through the `run` endpoint are still allowed while paused. Pausing sets `paused_reason` to `{"type": "manual"}`; unpausing clears it.

**Unpause** resumes the schedule from the next scheduled occurrence. Missed triggers are not backfilled.

**Archive**, unlike **pause**, is terminal: the schedule terminates and the deployment cannot be modified.

### Failure behavior

Session creation rate-limit responses are recorded immediately as a `session_rate_limited_error` run without retry; the schedule attempts again at the next scheduled occurrence. Rate limits on underlying API calls within a session are handled by the session itself.

If a deployment's agent has been archived, the deployment is automatically archived in the same operation. If the agent has been deleted, the next scheduled trigger detects the missing agent and automatically archives the deployment. In both cases no deployment run is recorded. If a subagent referenced by the agent has been archived, the next trigger records a failed run with `error.type: "agent_archived_error"` and the deployment is automatically paused so you can update the agent and resume. Other unrecoverable session-creation errors, such as an archived environment or vault, behave the same way: the trigger records a failed run and the deployment is automatically paused. The deployment's `paused_reason.error.type` mirrors the failed run's `error.type`.

## Trigger a manual run

To run a deployment outside its schedule, call the [`run` endpoint](https://platform.claude.com/docs/en/api/beta/deployments/run). This creates a session immediately and writes a deployment run with `trigger_context.type: "manual"`. This allows you to test a deployment before committing to the schedule.