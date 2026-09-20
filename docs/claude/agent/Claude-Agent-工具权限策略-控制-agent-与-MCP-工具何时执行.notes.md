# Claude Agent 工具权限策略：控制 agent 与 MCP 工具何时执行

*原文: [https://platform.claude.com/docs/en/managed-agents/permission-policies](https://platform.claude.com/docs/en/managed-agents/permission-policies) · 来源: web · 生成时间: 2026-09-20T07:16:27.829903+00:00*

## 背景

当 agent 具备调用 Bash、外部 MCP 工具等能力时，执行动作可能修改文件、访问敏感接口或产生不可逆副作用，因此需要一种机制在“自动执行”和“人工确认”之间做取舍。权限策略的出现是为了让开发者既能保留工具能力，又能按风险等级控制执行链路，避免完全放权或每次打断。

## 痛点

如果不用权限策略，所有 server 工具要么全部自动执行，可能误操作或执行高风险命令；要么全部要求人工确认，导致频繁打断、降低 agent 自动化效率。尤其 MCP 服务器后续新增工具时，如果没有默认询问策略，新工具可能在未经审核的情况下被自动调用。

## 解决办法

Permission policies 把工具执行分为三级：always_allow 自动放行、always_ask 暂停等待用户确认、auto 由服务端逐次动态评估。它通过工具集 default_config 和单个工具 configs 两层配置实现“全局默认 + 局部覆盖”，类似防火墙默认规则与单条精调规则。每次工具调用事件会携带 evaluated_permission 和 evaluation 对象，客户端可据此审计和分支。核心思想是：策略控制的是“启用后的工具何时能跑”，而不是把工具从 agent 能力中移除。

## 关键代码示例

```json
{
  "tools": {
    "agent_toolset_20260401": {
      "default_config": { "permission_policy": { "type": "always_allow" } },
      "configs": [
        { "name": "bash", "permission_policy": { "type": "always_ask" } }
      ]
    },
    "mcp_toolsets": [
      {
        "mcp_server_name": "github",
        "default_config": { "permission_policy": { "type": "auto" } },
        "configs": [
          { "name": "delete_repo", "permission_policy": { "type": "always_ask" } }
        ]
      }
    ]
  }
}
```

这段配置创建 agent 时的 tools 部分：agent_toolset_20260401 默认 always_allow，但单独把 bash 覆盖为 always_ask，保证命令行操作必须人工确认。github MCP 工具集默认 auto，由服务端动态评估是否放行；同时把 delete_repo 这个高风险工具单独覆盖为 always_ask。这体现了 default_config 与 configs 的层级覆盖：先给整体一个默认策略，再对敏感工具加严。

## 关键流程

1. 在 agent 的 tools 配置中为整个 toolset 设置 default_config.permission_policy，决定该工具集的默认行为。
2. 如需单独调整某个工具，使用 configs 数组按工具名覆盖默认策略，例如让 bash 始终询问。
3. 如果需要服务端动态评估，将 permission_policy 设为 {"type": "auto"}，可作用于整个工具集或单个工具。
4. 在事件流中读取 agent.tool_use / agent.mcp_tool_use 的 evaluated_permission 和 evaluation 字段，判断本次调用是 allow、ask 还是 deny。
5. 对转发不可信终端用户输入到 user.message 的场景，在敏感工具上显式配置 always_ask，避免 auto 被用户意图诱导放行。

## 关键点

- Permission policies 只管控 server 执行的工具，即 agent toolset 和 MCP toolset；自定义工具由应用自身控制，不受这些策略约束，配置时必须区分边界。
- agent toolset 默认 always_allow，而 MCP toolsets 默认 always_ask，因为 MCP 工具集合可能随时新增未审核工具，默认询问更安全，体现安全与体验的权衡。
- auto 策略会结合工具、调用输入和会话上下文逐次评估，因此同一个工具的不同调用可能得到 allow、ask 或 deny 不同结果；但它不能完全替代 always_ask 的确定性安全边界。
- default_config 与 configs 形成层级：default_config 控制整个工具集默认值，configs 按工具名覆盖；这让配置既能全局宽松，又能对高危工具逐个加严。
- 事件中的 evaluated_permission 和 evaluation 用于审计和客户端分支，但 reason_code 是给程序看的，不应直接展示给最终用户；客户端还必须容忍未知的 evaluation.type 和 reason_code。
- 在 auto 模式下，user.message 中的内容会被服务端视为你的意图，可能影响原本会被拒绝的调用；因此转发不可信最终用户输入时，要对相关工具设置 always_ask。

## 对比与权衡

- 相比 always_allow，always_ask 在拦截高风险操作、防止误执行上更好，但在自动化体验和流程效率上不如 always_allow。
- 相比 always_ask，auto 在减少人工审批打断、提升流畅度上更好，但在确定性和安全可控性上不如 always_ask 那么强。
- 相比禁用工具（disable），permission policy 在保留工具能力的同时控制执行方式上更灵活，但在彻底消除风险上不如直接禁用工具。
- 相比完全由应用自定义工具自行实现审批，服务端 permission policies 在统一生效、可审计和减少应用开发成本上更好，但在审批流定制灵活性上不如应用自己控制。

## 自测问题

**问: permission policy 和禁用工具（disable）有什么区别？**

disable 是把工具从会话可用集合中移除，模型无法再调用；permission policy 是工具仍启用，但在执行前按策略放行、询问或拒绝。禁用适用于不需要该能力的场景，策略适用于需要能力但控制风险的场景。事件流上，禁用工具调用会被直接 deny 且没有 evaluation 对象。

**问: 为什么 agent toolset 默认 always_allow，而 MCP toolsets 默认 always_ask？**

agent toolset 是平台预构建的可信工具，默认放行可以降低交互成本；MCP 连接外部服务器，工具集可能动态变化且信任度低，默认询问避免新增工具未经审核就自动执行。这体现了对不同工具来源信任级别的差异。

**问: auto 模式的服务端评估依据是什么？它能替代 always_ask 吗？**

auto 会评估工具名、调用输入、会话上下文，以及 user.message 中表达的意图；它不会把工具结果、网页内容、MCP 响应或子线程消息当作指令，只评估这些内容。不能完全替代 always_ask，因为 auto 有不确定性：某些高风险调用会直接 deny，不确定时仍会 ask；对真正不可信的输入，需要显式 always_ask。

**问: 客户端收到 evaluated_permission: deny 但 evaluation 字段缺失，可能是什么情况？**

有两种情况：一是工具在会话中未启用，服务端直接 deny 而无需评估策略；二是旧事件在 evaluation 字段引入前生成，需要按旧规则兼容。客户端应容忍 evaluation 缺失和未知 type、reason_code。

**问: 在 auto 模式下转发终端用户输入有什么风险？如何缓解？**

服务端会把 user.message 里的内容视为“你的意图”，如果这个内容来自不可信终端用户，用户可能诱导 auto 放行本应询问的工具。缓解方式是：对不受终端用户控制的高危工具设置 always_ask，或者不要在 user.message 中原样转发不可信用户内容，做好意图边界隔离。

## 适用场景

- 使用受信任的 GitHub MCP 服务器时，将默认策略设为 always_allow 或 auto，减少每次调用都需要审批的摩擦。
- 对 Bash 这类高风险工具，即使 agent toolset 默认允许，也单独覆盖为 always_ask，保证命令执行前人工确认。
- 在工具较多且风险差异大时，使用 auto 让服务端动态评估，平衡自动化效率和安全性。
- 需要审计每个工具调用的权限判定结果时，读取事件流中的 evaluated_permission 和 evaluation 字段记录到审计系统。

## 标签

`权限策略` `Claude Agent` `MCP` `工具安全` `auto审批`
