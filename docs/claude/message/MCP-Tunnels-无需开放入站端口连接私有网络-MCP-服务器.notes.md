# MCP Tunnels：无需开放入站端口连接私有网络 MCP 服务器

*原文: [https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview) · 来源: web · 生成时间: 2026-09-20T06:25:25.561261+00:00*

## 背景

Claude 等 LLM 要通过 Model Context Protocol 调用工具时，许多企业的 MCP 服务器运行在防火墙、NAT 或内网之后。传统做法需要开放入站端口、配置公网暴露或维护 Anthropic IP 白名单，攻击面大且配置复杂。MCP tunnels 是 Anthropic 官方为这类私有网络接入场景提供的方案，基于 Cloudflare 的出站隧道实现内网服务可达。

## 痛点

没有 MCP tunnels 时，想让 Claude 访问私有网络中的 MCP 服务器，通常需要在防火墙上开入站端口、把服务暴露到公网或维护 IP 白名单。这不仅增加被扫描和攻击的风险，也可能不符合企业零信任、HIPAA 等合规要求。

## 解决办法

MCP tunnels 在客户网络内运行两个组件：cloudflared 和 Anthropic 的 proxy。cloudflared 主动向 Cloudflare 边缘发起出站连接，形成一条反向隧道，因此外网请求可以沿这条已建立的连接返回内网，而不需要任何入站端口。请求到达 proxy 后，proxy 终止内层 TLS、校验上游 IP 是否在允许范围，并根据 hostname 将流量路由到对应的 MCP server。安全上叠加三层：外层 mTLS 和 IP 校验防止未授权客户端到达隧道，内层 TLS 防止 Cloudflare 查看 payload，MCP server 自身的 OAuth 防止未授权工具调用。可以类比为内网主动拨出的加密电话，外网只能通过这条已建立的线路回拨，不能直接敲开服务器的大门。

## 关键代码示例

```yaml
tunnel: <your-tunnel-id>
credentials-file: /etc/cloudflared/<tunnel-id>.json

# 所有隧道子域流量先交给本地代理，由代理按 Host 路由到上游 MCP server
ingress:
  - hostname: docs.<your-tunnel-domain>
    service: http://proxy:8080
  - hostname: api.<your-tunnel-domain>
    service: http://proxy:8080
  - service: http_status:404
```

这段 cloudflared 配置把多个隧道子域名都转发到本地 proxy 的 8080 端口。外网请求经 Cloudflare 边缘进入已建立的出站隧道后，由本地 proxy 终止内层 TLS，再根据 HTTP Host 路由到具体的 MCP server。最后的 404 规则避免未声明的 hostname 落入默认服务。这体现了 outbound-only 连接与按 hostname 分发流量的核心原理。

## 关键流程

1. 在 Claude Console 或 API 创建隧道，获取 tunnel domain。
2. 选择认证方式：生产环境推荐 Workload Identity Federation 自动获取短期 token 并注册 CA；测试环境可手动配置 tunnel token 和 CA 证书。
3. 准备部署目标，例如 Kubernetes 集群或带 Docker Compose 的 VM，并确认出站网络满足要求。
4. 在私有网络内部署 cloudflared 和 Anthropic proxy 两个组件。
5. 将暴露的 MCP server 的 hostname，例如 docs.<your-tunnel-domain>，绑定到 Managed Agent 会话或通过 Messages API MCP connector 传入。
6. 如上游 MCP server 有自己的 OAuth 或 bearer token 认证，需要独立配置；隧道只负责加密传输，不替代 MCP 鉴权。

## 关键点

