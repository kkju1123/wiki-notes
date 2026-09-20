# Claude 工具定义提示缓存：cache_control、defer_loading 与缓存失效

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching) · 来源: web · 生成时间: 2026-09-20T04:13:18.411943+00:00*

## 背景

Claude 的 prompt caching 允许缓存请求前缀，减少多轮对话中重复处理相同内容的 tokens 和延迟。工具定义（尤其是 MCP、computer use、browser use 等套件）通常较长且跨轮不变，是缓存的重点对象。但工具动态加载、参数调整会影响缓存层级，因此需要明确断点位置和失效规则。

## 痛点

如果每轮都重新发送完整工具 schema，会产生大量重复输入 token，成本高、首 token 延迟增加。若在不合适的位置放置 cache_control 或在动态发现工具时不加控制，可能破坏本可复用的前缀缓存。不理解 tools → system → messages 的失效层级，调试缓存命中率下降会很困难。

## 解决办法

把 cache_control 放在 tools 数组最后一个工具上，从第一个工具到该断点的整段工具定义都会被缓存；对于 MCP/computer/browser 套件，放在 toolset 条目本身。defer_loading 使延迟工具不进入 system 前缀，而是被发现后以 tool_reference 追加到对话历史，因此前缀缓存不受影响。缓存按 tools → system → messages 前缀层级组织，工具定义变更会清空全缓存，而 tool_choice、图片等只影响 messages 层。当请求中已有至少一个 cache_control 时，服务器工具结果会自动获得 5 分钟 TTL 的缓存断点，支持 agentic loop 后续迭代直接读缓存。

## 关键代码示例

```json
{"model":"claude-3-5-sonnet-20241022","max_tokens":1024,"tools":[{"name":"get_weather","description":"Get weather","input_schema":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}},{"name":"get_time","description":"Get time","input_schema":{"type":"object","properties":{"tz":{"type":"string"}}},"cache_control":{"type":"ephemeral"}}],"messages":[{"role":"user","content":"巴黎现在几点？天气如何？"}]}
```

这段请求把 cache_control 放在 tools 数组的最后一个工具 get_time 上，表示缓存从 tools 开头到 get_time 结束的完整工具定义前缀。这样后续轮次复用同一 tools 数组时，工具 schema 部分可直接命中缓存，无需重新处理。若把断点放在第一个工具上，只会缓存第一个工具定义，达不到减少重复 tool schema 成本的目的。

## 关键流程

1. 将 cache_control 放在 tools 数组最后一个工具定义上，缓存整段工具前缀。
2. 对 MCP/computer/browser 等 toolset，把 cache_control 放在 toolset 条目自身，而不是成员 configs 中。
3. 启用 defer_loading/tool search 动态加载额外工具，使其以 tool_reference 追加到历史，避免污染前缀缓存。
4. 修改工具定义、tool_choice、thinking 等参数前，先按 tools→system→messages 层级评估会失效哪些缓存。
5. 确保请求中至少有一个 cache_control 标记，才能获得服务器工具结果的自动缓存断点。

## 关键点

- cache_control 是前缀断点而非独立缓存项：放在最后一个工具上才能缓存从第一个工具到该断点的完整 tools 前缀，这是控制工具缓存成本的关键。
- defer_loading 工具不进入 system 前缀，发现后以 tool_reference 追加，因此动态工具发现不会破坏既有 tools/system 前缀缓存；strict mode 的 grammar 构建仍基于完整工具集，不会因动态加载受到影响。
- 缓存失效范围遵循 tools→system→messages 层级：修改工具定义清空全部缓存；修改 tool_choice、图片或 thinking 参数通常只影响 messages 层，thinking 在部分模型上也会影响更前层级。
- 服务器工具结果只有在请求已启用 prompt caching 时才会自动写入默认 5 分钟 TTL 的缓存断点，并出现在 usage 的 ephemeral_5m_input_tokens 中。
- mcp_toolset、computer use、browser use 的 cache_control 必须放在 toolset 条目本身，因为成员展开顺序不可控，API 会将其应用到最终展开的工具。
- 请求最多四个 breakpoints；同一批次中多个 marker 仍计为多个 breakpoints，因此每轮应只放一个用于批次缓存断点。

## 对比与权衡

- 相比全量加载所有工具，defer_loading + tool search 能显著缩小初始 system 前缀，保持 tools/system 缓存稳定；但会增加发现工具的额外轮次，可能带来少量延迟。
- 相比在 MCP 成员工具的 configs 里放 cache_control，放在 toolset 条目本身可行并由 API 映射到展开后的最后工具；前者不被接受，因为 toolset 成员作为整体加载。
- 相比自定义 cache_control（可设 1 小时 TTL），服务器工具结果自动缓存固定为 5 分钟 TTL，且只在请求已有 cache_control 时启用；它更适合 agentic loop 内的短期复用。

## 自测问题

**问: 为什么 cache_control 要放在 tools 数组的最后一个工具，而不是第一个？**

提示缓存以断点标记需要缓存的前缀终点，放在最后一个工具才能覆盖从第一个工具到该标记的完整 tools 前缀；放在第一个只会缓存第一个工具定义，后续工具的 schema 仍会被重复处理。

**问: defer_loading 为什么能保持缓存？**

延迟工具不会预先写入 system 前缀，模型通过 tool search 发现后才把定义作为 tool_reference 追加到 messages 历史。前缀 tools/system 不变，因此之前的缓存仍可命中；只是 messages 缓存会随新增引用增长。

**问: 修改 tool_choice 和修改工具定义对缓存的影响有什么不同？**

缓存按 tools→system→messages 层级组织。工具定义属于 tools 前缀，修改会让 tools、system、messages 全部失效；而 tool_choice 不改变前缀定义，只影响 messages 层，因此 tools/system 缓存可保留。

**问: 服务器工具结果自动缓存有哪些前提和限制？**

请求必须至少有一个 cache_control 标记；API 会在服务器工具结果后自动加断点，供同一请求的后续迭代读取。该断点固定 5 分钟 TTL，独立于自定义 TTL，usage 中记为 ephemeral_5m_input_tokens。

**问: MCP toolset 的缓存断点为什么不能放在成员 configs 里？**

因为 MCP 工具成员会展开为一个整体定义，且工具顺序由服务端控制，放在成员 configs 不被接受。正确做法是把 cache_control 放在 mcp_toolset 条目本身，由 API 映射到展开后的最后一个工具。

## 适用场景

- 使用大量工具（尤其是 MCP、computer use、browser use 套件）的多轮 Agent 应用，需要降低每轮重复发送工具 schema 的成本。
- 需要按需动态发现工具的大工具库场景，结合 defer_loading/tool search 保持前缀缓存稳定。
- 在同一请求中执行多个 server tool 迭代的 agentic loop，如 web search、code execution 工作流。
- 需要精准控制 prompt caching 命中与失效范围，以优化 token 成本和首 token 延迟的 API 客户端开发。

## 标签

`prompt caching` `tool use` `cache_control` `defer_loading` `Claude API`
