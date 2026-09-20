---
title: 使用 Vaults 管理第三方服务认证凭据
url: https://platform.claude.com/docs/en/managed-agents/vaults
source_type: web
folder: claude/agent
author: null
tags:
- vaults
- secrets management
- MCP
- OAuth
- agent authentication
summary: Claude 平台用 Vaults 托管用户级第三方凭据，按会话引用自动注入，避免 agent 接触明文并支持无重启轮换。
fetched_at: '2026-09-20T07:41:02.298620+00:00'
---

Register per-user credentials when creating sessions.

Vaults and credentials are authentication primitives that let you register credentials for third-party services once and reference them by ID at session creation. This means you don't need to run your own secret store, transmit tokens on every call, or lose track of which end user an agent acted on behalf of.

The vault reference is a per-session parameter, so you can manage your product at the `agent` resource granularity and your users at the `session` resource granularity.

## Create a vault

A vault is the collection of `credentials` associated with an end user. Give it a `display_name` and optionally tag it with `metadata` so you can map it back to your own user records.

The response is the full vault record:

## Add a credential

Two credential categories are supported:

*   **MCP credentials** (`mcp_oauth`, `static_bearer`): each credential is keyed by an `mcp_server_url`. When the agent connects to a server at that URL at session runtime, the token is injected automatically.
*   **Environment variables** (`environment_variable`): each credential is keyed by a `secret_name` (the environment variable name) and stored in the sandbox as an opaque placeholder. When the agent initiates an outbound request, the opaque placeholder is substituted with the real secret at egress. The agent never sees the secret value. Use this for any service that authenticates through an environment variable, such as CLIs, SDKs, or direct API calls.

The actual credential values you supply (`token`, `access_token`, `refresh_token`, `client_secret`, `secret_value`) are treated as sensitive, write-only fields and never returned in API responses.

Credentials are stored as provided and are not validated until session runtime. An invalid credential surfaces as an authentication or downstream error during the session, which is emitted but does not block the session from continuing.

Constraints:

*   **Unique key per vault.**`mcp_server_url` (MCP credentials) and `secret_name` (environment variable credentials) must be unique among active credentials in a vault. Creating a duplicate returns a 409.
*   **Keys are immutable.** To change `mcp_server_url` or `secret_name`, archive the credential and create a new one.
*   **Maximum 20 credentials per vault.**

## Reference the vault at session creation

Pass `vault_ids` when creating a session:

Runtime behavior:

*   When no MCP credential matches by `mcp_server_url`, the connection is attempted unauthenticated and will error if the server requires authentication.
*   When multiple vaults contain a matching credential, the first vault with a match wins.
*   In [multiagent sessions](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration), vault credentials apply to every thread. An agent whose own definition declares the matching MCP server authenticates with these credentials. See [Connect agents to MCP servers](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration#connect-agents-to-mcp-servers).

## Rotate a credential

Secret values, `display_name`, and (on environment variable credentials) `injection_location` can be updated. `injection_location` updates merge per field, as described in the Environment variable tab of [Add a credential](https://platform.claude.com/docs/en/managed-agents/vaults#add-a-credential). For a running session, an `injection_location` update propagates the same way as a secret rotation: the session's credentials are re-resolved without a restart, as described in [Credential lifecycle](https://platform.claude.com/docs/en/managed-agents/vaults#credential-lifecycle), and the updated locations apply to the session's subsequent outbound requests. Structural fields (`mcp_server_url`, `secret_name`, `token_endpoint`, `client_id`) are locked after creation. To change them, archive the credential and create a new one.

## Credential lifecycle

Credentials are re-resolved periodically, both during a session and during the vault lifecycle. This ensures that credential rotation, archival, or deletion propagates to running sessions without a restart.

To be notified if a credential is archived, deleted, or fails to refresh, you can subscribe to the vault and credential [webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks) associated with those lifecycle changes.

| Event | Trigger |
| --- | --- |
| `vault.archived` | Vault archived. A `vault_credential.archived` event is also emitted for each underlying credential. |
| `vault.deleted` | Vault deleted. A `vault_credential.deleted` event is also emitted for each underlying credential. |
| `vault_credential.archived` | Credential archived, either directly or as a result of vault archival. |
| `vault_credential.deleted` | Credential deleted, either directly or as a result of vault deletion. |
| `vault_credential.refresh_failed` | An `mcp_oauth` credential cannot be refreshed (invalid refresh token, or irrecoverable error from the OAuth server). |

For `mcp_oauth` credentials, re-resolution also refreshes the access token if it has expired. If the refresh fails, a `vault_credential.refresh_failed` event is emitted.

### Diagnose an OAuth refresh failure

To diagnose why a refresh failed, call `POST /v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate` (or `client.beta.vaults.credentials.mcp_oauth_validate(...)` in the SDK). This lets you decide how to handle the failure; the right action depends on the error type.

The top-level `status` tells you what to do next:

*   `valid`: the token works; no action needed.
*   `invalid`: the grant is gone or the OAuth server rejected the refresh with a 4xx. Prompt the end user to re-authorize.
*   `unknown`: a transient error (5xx, 429, or network failure). Wait and retry.

The response is a `vault_credential_validation` object. `mcp_probe` includes the failed MCP handshake step; `refresh` includes the outcome of the attempted refresh.

## Other operations

*   **List vaults or credentials:** Paginated, newest first. Archived records are excluded by default (pass `include_archived=true` to include them).
*   **Archive a vault:**`POST /v1/vaults/{id}/archive`. Cascades to all credentials. Secrets are purged; records are retained for auditing. Future sessions referencing this vault fail; running sessions continue.
*   **Archive a credential:**`POST /v1/vaults/{id}/credentials/{cred_id}/archive`. Purges the secret payload; the credential key (`mcp_server_url` or `secret_name`) remains visible and is freed for a replacement credential.
*   **Delete a vault or credential:** Hard delete. The record is not retained. Use archive if you need an audit trail.