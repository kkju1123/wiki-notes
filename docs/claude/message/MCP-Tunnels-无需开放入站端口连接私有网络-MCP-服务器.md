---
title: MCP Tunnels：无需开放入站端口连接私有网络 MCP 服务器
url: https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview
source_type: web
folder: claude/message
author: null
tags:
- MCP
- Cloudflare Tunnel
- 零信任
- Claude
- 出站隧道
summary: 介绍 Anthropic 的 MCP 隧道如何通过 cloudflared 出站连接和本地代理，让 Claude 安全访问私有网络中的 MCP 服务器，无需公网暴露。
fetched_at: '2026-09-20T06:25:25.561261+00:00'
---

Securely connect Claude to MCP servers running in your private network without opening inbound ports or exposing services to the public internet.

MCP tunnels let you connect Claude to Model Context Protocol (MCP) servers that run inside your private network. Traffic flows over an outbound-only connection, so you don't need to open inbound firewall ports, expose services to the public internet, or allowlist Anthropic's IP ranges on your origin.

For Zero Data Retention and HIPAA BAA eligibility, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#feature-eligibility).

## How it works

The [tunnel stack](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components) is two components that run inside your network:

*   **[cloudflared](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components):** Cloudflare's open-source tunnel connector. It initiates outbound-only connections to the [tunnel edge](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components) and carries encrypted traffic from Anthropic to your proxy.
*   **[Proxy](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components):** Anthropic's routing component. It terminates [inner TLS](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components), validates that upstream IPs fall within an allowed range, and routes each request to the correct [upstream MCP server](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#components) based on hostname.

Each MCP server you expose gets a hostname under your tunnel domain (for example, `docs.<your-tunnel-domain>`). You attach these hostnames to a Managed Agent session in the Claude Console, or pass them to the Messages API through the [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector).

## Prerequisites

Before deploying, make sure you have:

