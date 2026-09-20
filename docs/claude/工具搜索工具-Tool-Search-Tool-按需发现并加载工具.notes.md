# 工具搜索工具（Tool Search Tool）：按需发现并加载工具

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) · 来源: web · 生成时间: 2026-09-20T03:55:11.070315+00:00*

## 背景

在大规模 Agent 工具集成场景中，如同时接入 GitHub、Slack、Sentry、Grafana 等，每个工具的名称、描述、参数 schema 都会进入模型上下文。随着工具数量增长，上下文开销和选择难度都非线性上升。工具搜索工具由此出现，把“全量加载”改成“按需检索加载”。Claude 部分新模型原生支持该能力。

## 痛点

没有工具搜索时，一个典型多服务器配置可能仅工具定义就占约 55k tokens，未开始工作先消耗大量上下文。可用工具超过 30–50 个后，模型选择准确率明显下降，容易选错工具或调用错误参数。

## 解决办法

核心思路类似“图书馆目录卡片”：请求中仍上传全部工具定义供服务端检索，但通过 defer_loading:true 标记哪些工具不立即注入上下文。模型开始只看到搜索工具和少数高频工具；需要时调用正则或 BM25 搜索工具，API 在服务端执行检索，返回 tool_reference 块并自动展开为完整工具定义，再交给模型选择调用。默认返回前 5 个匹配项，可通过 limit 调整。这样既减少上下文，又把最终工具选择限制在小候选集内。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    tools=[
        {
            'name': 'tool_search_tool_regex_20251119',
            'description': 'Search available tools using regex',
            'input_schema': {
                'type': 'object',
                'properties': {
                    'pattern': {'type': 'string'},
                    'limit': {'type': 'integer', 'default': 5}
                },
                'required': ['pattern']
            }
        },
        {
            'name': 'get_weather',
            'description': 'Get current weather',
            'input_schema': {
                'type': 'object',
                'properties': {'city': {'type': 'string'}},
                'required': ['city']
            },
            'defer_loading': True
        }
    ],
    messages=[{'role': 'user', 'content': 'What is the weather in SF?'}]
)

```

这段代码展示了最小配置：工具搜索工具自身没有 defer_loading，因此一开始就进入上下文；业务工具 get_weather 设置 defer_loading:true，服务端保留完整定义但不注入上下文。模型需要天气工具时，会先调用搜索工具，API 返回并自动展开 get_weather 的完整定义，然后模型才发起真正的工具调用。

## 关键流程

1. 在 tools 数组中包含一个工具搜索工具（regex 或 BM25）作为非 deferred 工具。
2. 为其它业务工具设置 defer_loading:true，至少保留 3–5 个高频工具非 deferred。
3. 初始上下文只包含搜索工具和非 deferred 工具。
4. Claude 需要额外工具时，调用搜索工具并传入 pattern 或 query，可带 limit。
5. API 在服务端执行搜索，返回 tool_reference 块（默认最多 5 个匹配工具）。
6. API 自动将 tool_reference 展开为完整工具定义并交给 Claude。
7. Claude 从发现的工具中选择并调用，客户端正常执行并返回 tool_result。

## 关键点

- defer_loading 控制的是工具定义是否进入上下文，而不是是否上传：请求中仍必须携带所有工具的完整定义，否则服务端无法检索和展开。
- 工具搜索工具自身必须保持非 deferred，否则模型在初始上下文里无法发起搜索。
- 保留 3–5 个高频工具非 deferred，可避免每个简单请求都先搜索，平衡延迟与上下文占用。
- API 自动展开 tool_reference，客户端不需要也不应该手动展开或为 server_tool_use 返回 tool_result。
- regex 变体适合按命名规范或精确模式检索，BM25 变体适合自然语言查询相关工具。

## 对比与权衡

- 相比一次性加载全部工具定义，工具搜索通常减少 85% 以上的定义 token 占用，但增加了搜索步骤，可能带来额外的服务端搜索延迟。
- 相比 regex 搜索，BM25 对自然语言查询更友好，按相关性召回；但在需要精确匹配工具名前缀或版本号等场景，regex 更可控。
- 相比客户端自定义检索实现，服务端工具搜索自动处理展开和 prompt caching，集成简单；但自定义方案可以接入自己的向量索引或权限过滤，灵活度更高。

## 自测问题

**问: 为什么请求里还是要传全部工具定义，defer_loading 还能降低上下文占用？**

defer_loading 只影响模型上下文注入，不影响 API 服务端接收完整定义。服务端用完整定义做搜索和展开，但模型看到的系统前缀不包含 deferred 工具，因此输入 token 和注意力负担都下降，prompt cache 也不会被大量工具定义撑大。

**问: tool search 的工作流程是怎样的？**

模型初始只看到搜索工具和非 deferred 工具；需要调用额外工具时，调用搜索工具发起服务端检索；API 返回匹配的 tool_reference 并自动展开成完整定义；模型再从中选择工具调用；用户只需正常处理最终的 tool_use。

**问: server_tool_use 和普通 tool_use 处理上有什么不同？**

server_tool_use 是模型调用服务端工具搜索时产生的，搜索由 Anthropic 服务器执行，客户端永远不要为它返回 tool_result；后续响应里会出现 tool_references 和最终 tool_use，继续对话时要把 server_tool_use 和 tool_search_tool_result 原样传回。

**问: 工具数量很多时，除了上下文问题，为什么工具选择准确率也会下降？**

模型在候选工具集很大时，相似工具名和描述会产生干扰，注意力分散导致选错工具或参数。搜索相当于先做一层召回，把候选从几百上千缩到默认 5 个左右，再做精确选择，因此准确率能保持。

**问: regex 和 BM25 两种搜索变体怎么选？**

如果工具命名规范、版本号有明确模式，或者需要精确控制匹配，选 regex；如果工具描述偏自然语言、用户 query 多变，选 BM25，它按关键词相关性召回更自然。

## 适用场景

- 需要同时接入数十到数千个工具的大型 Agent，如企业内部自动化平台或多服务运维助手。
- 工具定义 token 占用已经成为成本、延迟或上下文窗口瓶颈的场景。
- 希望保持 prompt cache 命中率，同时动态加载大量工具的长期运行会话。
- 工具选择准确率要求高且工具目录持续增长的应用。

## 标签

`Tool Search` `Claude API` `Agent Tools` `defer_loading` `上下文优化`
