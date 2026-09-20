# Claude Web Search 工具：实时检索、动态过滤与响应控制

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool) · 来源: web · 生成时间: 2026-09-20T03:47:37.227829+00:00*

## 背景

大模型存在知识截止时间，无法可靠回答训练后发生的新闻、价格、机构变动等实时问题。为了在推理过程中让模型自主获取外部实时信息并给出可溯源答案，Anthropic 在 Claude API 中引入了 web search 工具。后来为了控制搜索带来的上下文膨胀，又逐步加入动态过滤和响应包含控制。

## 痛点

没有该工具时，Claude 对知识截止后的信息只能拒绝回答或可能产生幻觉。基础版 web search 会把所有搜索结果直接加载到上下文窗口，无关内容会大量消耗 token，并淹没关键信息。此外，如果缺少域名和地区控制，搜索结果可能来自不可信来源，或者对本地用户不够相关。

## 解决办法

web search 作为工具注入 API 请求，Claude 根据 prompt 是否依赖实时、变化或训练数据外信息来决定是否搜索，API 执行检索并把带引用的结果返回给模型。20260209 及以后版本引入动态过滤：搜索运行在自动提供的 code execution 环境中，模型先写代码过滤结果，只让相关内容进入上下文，相当于给模型配了一个先筛选资料再提交的助手。通过 max_uses 限制搜索次数，通过 allowed_domains 或 blocked_domains 控制来源，通过 user_location 做本地化，response_inclusion 则控制是否在最终响应中回显原始搜索结果块。

## 关键代码示例

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "tools": [
    {
      "type": "web_search_20260318",
      "name": "web_search",
      "max_uses": 5,
      "allowed_callers": ["code_execution_20260120"],
      "response_inclusion": "excluded",
      "allowed_domains": ["example.com"],
      "user_location": {
        "type": "approximate",
        "city": "San Francisco",
        "region": "CA",
        "country": "US",
        "timezone": "America/Los_Angeles"
      }
    }
  ],
  "messages": [
    {"role": "user", "content": "What did Example Corp announce today?"}
  ]
}
```

这段工具定义选择了 web_search_20260318 版本，从而启用动态过滤和 response_inclusion。allowed_callers 指向 code_execution 环境，表示搜索结果先经过代码过滤，不会把所有原始结果直接塞进上下文；allowed_domains 只允许 example.com 来源，user_location 让结果偏向旧金山地区。response_inclusion 设为 excluded 后，已完成代码执行内部的原始搜索块不会写回最终响应，适合不需要回显中间搜索内容的 agent 工作流。

## 关键流程

1. 选择 web_search 版本并加入 tools 数组，例如 web_search_20260318 用于动态过滤和响应包含控制。
2. 根据查询复杂度设置 max_uses，简单事实查询通常 1–3 次，对比或多实体研究可能需要 10 次以上。
3. 用 allowed_domains 或 blocked_domains 限制来源，注意二者只能选其一，不能同时配置。
4. 需要本地化结果时，通过 user_location 提供 city、region、country 或 timezone。
5. 通过 allowed_callers 决定直接调用还是走 code execution 动态过滤，不支持程序化工具调用的模型需显式设为 direct。
6. 在 system prompt 中引导搜索倾向，必要时在响应中读取带引用的搜索结果。

## 关键点

- Claude 会根据 prompt 是否依赖当前、变化或训练数据外信息来决定搜索，因此不是每次请求都必须触发搜索；这能减少不必要的延迟和成本。
- 动态过滤不是简单截断结果，而是在 code execution 中运行模型生成的过滤代码，只把相关内容送入上下文，从信噪比和 token 两方面优化。
- web_search_20260209 及以上版本默认 allowed_callers 为 code execution；如果模型不支持程序化工具调用，必须显式设置为 direct，否则 API 会返回 400 错误。
- allowed_domains 和 blocked_domains 互斥，不能同时提供；条目是裸域名或带路径的域名，不包含 scheme，这避免了规则冲突并简化匹配。
- max_uses 是搜索次数的硬限制，超过后会返回 max_uses_exceeded 错误，适合防止搜索失控或控制单次请求成本。
- response_inclusion 的 excluded 选项面向 agentic workflow，会丢弃嵌套的 server_tool_use 和 result 块，从而减少最终响应的输出 token。

## 对比与权衡

- 相比基础版 web_search_20250305，web_search_20260209/20260318 的动态过滤在 token 效率和上下文信噪比上更好，但依赖模型支持程序化工具调用。
- 相比直接使用 web fetch 抓取指定 URL，web search 更适合探索式、不确定来源的实时查询；但在精确获取已知页面内容上，web fetch 更可控。
- 相比将外部资料离线导入向量库做 RAG，web search 更新近实时且无需维护索引，但每次请求存在网络延迟和不确定性，私有知识沉淀与权限控制不如自建 RAG 灵活。
- 相比在 system prompt 中要求模型“自己上网”，该工具由 API 托管执行检索并返回结构化引用，可靠性和可审计性更高，但定制抓取流程不如自定义检索服务灵活。

## 自测问题

**问: Claude 如何决定是否调用 web search？**

模型根据 prompt 是否依赖实时、变化或训练数据外信息判断；当前价格、新闻、机构变动等会触发搜索，稳定知识、创意写作或分析已提供内容则直接回答。系统提示词可以引导搜索倾向，max_uses 提供硬限制。

**问: dynamic filtering 和普通 web search 的关键区别是什么？**

普通 web search 把所有搜索结果加载进上下文，容易混入大量无关内容；dynamic filtering 在 code execution 环境中运行模型生成的过滤代码，只保留相关结果再进入上下文。API 会自动提供所需代码执行环境，不需要手动添加工具，也没有额外调用费用。

**问: 如果模型不支持 programmatic tool calling，配置动态过滤会怎样？**

web_search_20260209 及以上默认 allowed_callers 是 code_execution_20260120，不支持程序化调用的模型会无法使用；此时必须把 allowed_callers 设为 ["direct"]，否则 API 会返回 400 错误并提示修复。

**问: allowed_domains 和 blocked_domains 能同时使用吗？**

不能同时使用，否则 API 会返回 400。它们互斥是为了避免规则冲突；条目使用裸域名或带路径的域名，不包含 scheme。企业场景通常用 allowed_domains 做白名单，保证来源可信。

**问: response_inclusion: excluded 解决什么问题？**

在 agent 工作流中，搜索结果是 code execution 内部消费的中间产物，最终用户往往只需要最终答案。excluded 会完全丢弃嵌套的 server_tool_use 和 result 块，降低输出 token 成本和响应体积。

## 适用场景

- 金融、体育或新闻问答机器人：获取最新价格、比分、公告，并给出可点击的引用来源。
- 多实体对比研究 agent：同时对多个公司、产品或人物的最新信息进行搜索、过滤和汇总。
- 企业可信来源检索：通过 allowed_domains 把搜索限制在官方或内部站点，避免不可信网页。
- 本地化实时信息：按用户国家、地区或时区返回更相关的当地新闻、法规或价格。

## 标签

`Claude` `Web Search Tool` `Dynamic Filtering` `Tool Use` `Agentic Workflows`