*   A deployment target: a Kubernetes cluster, or a VM with Docker and Docker Compose.
*   A tunnel. Create one in the Claude Console (see [Create a tunnel](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/console#create-a-tunnel)) or through the API; the Helm chart's setup hook can also create one for you during install.
*   A way for your stack to authenticate to the Tunnels API. Choose one:
    *   **[Programmatic access](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#credential-provisioning) (recommended).** Set up [Workload Identity Federation](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) when you create the tunnel. Your stack mints short-lived API tokens from your identity provider, fetches the tunnel token, and generates and registers a CA certificate automatically. Requires permission to manage federation rules, a registered OIDC issuer, and a federation rule with the `workspace:manage_tunnels` scope.
    *   **[Manual](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts#credential-provisioning).** Supply static credentials yourself: the tunnel token from the Console and a server certificate signed by a CA you register there. See [Get the connection details](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/console#get-the-connection-details) and [Add a CA certificate](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate).

*   One or more MCP servers running in your private network. See [Remote MCP servers](https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers) for examples.
*   Outbound connectivity as listed under [Network requirements](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview#network-requirements).

### Network requirements

| Component | Destination | Port / protocol | Used during |
| --- | --- | --- | --- |
| Setup component | `api.anthropic.com` | 443 TCP | Provisioning and token rotation |
| cloudflared | Tunnel edge (`198.41.192.0/19`, `2606:4700:a0::/44`) | 7844 TCP and UDP | Runtime |
| Proxy | Your upstream MCP servers | As configured | Runtime |

## Security model

### Security layers

Three independent layers protect every request:

| Layer | Protects against |
| --- | --- |
| Outer mTLS between Anthropic and the transport provider, with IP validation | Unauthorized clients reaching the tunnel |
| Inner TLS from Anthropic's back end to your proxy | Payload inspection by the transport provider or any network intermediary |
| OAuth on each MCP server | Unauthorized use of MCP tools by authenticated tunnel traffic |

The tunnel transport runs on Cloudflare's network. Because the proxy terminates inner TLS using a certificate that only you hold, Cloudflare cannot read request or response payloads. Anthropic does not connect to a tunnel until a CA certificate is registered, so payloads are always encrypted when they cross Cloudflare's network. Cloudflare does receive connection metadata; see [What the transport provider can observe](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview#what-the-transport-provider-can-observe).

### Shared responsibility model

| Anthropic handles | Your organization handles |
| --- | --- |
| Tunnel access control | All content and traffic that transits your tunnel, and compliance with applicable third-party acceptable-use policies (including Cloudflare's) |
| Validating your CA certificate before connecting to your proxy | Adherence to the deployment guidance on these pages |
| Ensuring Claude only sends requests to tunnels owned by your organization | Securing tunnel tokens and TLS private keys |
|  | Managing the server certificate and renewing it before it expires |
|  | Configuring OAuth on each MCP server |
|  | Restricting network access for the proxy and MCP servers |
|  | Notifying Anthropic if you suspect a breach |

### What the transport provider can observe

Cloudflare provides the outbound transport. It cannot read MCP request or response payloads, but it does receive the following connection metadata:

*   the egress IP address of the host running cloudflared
*   a cloudflared host fingerprint
*   connection timing and byte-volume
*   the `*.tunnel.anthropic.com` subdomain assigned to your tunnel

Anthropic's agreement with Cloudflare restricts Cloudflare's use of this telemetry. Cloudflare acts as a subprocessor for this research preview.

## Deploy a tunnel

If you're new to MCP tunnels, start with the quickstart to get a working tunnel locally before configuring a production deployment.

The shortest path to a working tunnel: Docker Compose with a sample MCP server.

Install on a Kubernetes cluster using the Anthropic Helm chart.

Install on a VM using Docker Compose.

Choosing between them:

*   **Deployment target**
    *   **Helm** when deploying to Kubernetes.
    *   **Docker Compose** for a single host or local testing.

*   **Authentication for setup**
    *   **Programmatic access** (through Workload Identity Federation) when you have an OIDC identity provider such as a Kubernetes cluster, cloud IAM, or SPIFFE.
    *   **Manual credentials** when you don't, or when you're testing.

## Use the tunneled MCP servers

Once your tunnel is active (it has an active CA certificate and your tunnel stack is connected), the upstream MCP servers are reachable from Claude Managed Agents and the Messages API.

In both cases, the tunnel carries encrypted traffic to your MCP server but does not authenticate to it. If the upstream MCP server requires its own authentication (OAuth, bearer token), supply it the same way you would for any other MCP server; it is independent of the tunnel.

### Managed Agents (Console)

1.   In **Managed Agents > Sessions**, create a session and choose **Create new agent** so you can edit the MCP server list.
2.   Click **+ MCP Server** and open the dropdown. Tunnels in the session's workspace that have at least one active certificate appear at the top of the list, above the public connector catalog.
3.   Select the tunnel and supply the **Subdomain** that your proxy routes to a specific MCP server, and the **Path** the upstream MCP server expects. The **Resolves to** line shows the exact URL.

### Messages API

Pass the upstream MCP server's URL in the `mcp_servers` array, the same way as any other remote MCP server. The request body and `anthropic-beta` header follow the standard [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) format; only the `url` is tunnel-specific. The following example uses the MCP connector's `mcp-client` beta header, which is separate from the `mcp-tunnels` beta used by the [Tunnels API](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/reference). Make the request in the workspace the tunnel was created in by using an API key for that workspace or, if your key has access to multiple workspaces, by setting the [`anthropic-workspace-id` header](https://platform.claude.com/docs/en/manage-claude/authentication#select-a-workspace) to that workspace.

The URL's host is `<subdomain>.<your-tunnel-domain>`. The path depends on your upstream MCP server, not the tunnel: FastMCP's `streamable-http` transport serves at `/mcp`, and other servers may use `/` or a custom path (check the server's documentation). The proxy forwards the path untouched.

For authenticating to the upstream MCP server (`authorization_token`) and other `mcp_servers` options, see [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector).

## Next steps

Hardening guidance, credential rotation, and breach response.

Diagnose connectivity, TLS, and routing issues.

Proxy config fields, the Tunnels API, certificate requirements, and the setup component.

Use tunneled servers from the Messages API.