- outbound-only 连接是核心优势：cloudflared 主动连接 Cloudflare 边缘，内网无需开放任何入站端口，从拓扑上消除了入口暴露。
- 双组件架构中，cloudflared 负责出站传输，Anthropic proxy 负责终止内层 TLS、校验来源 IP 并按 hostname 路由，二者分离让传输提供商无法读取 payload。
- 三层安全模型分别解决不同风险：外层 mTLS 和 IP 校验防未授权客户端，内层 TLS 防中间人窃听，MCP server 上的 OAuth 防未授权工具调用。
- Cloudflare 只能看到连接元数据，例如 egress IP、cloudflared 指纹、连接时序、字节量和隧道子域，看不到 MCP 请求和响应内容，因为它不持有内层 TLS 私钥。
- 认证方式有两种：Programmatic access 通过 Workload Identity Federation 自动处理短期 token 和 CA 证书，更适合生产；Manual credentials 使用静态凭据，适合没有 OIDC 或仅测试的情况。
- 隧道加密流量但不负责身份认证：即使流量来自可信隧道，上游 MCP server 仍然需要自己的 OAuth 或 bearer token，以确认最终调用者是否被授权使用工具。

## 对比与权衡

- 相比直接在公网暴露 MCP 服务器并 allowlist Anthropic IP，MCP tunnels 不需要入站端口，显著减少公网攻击面，但需要信任 Cloudflare 作为传输子处理器，并额外维护 cloudflared 和 proxy 组件。
- 相比直接使用原始 Cloudflare Tunnel 暴露 HTTP 服务，MCP tunnels 增加了内层 TLS 和 Anthropic proxy 路由，能防止 Cloudflare 读取 payload 并自动按 hostname 分发到多个 MCP server，但它绑定 Anthropic 生态，目前属于 research preview。
- 相比传统 VPN 或专线打通内网，MCP tunnels 部署更轻、按 hostname 路由更细粒度，适合只暴露 MCP 服务；但在性能、网络路径可控性和合规细节上依赖第三方云传输。

## 自测问题

**问: 为什么 MCP tunnels 不需要开放入站端口？**

因为 cloudflared 在内网主动向 Cloudflare 边缘发起出站连接，建立长连接。外网请求沿这条已建立的连接反向回传，防火墙只需允许出站流量，不需要暴露任何入站端口。可以类比反向代理或内网主动拨号。

**问: Cloudflare 作为传输提供商为什么读不到请求内容？**

链路采用两层 TLS。外层 mTLS 保护 Anthropic 与传输提供商之间的边缘访问，内层 TLS 从 Anthropic 后端到客户 proxy，只有客户持有内层 TLS 私钥和证书。Cloudflare 只转发加密流量，因此无法解密 payload，只能看到连接元数据。

**问: MCP tunnel 可以替代 MCP server 自身鉴权吗？**

不能。隧道解决的是网络可达性和传输安全，不能证明最终调用者身份。即使流量来自可信隧道，仍需在 MCP server 上配置 OAuth 或 bearer token，确保只有被授权的调用者能使用工具。

**问: Programmatic access 和手动凭据怎么选？**

如果有 Kubernetes、云 IAM 或 SPIFFE 等 OIDC identity provider，并能管理 federation rules，优先选 Programmatic access。它能自动获取短期 API token、生成并注册 CA 证书，更安全且减少证书管理负担。没有 OIDC 或测试时用手动凭据，但要自己管理 tunnel token 和证书续期。

**问: 多个 MCP server 如何共享一个隧道入口？**

每个 MCP server 分配一个隧道子域的 hostname，例如 docs.<your-tunnel-domain>。cloudflared 将流量统一转发给本地 proxy，proxy 终止内层 TLS 后读取 HTTP Host 头，按照映射关系路由到不同上游 MCP server。这样通过一个隧道就可以暴露多个服务。

## 适用场景

- 企业内部 Kubernetes 或 VM 中运行 MCP server，需要让 Claude 安全访问但不希望公网暴露。
- 防火墙策略严格、只允许出站连接的零信任环境，适合通过出站隧道接入 Claude。
- 多个 MCP server 需要统一域名入口，并按不同 hostname 路由到不同内部服务。
- 需要评估 Zero Data Retention 或 HIPAA BAA 的企业场景，可结合该方案查看官方资格说明。

## 标签

`MCP` `Cloudflare Tunnel` `零信任` `Claude` `出站隧道`
