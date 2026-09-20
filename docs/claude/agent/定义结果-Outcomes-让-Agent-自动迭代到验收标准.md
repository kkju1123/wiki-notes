---
title: 定义结果（Outcomes）：让 Agent 自动迭代到验收标准
url: https://platform.claude.com/docs/en/managed-agents/define-outcomes
source_type: web
folder: claude/agent
author: null
tags:
- Agents
- Outcomes
- Grader
- Rubric
- Claude API
summary: 通过定义可验收的结果和评分标准，让托管代理自动迭代直到交付物满足质量要求，减少人工审查与返工。
fetched_at: '2026-09-20T07:39:18.794672+00:00'
---

Tell the agent what 'done' looks like, and let it iterate until it gets there.

An outcome tells the session what the end result should look like and how to measure its quality. The agent works toward that target, self-evaluating and iterating until the outcome is met.

When you define an outcome, the harness automatically provisions a _grader_ to evaluate the artifact against a rubric. The grader uses a separate context window to avoid being influenced by the main agent's implementation choices.

The grader returns an explanation summarizing which criteria passed or failed, or confirming that the artifact satisfies the rubric. That feedback is handed back to the agent for the next iteration.

## Create a rubric

A rubric is a markdown document describing per-criterion scoring. The rubric is required.

Example rubric:

Pass the rubric as inline text on `user.define_outcome` (see [Create a session with an outcome](https://platform.claude.com/docs/en/managed-agents/define-outcomes#create-a-session-with-an-outcome)), or upload it through the Files API for reuse across sessions.

## Create a session with an outcome

The following examples create a [session](https://platform.claude.com/docs/en/managed-agents/sessions) for an existing [agent](https://platform.claude.com/docs/en/managed-agents/agent-setup) and [environment](https://platform.claude.com/docs/en/managed-agents/environments) (both created separately), then send a `user.define_outcome` event. The agent begins work immediately. No additional user message event is required.

## Outcome events

Progress on an outcome-oriented session is surfaced on the events [stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming).

*   `agent.*` events (such as messages and tool use) show progress toward the outcome.
*   `span.outcome_evaluation_*` events are only emitted for outcome-oriented sessions and show the number of iteration loops and the grader's feedback process.
*   You can also send `user.message`[events](https://platform.claude.com/docs/en/managed-agents/reference#event-types) to an outcome-oriented session to direct the agent's work as it progresses, but it isn't required: the agent works toward the outcome on its own, iterating until it succeeds or runs out of iterations.
*   A `user.interrupt` event pauses work on the current outcome and marks the `span.outcome_evaluation_end.result` as `interrupted`, allowing you to kick off a new outcome.
*   After the final outcome evaluation, the session can be continued as a conversational session, or a new outcome can be started. The session retains history of the prior outcome.

### Define outcome user event

This is the event you send to initiate an outcome. It is echoed back on receipt, including a `processed_at` timestamp and `outcome_id`.

### Outcome evaluation start

Emitted once the grader starts an evaluation over one iteration loop. The `iteration` field is a 0-indexed revision counter: `0` is the first evaluation, `1` is the re-evaluation after the first revision, and so on.

### Outcome evaluation ongoing

Heartbeat emitted while the grader runs. The grader's internal reasoning is opaque: you see that it's working, not what it's thinking.

### Outcome evaluation end

Emitted when an outcome evaluation cycle ends: after the grader finishes evaluating one iteration, or when the session is interrupted while an outcome is active. The `result` field indicates what happens next.

| Result | Next |
| --- | --- |
| `satisfied` | Session transitions to `idle`. |
| `needs_revision` | Agent starts a new iteration cycle. |
| `max_iterations_reached` | One final acknowledgment turn follows before the session transitions to `idle`. No further evaluation runs. |
| `failed` | Session transitions to `idle`. Returned when the rubric does not apply to the deliverables, for example if the description and rubric contradict each other. |
| `interrupted` | Emitted when the session is interrupted while an outcome is active, even if evaluation hadn't started yet. If no `outcome_evaluation_start` fired before the interrupt, `outcome_evaluation_start_id` is an empty string. |

## Check outcome status

You can either listen on the [event stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming) for `span.outcome_evaluation_end`, or poll `GET /v1/sessions/{session_id}` and read `outcome_evaluations[].result`. Until an evaluation completes, `result` reports `pending`, `running`, or `evaluating`:

## Retrieve deliverables

The agent writes output files to `/mnt/session/outputs/` inside the sandbox. To retrieve them, list files through the [Files API](https://platform.claude.com/docs/en/build-with-claude/files) with the session ID as the `scope_id`, then download them by ID. Filtering by `scope_id` requires the `managed-agents-2026-04-01` beta header on the list request, so the SDK and CLI examples make that call through the `beta` namespace and pass the header explicitly. Files appear in the list shortly after the agent finishes writing them, sometimes a few seconds after the session goes idle. If a file you expect is not listed yet, list again after a short delay; once it appears in the list, its upload has finished.

## Next steps

Register per-user credentials when creating sessions.

Send events, stream responses, and interrupt or redirect your session mid-execution.

Upload files and mount them in your sandbox for reading and processing.