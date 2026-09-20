# Claude Web Fetch 工具：按需抓取网页/PDF并动态过滤上下文

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool) · 来源: web · 生成时间: 2026-09-20T03:50:16.376774+00:00*

## 背景

大模型存在知识截止问题，无法直接获取实时网页内容；传统做法是用户手动粘贴或开发者自建抓取管道，工程成本高且容易引入安全风险。Web fetch 作为 Anthropic API 原生的 server tool，把抓取、解析和上下文注入放到平台侧完成，让模型在对话中按需获取指定网页或 PDF。后续版本进一步引入动态过滤，解决长文档带来的 token 爆炸问题。

## 痛点

没有 web fetch 时，模型对具体 URL 内容只能靠用户手动粘贴，长文档会迅速占满上下文并混入大量无关信息。开发者自研抓取方案需要分别处理 HTML/PDF 解析、缓存、反爬、SSRF 和域名白名单，维护成本很高。模型本身也无法主动获取时效性强的页面，回答容易过时。

## 解决办法

核心机制是 server tool：API 在请求过程中完成抓取，并把结果直接插入对话，客户端无需运行或返回 tool_result。工具定义使用版本化 type（如 web_fetch_20260318）声明能力；当提示中存在具体 URL 或可定位资源时，模型决定抓取。支持动态过滤的版本会先让模型写代码对抓取内容做筛选，只把相关信息送入上下文，而不是整页加载。配套参数从不同维度控制行为：max_uses 限制次数、max_content_tokens 限制文本 token、use_cache 控制是否允许缓存、response_inclusion 控制是否把嵌套结果返回客户端。可以类比为给模型配了一个可编程的取书器：先取回材料，再由模型写代码撕下相关页，最后只把那几页塞进上下文。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.create(
    model='claude-sonnet-5', max_tokens=1024,
    tools=[{
        'type': 'web_fetch_20260318', 'name': 'web_fetch',
        'max_uses': 3, 'max_content_tokens': 2000,
        'use_cache': False, 'response_inclusion': 'excluded',
    }],
    messages=[{'role': 'user', 'content': 'Summarize this article: https://example.com/long-report'}]
)
print(resp.content)
```

这段代码向 Anthropic Messages API 传入一个 web_fetch 工具定义，工具类型带版本号 `web_fetch_20260318`，因此具备动态过滤、缓存绕过和响应排除能力。`max_uses` 和 `max_content_tokens` 分别限制抓取次数与进入上下文的文本 token 数；`use_cache=False` 强制获取最新内容。客户端只发请求，抓取由服务端完成，最终模型基于抓取内容回答。

## 关键流程

1. 在 API 请求的 `tools` 数组中声明 web fetch 工具，`type` 使用支持所需特性的版本号（如 `web_fetch_20260318`）。
2. 在用户消息中给出具体 URL，或同时启用 web search 让模型先搜索定位资源后再抓取。
3. API 在服务端抓取网页文本；若为 PDF，则以 base64 返回并按附件方式处理。
4. 根据需求配置 `max_uses`、`max_content_tokens`、`use_cache`、`response_inclusion` 以及域名过滤参数。
5. 若使用动态过滤版本，模型会对抓取内容执行过滤代码，仅将相关部分加载到上下文后生成回答。

## 关键点

- Web fetch 是 server tool，抓取在平台侧完成并直接注入对话，客户端无需自行抓取或返回 `tool_result`，这与普通 client tool 的集成模型有本质区别。
- 工具版本号决定能力边界：`20250910` 是基础抓取，`20260209` 起支持动态过滤，`20260309` 增加缓存绕过，`20260318` 增加响应排除；选型时应显式选择满足需求的最新版本。
- 动态过滤让模型在内容进入上下文前用代码提取相关信息，比 `max_content_tokens` 的硬截断更语义化，能显著降低长文档场景的 token 消耗。
- `max_content_tokens` 只对文本内容生效，PDF 等二进制内容不适用；PDF 以 base64 形式返回并作为附件处理，大 PDF 仍可能消耗大量 token。
- `use_cache=False` 只应在用户明确要求新鲜内容或抓取高时效来源时使用，因为它会增加延迟；`response_inclusion='excluded'` 能减少 agent 工作流的输出 token。
- 生产落地必须关注部署限制与安全边界：该工具当前在 Amazon Bedrock 和 Google Cloud 不可用，Azure 托管仅支持基础版；应通过 `allowed_domains`/`blocked_domains` 做域名过滤防 SSRF。

## 对比与权衡

- 相比 web search 工具，web fetch 更擅长精读用户指定的 URL 或 PDF，而不是从开放互联网检索并综合多个来源；开放性问题用搜索，已知页面用抓取。
- 相比自建抓取管道（requests/Playwright + BeautifulSoup/PDF parser 等），web fetch 省去了解析、缓存、反爬、域名白名单和 SSRF 防护等工程负担，但灵活性受平台版本限制，不适合需要登录态或复杂交互的页面。
- 相比仅用 `max_content_tokens` 做粗粒度截断，动态过滤在内容进入上下文前由模型写代码提取相关片段，虽然多了一次代码执行开销，但更可能保留关键语义并减少无关内容干扰。
- 相比 client tool 方式实现自主抓取，server tool 由平台托管执行，客户端代码更简单、可靠性更高；但同一轮中混合 client tool 和 server tool 时会有先返回 `stop_reason: “tool_use”` 的时序差异，需要客户端正确处理。

## 自测问题

**问: web fetch 是 server tool 还是 client tool？和普通 tool use 有什么不同？**

它是 server tool，由 Anthropic API 在请求期间抓取并注入结果，客户端不用执行或返回 tool_result。若同一轮同时调用 web fetch 和 client tool，API 会先以 `stop_reason: “tool_use”` 返回，要求客户端先处理 client tool 再继续。

**问: 动态过滤是如何工作的？和 `max_content_tokens` 有何区别？**

支持动态过滤的版本允许模型生成并执行代码，在内容进入上下文前对抓取结果做筛选、提取或结构化处理；`max_content_tokens` 是无差别的长度截断。前者更语义化，但依赖模型和代码执行能力，后者简单可靠但可能切掉关键信息。

**问: 什么时候应该把 `use_cache` 设为 false？**

当用户明确要最新数据，或数据源变化很快（如价格、新闻、状态页）时才设为 false。默认允许缓存可以降低延迟、减少重复抓取；绕过缓存会增加延迟，所以不应作为默认配置。

**问: PDF 内容在 web fetch 中怎么处理？为什么要注意 token？**

API 会把 PDF 作为 base64 编码数据返回，并像直接附加的 PDF 一样处理，而不是纯文本；因此 `max_content_tokens` 对 PDF 不生效，大 PDF 仍会大量占用上下文。处理大 PDF 时应优先考虑动态过滤或先缩小页数。

**问: 如何限制 Web Fetch 只访问安全域名，避免 SSRF？**

使用 server tools 的域名过滤机制，配置 `allowed_domains` 白名单和 `blocked_domains` 黑名单；在 Managed Agents 中每个域名必须是纯主机名，不能带路径。域名过滤是从入口做安全控制，比仅靠提示词约束可靠。

## 适用场景

- 用户给出具体文章、README、论文或定价页 URL，要求总结、提取条款或对比更新。
- 处理大型 PDF/网页时只需要其中某个章节或表格，避免整篇进入上下文。
- Agent 工作流需要实时读取文档、政策或价格页面，并用 `response_inclusion='excluded'` 控制输出成本。
- 先通过 web search 发现相关页面，再用 web fetch 精读内容，形成“搜索定位 + 抓取精读”的组合。

## 标签

`Claude API` `Web Fetch` `Server Tool` `Token 优化` `Agent 工具`
