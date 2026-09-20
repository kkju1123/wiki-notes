# Claude Managed Agents 参考：事件类型、自托管 Worker、MCP、限流与品牌规范

*原文: [https://platform.claude.com/docs/en/managed-agents/reference](https://platform.claude.com/docs/en/managed-agents/reference) · 来源: web · 生成时间: 2026-09-20T07:51:23.374896+00:00*

## 背景

Claude Managed Agents 允许开发者通过 API 创建由 Claude 驱动的代理，处理长任务、工具调用、审批和多方交互。为了让集成方在不同运行环境中正确接入，需要一份权威参考，统一事件类型、Worker 参数、MCP 连接方式和运营限制。本文就是这些约定的集中说明，避免集成时靠猜测。

## 痛点

如果没有这份参考，集成者会对事件命名和流式增量混淆，无法正确驱动会话；自托管 Worker 参数配置错误可能导致工具无法访问、空闲超时失控或误把工作目录当沙箱；MCP 传输选错、限流未处理、品牌违规则会造成生产事故和合规风险。

## 解决办法

事件类型采用 `{domain}.{action}` 命名，持久化事件可通过 API 列出/检索，流式 delta 只用于实时增量推送，客户端需要同时处理两种数据结构。自托管 worker 通过 `ant beta:worker` 轮询指定 environment，用 environment key 认证，在工作目录内执行文件工具和自定义脚本；`--workdir` 只是文件工具防护，不是安全边界。MCP 连接优先使用 streamable HTTP transport，私有服务通过 MCP 隧道暴露，旧的 SSE 只作为兼容回退。组织级创建接口 300 rpm、读取接口 1200 rpm 的限流要求客户端做限速和重试。品牌上允许 'Claude Agent' 和 'Powered by Claude'，禁止冒充 Claude Code 等产品。

## 关键代码示例

```bash
export ANTHROPIC_ENVIRONMENT_ID=env_123
export ANTHROPIC_ENVIRONMENT_KEY=key_xxx
ant beta:worker --environment-id $ANTHROPIC_ENVIRONMENT_ID --environment-key $ANTHROPIC_ENVIRONMENT_KEY --workdir /workspace --unrestricted-paths /tmp/data,/var/log --max-idle 120s --log-format json
```

这段命令启动一个自托管 worker：环境 ID 和 key 用于认证并轮询工作队列；`--workdir` 限制文件工具默认读写范围，`--unrestricted-paths` 再开放额外路径，但注意它不限制 bash；`--max-idle` 避免会话空闲后进程无限等待；`--log-format json` 便于日志采集和结构化分析。如果业务需要 memory stores，应改用 SDK worker。

## 关键点

- 事件类型遵循 `{domain}.{action}` 约定，持久化事件与流式 delta 命名不同，理解这一点能避免在事件流中错误跳过或重复处理动作。
- `user.tool_result` 只在 `self_hosted` 环境使用，因为工具执行发生在你自己的环境中，集成方必须把 `agent_toolset` 结果发回平台。
- 自托管 worker 的 `--workdir` 是文件工具护栏而非沙箱，bash 仍可访问整个主机，所以必须配合容器、权限最小化和网络策略加固。
- CLI worker 不挂载 memory stores，需要长期记忆的会话应使用 SDK worker，否则 store 的挂载路径为空且数据不会同步。
- MCP 服务器应优先支持 streamable HTTP transport，SSE 仅作为兼容回退，新服务不要继续依赖已废弃的 SSE。
- 创建类接口限流 300 rpm、读取类 1200 rpm，且组织级 spend limits 也生效，客户端需实现指数退避重试和并发控制。

## 对比与权衡

- 相比直接调用 Claude API，Managed Agents 提供了会话状态、事件流、工具确认、审批流程和自托管环境等高阶能力，但会牺牲一些底层控制粒度，适合产品化集成而不是一次性脚本。
- 相比 SDK worker，CLI worker 启动更简单、适合快速验证和轻量工具，但不支持挂载 memory stores，且扩展逻辑受限于脚本回调，复杂场景应使用 SDK worker。
- MCP streamable HTTP 相比旧的 SSE transport，在多路复用、双向流和经过代理/负载均衡部署上更好，但 SSE 仍有自动 fallback，方便兼容旧服务。
- 自托管环境相比托管沙箱，数据安全和网络可控性更好，但需要自己负责安全加固、补丁、凭据轮换和资源伸缩，运维成本更高。

## 自测问题

**问: Managed Agents 的持久化事件和 stream-only event deltas 有什么区别？为什么要区分？**

持久化事件遵循 `{domain}.{action}`，可以列出、检索和恢复会话状态；delta 只用于流式实时推送，不会持久化，命名也不同。客户端应把 delta 视为临时增量，最终状态以持久化事件为准，否则容易出现状态错乱或重复消费。

**问: 在 self_hosted 环境里，为什么需要集成方提供 `user.tool_result`？**

因为工具执行在客户自己的 worker 环境中进行，平台不执行 `agent_toolset`，而是由 worker/SDK 执行后把结果作为事件回传。实现时要注意关联正确的 tool_call_id，并按事件类型发送。

**问: 自托管 worker 的 `--workdir` 能作为安全沙箱吗？**

不能。它只限制文件工具读写路径，bash 命令仍可访问整个文件系统。真正的隔离需要容器/VM、只读根文件系统、非 root 用户、网络 egress 限制等。面试常考这一点，说明理解护栏和沙箱的区别。

**问: MCP 服务器要用什么传输，SSE 还能用吗？**

应使用 streamable HTTP transport，它支持多路复用和现代网络部署；如果旧服务只支持 SSE，Managed Agents 会自动 fallback，但这是兼容路径，不建议新服务采用。

**问: 产品里集成 Claude 时，品牌命名有哪些雷区？**

不能用 'Claude Code'、'Claude Cowork' 或模仿其视觉，可以用 'Claude Agent'、菜单里的 'Claude' 或 'Powered by Claude'。关键是保持自己的产品品牌，不冒充 Anthropic 官方产品。

## 适用场景

- 需要在自己 VPC 或本地服务器执行自定义工具、访问私有数据库的合规敏感场景。
- 通过 MCP 服务器连接内部 API、知识库或数据仓库，扩展代理能力的平台集成。
- 构建需要长会话、人工审批、工具确认的企业自动化产品，如 IT 运维、客服代理。
- 上线前做限流评估和品牌合规审查，确保产品命名和调用策略符合 Anthropic 要求。

## 标签

`Claude Managed Agents` `事件流` `自托管 Worker` `MCP` `速率限制`
