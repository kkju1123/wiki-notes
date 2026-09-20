# Server tools：服务端工具调用、暂停与混合轮次机制

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) · 来源: web · 生成时间: 2026-09-20T03:46:06.062093+00:00*

## 背景

在 Agent 工具调用中，网页搜索、抓取等外部能力如果全部由开发者自行实现，会重复集成外部服务并增加多轮状态管理成本。Anthropic 把这类工具下沉为 Server tools，由 API 在服务端执行，Claude 可以在服务端 agentic loop 中连续完成多步工具调用。这样既简化了接入，也让模型能自主完成检索、过滤和归纳。

## 痛点

如果混淆 server_tool_use 和普通 tool_use，容易误给服务端工具回传 tool_result 导致请求错误；在混合调用中若分不清 pause_turn 与 tool_use，会选错续接方式，造成空转或状态丢失；续接时遗漏 tools 数组还会让 pending server tool 验证失败。

## 解决办法

Server tools 通过 server_tool_use 块暴露调用，id 前缀 srvtoolu_ 区分于 client tool_use；API 内部执行并把结果块放在同一 assistant turn，按 tool_use_id 配对。长任务服务端循环暂停时返回 pause_turn，需要原样重发 assistant 消息并保持 tools。若同一个并行调用组里混有 client tool，响应会以 tool_use 停止，但 server_tool_use 没有 result；此时只需执行并回传 client tool_result，API 会在下一次请求开始时执行 pending server tool。可以把 server tool 理解为平台托管的异步任务，你只看到回执；client tool 才是必须由你处理的待办项。

## 关键代码示例

```python
def handle_turn(client, model, messages, tools, resp):
    messages.append({'role': 'assistant', 'content': resp.content})

    if resp.stop_reason == 'pause_turn':
        return client.messages.create(model=model, messages=messages, tools=tools, max_tokens=1024)

    if resp.stop_reason == 'tool_use':
        results = [
            {'type': 'tool_result', 'tool_use_id': b.id, 'content': run_client_tool(b)}
            for b in resp.content
            if b.type == 'tool_use'
        ]
        messages.append({'role': 'user', 'content': results})
        return client.messages.create(model=model, messages=messages, tools=tools, max_tokens=1024)

    return resp
```

这段代码处理一轮可能混合 server 与 client 工具的响应。先追加 assistant 消息保留上下文；如果 stop_reason 是 pause_turn，说明服务端循环暂停且没有等待你的 client tool，因此原样重发 assistant 内容并保持 tools 数组继续。如果 stop_reason 是 tool_use，则只对 type 为 tool_use 的 client 块执行工具并构造 tool_result，忽略 server_tool_use；然后以 user 消息回传结果并保持 tools 继续请求，API 会在下一轮开始时执行 pending server tool。

## 关键流程

1. 检查每个响应最外层的 stop_reason，区分 pause_turn、tool_use 和其他终止原因。
2. 遇到 pause_turn 时，将响应原样作为 assistant 消息追加到 messages，保持 tools 数组不变，继续请求。
3. 遇到 tool_use 时，遍历 content，只对 type 为 tool_use 的 client 块执行工具并生成 tool_result，忽略没有 result 的 server_tool_use 块。
4. 将 assistant 响应和包含 tool_result 的 user 消息按顺序追加到 messages，保持 tools 数组，继续请求。
5. 重复检查 stop_reason，直到不再是 pause_turn 或 tool_use，并为续接次数设置上限，避免无限循环。

## 关键点

