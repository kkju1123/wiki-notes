---
title: Define your Agent：可复用、版本化的 Claude 智能体配置
url: https://platform.claude.com/docs/en/managed-agents/agent-setup
source_type: web
folder: claude/agent
author: null
tags:
- Claude Managed Agents
- Agent 配置
- 版本控制
- MCP
- inference_geo
summary: 介绍 Claude Managed Agents 的 agent 配置字段、创建、版本化更新、推理地理固定与 session 覆盖机制。
fetched_at: '2026-09-20T07:08:18.226459+00:00'
---

Create a reusable, versioned agent configuration.

An agent is a reusable, versioned configuration that defines persona and capabilities. It bundles the model, system prompt, tools, MCP servers, and skills that shape how Claude behaves during a session.

Create the agent once as a reusable resource and reference it by ID each time you [start a session](https://platform.claude.com/docs/en/managed-agents/sessions). Agents are versioned and easier to manage across many sessions.

## Agent configuration fields

| Field | Description |
| --- | --- |
| `name` | Required. A human-readable name for the agent. |
| `model` | Required. The Claude [model](https://platform.claude.com/docs/en/models/overview) that powers the agent. Accepts a model ID string or an object, for example `{"id": "claude-opus-5"}`. Claude 4.5 and later models are supported. The object form also accepts `speed`, `effort`, and `inference_geo` fields; see the tips under [Create an agent](https://platform.claude.com/docs/en/managed-agents/agent-setup#create-an-agent), [Effort levels](https://platform.claude.com/docs/en/build-with-claude/effort#effort-levels), and [Pin the inference geo](https://platform.claude.com/docs/en/managed-agents/agent-setup#pin-the-inference-geo). |
| `system` | A [system prompt](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#give-claude-a-role) that defines the agent's behavior and persona. The system prompt is distinct from [user messages](https://platform.claude.com/docs/en/managed-agents/reference#event-types), which should describe the work to be done. |
| `tools` | The tools available to the agent. Combines [pre-built agent tools](https://platform.claude.com/docs/en/managed-agents/tools), [MCP tools](https://platform.claude.com/docs/en/managed-agents/mcp-connector), and [custom tools](https://platform.claude.com/docs/en/managed-agents/tools#custom-tools). |
| `mcp_servers` | [MCP servers](https://platform.claude.com/docs/en/managed-agents/mcp-connector) that provide standardized third-party capabilities. |
| `skills` | [Skills](https://platform.claude.com/docs/en/managed-agents/skills) that supply domain-specific context with progressive disclosure. |
| `multiagent` | A coordinator declaration listing the agents this agent can delegate to. See [Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration). |
| `description` | A description of what the agent does. |
| `metadata` | Arbitrary key-value pairs for your own tracking. |

You can also override `model`, `system`, `tools`, `mcp_servers`, and `skills` for a single session without changing the agent. An `effort` level set inside a per-session `model` override isn't applied, and because the override replaces the agent's `model` object in full, a session created with a `model` override runs at the model's default effort level; to run at a specific effort level, set `effort` on the agent and don't override `model` for that session. See [Override agent configuration for a session](https://platform.claude.com/docs/en/managed-agents/sessions#override-agent-configuration-for-a-session).

## Create an agent

The following example defines a coding agent that uses Claude Opus 5 with access to the pre-built agent toolset. The toolset lets the agent write code, read files, search the web, and more. See the [agent tools reference](https://platform.claude.com/docs/en/managed-agents/tools) for the full list of supported tools.

The examples use curl, the `ant` CLI, or one of the SDKs. If you haven't set one up, the [quickstart](https://platform.claude.com/docs/en/managed-agents/quickstart#install-the-cli) covers installation and client setup.

The response echoes your configuration and adds `id`, `type`, `version`, `created_at`, `updated_at`, and `archived_at` fields, and fills in `model` fields you omit, such as `effort`, with their defaults. The `version` starts at 1 and increments each time an update changes the agent.

The `default_config` on the toolset shows its default [permission policy](https://platform.claude.com/docs/en/managed-agents/permission-policies), `always_allow`, which applies unless you configure one.

### Pin the inference geo

Like `speed` and `effort`, `inference_geo` is set through the object form of `model`: pass `model` as an object and set `inference_geo` alongside `id`. The field accepts `"us"` or `"global"`. When it's unset, each model request follows the workspace's default inference geo at the time it's served. See [Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency) for the workspace-level geo controls and pricing.

The following example pins an agent to US inference and prints the `inference_geo` value from the agent's `model` object:

An `inference_geo` pin is validated against the workspace's [`allowed_inference_geos`](https://platform.claude.com/docs/en/manage-claude/data-residency#workspace-level-restrictions) when the agent is saved, when a session is created from it, and on every turn the session serves. If the workspace allowlist narrows so a pin is no longer allowed, new sessions can't be created from the agent and running sessions refuse further turns; pins are never exempted, because workspaces rely on them for compliance and data residency.

Setting `inference_geo` on a model that doesn't support geographic inference pinning returns a 400 error; see [Model availability](https://platform.claude.com/docs/en/manage-claude/data-residency#model-availability) for the models that do. In a `multiagent` configuration, the coordinator's pin and every roster member's must all be set to the same value or all be unset; see [Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration). To change or clear the pin later, update the agent's `model` object; supplying `model` without `inference_geo` clears it, as described under [Update semantics](https://platform.claude.com/docs/en/managed-agents/agent-setup#update-semantics).

## Update an agent

Updating an agent generates a new version when the configuration changes. The `version` field is optional: supply it for optimistic concurrency (a mismatch returns a 409), or omit it to apply the update unconditionally (last write wins). Updates to archived agents are rejected.

With the CLI, edit the agent's file and run `ant apply` again; apply supplies `version` for you.

The preceding example supplies `version` from the create response, so the update only applies if nothing else has changed the agent since you read it. To apply an update unconditionally, omit `version` from the request:

### Update semantics

*   **`version`** is optional and must be at least 1 when supplied. When supplied, the request returns a 409 if it doesn't match the agent's current version, even when the fields you send already match the stored values; re-read the agent and retry. When omitted, the update applies unconditionally and the most recent update silently replaces any concurrent one, with no error to either caller. Supplying `version` is the recommended default for interactive callers, and omitting it fits declarative apply loops, such as a CI job that syncs checked-in agent definitions, where the loop owns the agent.

*   **Omitted fields are preserved.** You only need to include the fields you want to change.

*   **Scalar fields** (`model`, `system`, `name`, `description`) are replaced with the new value. `system` and `description` can be cleared by passing `null`. `model` and `name` are mandatory and cannot be cleared. Within a `model` object you supply, `effort` is the sole exception: if the model `id` is unchanged, omitting `effort` leaves the stored effort level unchanged. If you change the model `id`, an omitted `effort` resets to the new model's default. Other `model` fields are replaced along with the object: supplying `model` without `inference_geo` clears the agent's inference geo pin.

*   **Array fields** (`tools`, `mcp_servers`, `skills`) are fully replaced by the new array. To clear an array field entirely, pass `null` or an empty array.

*   **`multiagent`** is replaced as a whole, including its `agents` roster. Pass `null` to clear it.

*   **Metadata** is merged at the key level. Keys you provide are added or updated. Keys you omit are preserved. To delete a specific key, set its value to `null`.

*   **No-op detection.** If the update produces no change relative to the current version, no new version is created and the existing version is returned.

*   **Coordinator rosters are not updated.** Coordinators that reference this agent in their `multiagent.agents` roster keep the version that was pinned when the coordinator was created or last updated, even if the reference omits `version`. To delegate to the new version, [update the coordinator](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration#configure-the-coordinator) so its roster references it.

## Agent lifecycle

| Operation | Behavior |
| --- | --- |
| **Update** | Generates a new agent version when the configuration changes. |
| **List versions** | Returns the full version history so you can track changes over time. |
| **Archive** | Makes the agent read-only. New sessions cannot reference it, but existing sessions continue to run. |

### List versions

Fetch the full version history to track how an agent has changed over time. Results are paginated, and the SDK examples fetch every page automatically.

### Archive an agent

Archiving makes the agent read-only and cannot be undone. Existing sessions continue to run, but new sessions cannot reference the agent. The response sets `archived_at` to the archive timestamp.

## Next steps

Configure tools available to your agent.

Attach reusable, filesystem-based expertise to your agent for domain-specific workflows.

Create a session to run your agent and begin executing tasks.

Event types, self-hosted worker CLI flags, supported MCP server types, rate limits, and branding guidelines for Claude Managed Agents.