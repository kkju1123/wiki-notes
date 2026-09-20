# Claude Managed Agents 工具配置：内置工具集、权限策略与 Web 域名限制

*原文: [https://platform.claude.com/docs/en/managed-agents/tools](https://platform.claude.com/docs/en/managed-agents/tools) · 来源: web · 生成时间: 2026-09-20T07:11:15.761680+00:00*

## 背景

在 LLM 从聊天模型走向自主执行任务的 Agent 过程中，模型需要在会话里操作文件系统、执行命令、访问网页。Claude Managed Agents 为此提供了一组内置工具，但直接全开会让 Agent 拥有过大权限。于是平台引入工具集配置，让开发者根据任务边界精确控制模型能做什么，在能力与安全之间取得平衡。

## 痛点

如果所有工具默认开启且不可控，Agent 可能误删文件、执行危险命令、访问内部敏感地址或被恶意网页诱导。开发者若不了解 default_config 与 per-tool override 的语义，容易配置出比预期更宽的权限。此外，不限制 Web 域或抓取内容长度，可能造成数据外泄或上下文窗口被大量无关内容占满。

## 解决办法

通过 agent_toolset_20260401 这个工具集对象统一管理内置工具：先以 default_config 设置基线（比如全部关闭），再用 configs 数组为单个工具覆盖 enabled 和 permission_policy，形成白名单或黑名单模式。对 web_search 和 web_fetch 进一步用 allowed_domains 或 blocked_domains 做域名边界控制，用 max_content_tokens 限制抓取内容进入上下文的大小。工具输出超过 100,000 字符时自动落盘到沙箱文件，只给模型预览和路径，防止上下文爆炸。这类似给新员工分配最小权限：默认不许动，按任务开放指定工具，访问外网还要加防火墙规则。

## 关键代码示例

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": false },
      "configs": [
        { "name": "read", "enabled": true },
        { "name": "write", "enabled": true, "permission_policy": "requires_confirmation" },
        { "name": "web_search", "enabled": true, "allowed_domains": ["example.com"] },
        { "name": "web_fetch", "enabled": true, "allowed_domains": ["example.com"], "max_content_tokens": 5000 }
      ]
    }
  ]
}
```

这段 JSON 在 agent 的 tools 数组里放入 agent_toolset_20260401 工具集。default_config.enabled=false 先关闭所有内置工具，per-tool configs 再打开 read、write、web_search、web_fetch。web 工具通过 allowed_domains 限制可达域名，max_content_tokens 防止抓取内容占用过多上下文；write 需要确认，体现最小权限和人工介入。

## 关键流程

1. 在 agent 配置的 tools 数组中加入 agent_toolset_20260401 工具集。
2. 如需白名单模式，设置 default_config.enabled=false。
3. 在 configs 数组中按 name 指定工具，覆盖 enabled 或 permission_policy。
4. 为 web_search 和 web_fetch 设置 allowed_domains 或 blocked_domains（二选一），以及 max_content_tokens 和 user_location。
5. 创建或更新 agent/session 时系统会验证配置，非法域名或冲突列表返回 400 invalid_request_error。

## 关键点

- default_config 是基线，configs 按工具覆盖，二者组合可实现默认全开或默认全关的权限模型，这是控制工具暴露面的核心机制。
- allowed_domains 与 blocked_domains 互斥且均可匹配子域，开发者必须理解这一规则，否则可能在配置阶段就被拒绝或留下意外放行。
- permission_policy 提供无需确认、需要确认、服务端逐个评估三种级别，让高风险工具可以有人工介入，降低不可逆操作风险。
- web_fetch 的 max_content_tokens 直接限制进入上下文的内容量，是控制成本与上下文污染的关键参数。
- 超过 100,000 字符的工具输出会自动落盘到沙箱，模型只拿预览和路径，需要时用 read 读取，避免一次性塞爆上下文窗口。
- 域名规则很严格：不允许 IP、不允许裸顶级域、不允许 localhost 等，目的是强制使用可审计的真实域名，防止策略绕过。

## 对比与权衡

- 相比通过 MCP connector 接入外部工具，内置工具集开箱即用、无需额外服务，但工具种类固定，灵活性不如 MCP 可接入任意 MCP server。
- 相比自定义 user-defined tools，内置工具由平台托管、生命周期简单，但无法执行应用自定义逻辑，只能使用平台提供的能力。
- 相比默认全开 + blocked_domains 的黑名单模式，default_config.enabled=false + allowed_domains 的白名单模式更安全，但需要开发者预先枚举所有需要的能力，维护成本更高。

## 自测问题

**问: allowed_domains 和 blocked_domains 为什么不能同时设置？**

同一入口如果既允许又阻止，会出现优先级歧义，策略判断变复杂且容易绕过；平台强制二选一，让边界要么是默认拒绝+白名单，要么是默认允许+黑名单，保证可预测性。

**问: 工具输出超过 100,000 字符时，Agent 如何获得完整内容？**

平台自动把完整输出写入沙箱文件，模型只收到截断预览和文件路径；模型可以调用 read 工具读取该文件。这样避免海量输出直接进入上下文，既节省 token 又保留完整信息。

**问: default_config 与 configs 的生效优先级是什么？**

default_config 对所有工具生效，configs 中同 name 条目覆盖 default_config 的字段；未出现在 configs 的工具沿用 default_config。理解这一点才能正确配置白名单或黑名单。

**问: 为什么域名列表不接受 IP 地址和 localhost？**

IP 直连可以绕过 DNS 层面的域名策略和审计，且 IP 不易维护；localhost 等本地域可能被用于访问宿主机敏感服务。强制真实域名让策略更可控、可审计。

**问: 如果 web_fetch 请求了不在 allowed_domains 里的 URL，会发生什么？**

运行时该调用会返回错误结果给 Agent，事件中 is_error=true，错误码为 url_not_allowed；web_search 则直接忽略不符合域限制的结果。这样模型能感知边界并调整后续行为。

## 适用场景

- 企业知识库问答 Agent：只允许 web_search 访问公司内部文档域名，并开启其他必要工具，防止信息外泄。
- 代码开发辅助 Agent：开放 read/write/edit/bash 工具处理沙箱文件，但关闭 web 工具避免访问外部网络。
- 网页研究 Agent：开放 web_search 和 web_fetch，并用 max_content_tokens 限制抓取内容，避免大量无关网页占满上下文。
- 需要人工审批的运维 Agent：对 bash、write 设置 requires_confirmation，确保危险操作在执行前得到确认。

## 标签

`Claude Managed Agents` `工具配置` `安全策略` `Web 工具` `权限策略`