- server_tool_use 的 id 前缀为 srvtoolu_，用于区分服务端工具调用和普通 client tool_use，因为二者处理路径不同：前者无需回传结果，后者必须回传 tool_result。
- pause_turn 表示服务端 agentic loop 暂时挂起，续接时必须原样重发 assistant content 并保留 tools 数组，否则会丢失服务端循环状态或触发验证错误。
- 混合 server/client 工具调用时 stop_reason 一定是 tool_use 而不是 pause_turn，判断依据是响应中是否存在没有匹配 result 的 server_tool_use 块。
- 续接混合调用时只能回传 client tool_use 对应的 tool_result，不能给 server_tool_use 构造结果；API 会在下一次请求开始时执行 pending server tool。
- server_tool_use 与 result 通过 tool_use_id 配对而不是靠位置；跨响应时 server_tool_use 不会重复，因此必须按顺序维护完整 messages 数组。
- 带动态过滤的 _20260209 及以后 web 工具默认走 code execution caller，不符合 ZDR；要启用 ZDR 必须设置 allowed_callers: ['direct'] 绕过内部代码执行。

## 对比与权衡

- 相比客户端工具 client tools，server tools 将执行托管在 API 侧，减少开发者实现外部服务集成和多轮状态管理的负担，但灵活性和可控性不如自己执行，例如无法自定义执行环境或访问内部网络。
- 相比 pause_turn，混合 client tool 的 tool_use 停止在续接方式上更直接：前者原样重发 assistant content，后者回传 client tool_result；两者都会在下一请求执行 pending server tool，但 pause_turn 不会留下 client tool_use 等待你处理。
- 相比 _20260209 动态过滤默认的 code execution caller，设置 allowed_callers 为 ['direct'] 在 ZDR 合规上更好，但会失去动态过滤依赖的内部代码执行能力。

## 自测问题

**问: server_tool_use 和普通 tool_use 有什么区别？**

从 id 前缀、执行方、是否需要回传 tool_result、结果是否在同一响应中配对四个方面回答。server_tool_use 的 id 以 srvtoolu_ 开头，由 API 执行，结果块跟在同轮 assistant 内容中；普通 tool_use 需要客户端执行并在下一请求回传 tool_result。混合场景中可通过响应里是否存在无 result 的 server_tool_use 来识别。

**问: pause_turn 和 stop_reason: tool_use 怎么区分？**

pause_turn 是服务端循环暂停，不会留下 client tool_use 等待你处理；tool_use 是停下来等你执行 client 工具。续接方式不同：前者原样重发 assistant content，后者回传 tool_result。两者的共同点是 API 都会在下一次请求开始时执行 pending server tool。

**问: 为什么续接请求必须保持相同的 tools 数组？**

pending server tool 尚未执行，API 需要在续接的第一个请求中还原并执行该工具；如果缺少 tool 定义，会返回 400，错误信息结尾类似 but no web_fetch tool was provided。这是服务端工具状态跨请求保持的要求。

**问: ZDR 与 allowed_callers 有什么关系？**

动态过滤依赖内部 code execution，代码执行容器可能带来数据合规风险，所以默认不符合 Zero Data Retention。设置 allowed_callers: ['direct'] 可以绕过内部代码执行步骤，使工具以 ZDR 合规方式运行，但会牺牲动态过滤等依赖代码执行的能力。

**问: 域过滤怎么配置？**

在 tool object 上设置 allowed_domains 和 blocked_domains。域名不要带 http/https scheme；子域名自动包含，指定具体子域名则只匹配该子域名；web search 支持子路径匹配，web fetch 仅按域名匹配，不按路径过滤。

## 适用场景

- 使用 Claude 内置 web search 或 web fetch 构建研究助手或 RAG 流程，不想自行集成搜索 API 和网页抓取服务。
- Agent 需要同时使用服务端检索工具和自定义客户端工具，例如混合 web_search 与数据库查询、命令执行。
- 在合规要求 Zero Data Retention 的场景下配置 server web tools，并限制 allowed_callers 以绕过内部代码执行。
- 需要处理长时服务端工具调用或自动多步搜索，并且客户端要正确续接 pause_turn 和混合工具轮次。

## 标签

`Claude API` `server_tool_use` `pause_turn` `tool_use` `ZDR`
