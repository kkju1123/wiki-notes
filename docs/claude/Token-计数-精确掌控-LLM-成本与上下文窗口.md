---
title: Token 计数：精确掌控 LLM 成本与上下文窗口
url: https://platform.claude.com/docs/en/build-with-claude/token-counting
source_type: web
folder: claude
author: null
tags:
- LLM
- Token 计数
- Anthropic Claude
- API 成本
- 上下文管理
summary: 讲解 LLM token 计数原理、Claude API 的 count_tokens 与 usage 用法，帮助管理成本与上下文。
fetched_at: '2026-09-20T03:10:05.014056+00:00'
---

[Claude Platform Docs](https://platform.claude.com/docs/en/home)
*   [Messages](https://platform.claude.com/docs/en/intro)
*   [Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview)
*   [Admin](https://platform.claude.com/docs/en/manage-claude/admin-api)
*   
Resources
    *   [Best practices](https://platform.claude.com/docs/en/about-claude/use-case-guides/overview)
    *   [Models & pricing](https://platform.claude.com/docs/en/models/overview)
    *   [CLI, SDKs, and libraries](https://platform.claude.com/docs/en/cli-sdks-libraries/overview)
    *   [Claude API skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/claude-api-skill)
    *   [Release notes](https://platform.claude.com/docs/en/release-notes/overview)

[API reference](https://platform.claude.com/docs/en/api/overview)

English

[Console](https://platform.claude.com/)[Log in](https://platform.claude.com/login?returnTo=%2Fdocs%2Fen%2Fbuild-with-claude%2Ftoken-counting)



Search Ctrl K

First steps

[Intro to Claude](https://platform.claude.com/docs/en/intro)[Get your API key](https://platform.claude.com/docs/en/get-api-key)[Quickstart](https://platform.claude.com/docs/en/get-started)[Authentication](https://platform.claude.com/docs/en/manage-claude/authentication)

Building with Claude

[Features overview](https://platform.claude.com/docs/en/build-with-claude/overview)[Using the Messages API](https://platform.claude.com/docs/en/build-with-claude/working-with-messages)[Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)[Refusals and fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)[Fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit)

Model capabilities

[Effort](https://platform.claude.com/docs/en/build-with-claude/effort)[Task budgets (beta)](https://platform.claude.com/docs/en/build-with-claude/task-budgets)[Fast mode (research preview)](https://platform.claude.com/docs/en/build-with-claude/fast-mode)[Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)[Citations](https://platform.claude.com/docs/en/build-with-claude/citations)[Streaming Messages](https://platform.claude.com/docs/en/build-with-claude/streaming)[Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)[Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)[Streaming refusals](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)[Multilingual support](https://platform.claude.com/docs/en/build-with-claude/multilingual-support)[Embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings)

[Thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)

Tools

[Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)[How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)[Tutorial: Build a tool-using agent](https://platform.claude.com/docs/en/agents-and-tools/tool-use/build-a-tool-using-agent)[Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)[Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)[Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)[Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)[Strict tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)[Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools)[Web search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool)[Web fetch tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool)[Code execution tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)[Advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)[Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)[Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)[Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool)[Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool)[Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)[Browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool)[Troubleshooting](https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use)

Tool infrastructure

[Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)[Manage tool context](https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context)[Tool combinations](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-combinations)[Tool use with prompt caching](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching)[Programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)[Fine-grained tool streaming](https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming)

Context management

[Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)[Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)[Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)[Mid-conversation system messages and tool changes](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages)[Build an orchestration mode](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-effort-example)[Cache diagnostics (beta)](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics)[Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)

Working with files

[Files API](https://platform.claude.com/docs/en/build-with-claude/files)[PDF support](https://platform.claude.com/docs/en/build-with-claude/pdf-support)

[Images and vision](https://platform.claude.com/docs/en/build-with-claude/vision)

Skills

[Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)[Quickstart](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart)[Best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)[Skills for enterprise](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise)[Skills in the API](https://platform.claude.com/docs/en/build-with-claude/skills-guide)

MCP

[Remote MCP servers](https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers)[MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)

[MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview)

Claude on cloud platforms

[Amazon Bedrock (Opus 4.7 and later)](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock)[Amazon Bedrock (Opus 4.6 and earlier)](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy)[Claude Platform on AWS](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws)[Google Cloud](https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai)[Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry)

[Console](https://platform.claude.com/)

[Messages](https://platform.claude.com/docs/en/intro)Context management

# Token counting

Copy page



Count the tokens in a message before you send it to Claude. Use token counts to manage rate limits and costs, make model routing decisions, and fit prompts to a target length.

Copy page



Token counting

[ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)

Eligible

excludes[Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)

Token counting lets you determine the number of tokens in a message before you send it to Claude. This helps you make informed decisions about your prompts and usage. With token counting, you can:

*   Proactively manage rate limits and costs
*   Make smart model routing decisions
*   Optimize prompts to a specific length

* * *

## How to count message tokens

The [token counting](https://platform.claude.com/docs/en/api/messages-count-tokens) endpoint accepts the same structured list of inputs for creating a message, including support for system prompts, [tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), [images](https://platform.claude.com/docs/en/build-with-claude/vision), and [PDFs](https://platform.claude.com/docs/en/build-with-claude/pdf-support). The response contains the total number of input tokens.

This endpoint returns an `invalid_request_error` for a few inputs that the Messages API accepts: [server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) such as web search, web fetch, code execution, and tool search (every server tool except the [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)), the [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector), and `image` or `document` blocks with a `url` or `file` source. Send images and PDFs as base64 to count them. For requests that use server tools or MCP servers, the Messages API response reports the tokens used in its `usage` object.



The token count is an **estimate**. In some cases, the actual number of input tokens used when creating a message might differ by a small amount.

Token counts may include tokens added automatically by Anthropic for system optimizations. **You are not billed for system-added tokens**. Billing reflects only your content.

### Supported models

All [active models](https://platform.claude.com/docs/en/models/overview) support token counting.



Claude 4.7 and later models and Claude Mythos Preview use a newer tokenizer. The same input text produces approximately 30 percent more tokens than on earlier models. The exact increase depends on the content and workload shape. Recount prompts against the model you plan to use rather than reusing counts measured against earlier models.

### Count tokens in basic messages

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-opus-5",
    system="You are a scientist",
    messages=[{"role": "user", "content": "Hello, Claude"}],
)

print(response.json())
```

Output



`{ "input_tokens": 14 }`

### Count tokens in messages with tools



Token counting supports client tools and the [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool). Requests that include other [server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) return an error. For the advisor tool, the count covers the executor's first sampling call only.

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-opus-5",
    tools=[
        {
            "name": "get_weather",
            "description": "Get the current weather in a given location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    }
                },
                "required": ["location"],
            },
        }
    ],
    messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
)

print(response.json())
```

Output



`{ "input_tokens": 403 }`

### Count tokens in messages with images

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
import base64
import httpx2

image_url = "https://platform.claude.com/docs/images/vision-example.jpg"
image_media_type = "image/jpeg"
image_data = base64.standard_b64encode(httpx2.get(image_url).content).decode("utf-8")

client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-opus-5",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": image_media_type,
                        "data": image_data,
                    },
                },
                {"type": "text", "text": "Describe this image"},
            ],
        }
    ],
)
print(response.json())
```

Output



`{ "input_tokens": 1028 }`

An embedded image block that sets [`"oversized_image": "error"`](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#oversized-image-error) is rejected at count time exactly as the Messages API would reject it.

### Count tokens in messages with thinking



See [Thinking and the context window](https://platform.claude.com/docs/en/build-with-claude/thinking#thinking-and-the-context-window) for more details.

*   Thinking blocks from **previous** assistant turns count toward your input tokens on models that [keep all prior turns](https://platform.claude.com/docs/en/build-with-claude/thinking#thinking-block-preservation-by-model); on models that keep only the last turn, the API strips them and they do **not** count
*   **Current** assistant turn thinking **does** count toward your input tokens

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-opus-5",
    thinking={"type": "adaptive"},
    messages=[
        {
            "role": "user",
            "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?",
        },
        {
            "role": "assistant",
            "content": [
                {
                    "type": "thinking",
                    "thinking": "This is a nice number theory question. Let's think about it step by step...",
                    "signature": "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...",
                },
                {
                    "type": "text",
                    "text": "Yes, there are infinitely many prime numbers p such that p mod 4 = 3...",
                },
            ],
        },
        {"role": "user", "content": "Can you write a formal proof?"},
    ],
)

print(response.json())
```

Output



`{ "input_tokens": 88 }`

### Count tokens in messages with PDFs



Token counting supports base64-encoded PDFs with the same [PDF requirements](https://platform.claude.com/docs/en/build-with-claude/pdf-support#check-pdf-requirements) as the Messages API. This endpoint doesn't support `url` or `file` document sources.

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
import base64
import anthropic

client = anthropic.Anthropic()

with open("/path/to/document.pdf", "rb") as pdf_file:
    pdf_base64 = base64.standard_b64encode(pdf_file.read()).decode("utf-8")

response = client.messages.count_tokens(
    model="claude-opus-5",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "base64",
                        "media_type": "application/pdf",
                        "data": pdf_base64,
                    },
                },
                {"type": "text", "text": "Please summarize this document."},
            ],
        }
    ],
)

print(response.json())
```

Output



`{ "input_tokens": 2188 }`

* * *

## Token counts on Claude Fable and Claude Mythos models

Claude Fable 5.1, Claude Mythos 5.1, Claude Fable 5, and Claude Mythos 5 share the tokenizer introduced with Claude Opus 4.7. A prompt counts the same on all four, and roughly 30 percent higher than on models before Claude Opus 4.7 (the exact increase depends on the content). The token counting endpoint counts under the tokenizer of the `model` you pass. To measure the difference for your workload, count the same request twice, once with your current model and once with the model you plan to move to, and compare the two `input_tokens` values.



**Billing and migration:** Usage and billing on these models reflect this tokenizer's counts. When migrating from a model before Claude Opus 4.7, don't reuse token counts measured on the older model to estimate costs or context window fit. Count your prompts with the `model` ID you plan to use (for example, `"claude-fable-5-1"`).

* * *

## Pricing and rate limits

Token counting is **free to use** but subject to requests per minute rate limits based on your [usage tier](https://platform.claude.com/docs/en/api/rate-limits#rate-limits). If you need higher limits, use **Request rate limit increase** on the [Rate limits](https://platform.claude.com/settings/limits) page.

| Usage tier | Requests per minute (RPM) |
| --- | --- |
| Start | 5,000 |
| Build | 10,000 |
| Scale | 20,000 |



Token counting and message creation have separate and independent rate limits. Usage of one does not count against the limits of the other.

* * *

## FAQ

### Does token counting use prompt caching?

No, token counting provides an estimate without using caching logic. Although you may provide `cache_control` blocks in your token counting request, prompt caching only occurs during actual message creation.

* * *

## Next steps



[Count message tokens](https://platform.claude.com/docs/en/api/messages-count-tokens)

Read the full API reference for the token counting endpoint.



[Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)

Use token counts to keep prompts within a model's context window.



[Rate limits](https://platform.claude.com/docs/en/api/rate-limits)

Check token counts before you send a request to stay within your usage tier.



[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

Reduce cost and latency on repeated prompts by caching prompt prefixes.

## Compatibility

| Supported platforms | * Claude API * Claude Platform on AWS * Amazon Bedrock * Google Cloud * Microsoft Foundry |
| --- |

Was this page helpful?



Token counting

[ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)

Eligible

excludes[Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)

*   [How to count message tokens](https://platform.claude.com/docs/en/build-with-claude/token-counting#how-to-count-message-tokens)
*   [Supported models](https://platform.claude.com/docs/en/build-with-claude/token-counting#supported-models)
*   [Count tokens in basic messages](https://platform.claude.com/docs/en/build-with-claude/token-counting#count-tokens-in-basic-messages)
*   [Count tokens in messages with tools](https://platform.claude.com/docs/en/build-with-claude/token-counting#count-tokens-in-messages-with-tools)
*   [Count tokens in messages with images](https://platform.claude.com/docs/en/build-with-claude/token-counting#count-tokens-in-messages-with-images)
*   [Count tokens in messages with thinking](https://platform.claude.com/docs/en/build-with-claude/token-counting#count-tokens-in-messages-with-thinking)
*   [Count tokens in messages with PDFs](https://platform.claude.com/docs/en/build-with-claude/token-counting#count-tokens-in-messages-with-pdfs)
*   [Token counts on Claude Fable and Claude Mythos models](https://platform.claude.com/docs/en/build-with-claude/token-counting#token-counts-on-claude-fable-5)
*   [Pricing and rate limits](https://platform.claude.com/docs/en/build-with-claude/token-counting#pricing-and-rate-limits)
*   [FAQ](https://platform.claude.com/docs/en/build-with-claude/token-counting#faq)
*   [Next steps](https://platform.claude.com/docs/en/build-with-claude/token-counting#next-steps)
*   [Compatibility](https://platform.claude.com/docs/en/build-with-claude/token-counting#compatibility)

[Claude Platform Docs](https://platform.claude.com/docs/en/home)

[](https://x.com/claudeai)[](https://www.threads.com/@claudeai)[](https://www.linkedin.com/showcase/claude)[](https://www.youtube.com/@anthropic-ai)[](https://instagram.com/claudeai)



### Solutions

*   [AI agents](https://claude.com/solutions/agents)
*   [Code modernization](https://claude.com/solutions/code-modernization)
*   [Coding](https://claude.com/solutions/coding)
*   [Customer support](https://claude.com/solutions/customer-support)
*   [Financial services](https://claude.com/solutions/financial-services)
*   [Government](https://claude.com/solutions/government)
*   [Higher education](https://claude.com/solutions/education)
*   [K-12 teachers](https://claude.com/solutions/teachers)
*   [Life sciences](https://claude.com/solutions/life-sciences)

### Partners

*   [Claude on AWS](https://claude.com/partners/amazon-bedrock)
*   [Claude on Google Cloud](https://claude.com/partners/google-cloud-vertex-ai)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Company

*   [Anthropic](https://www.anthropic.com/company)
*   [Careers](https://www.anthropic.com/careers)
*   [Economic Futures](https://www.anthropic.com/economic-futures)
*   [Research](https://www.anthropic.com/research)
*   [News](https://www.anthropic.com/news)
*   [Responsible Scaling Policy](https://www.anthropic.com/news/announcing-our-updated-responsible-scaling-policy)
*   [Security and compliance](https://trust.anthropic.com/)
*   [Transparency](https://www.anthropic.com/transparency)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Help and security

*   [Availability](https://www.anthropic.com/supported-countries)
*   [Status](https://status.claude.com/)
*   [Support](https://support.claude.com/)
*   [Discord](https://www.anthropic.com/discord)

### Terms and policies

*   [Privacy policy](https://www.anthropic.com/legal/privacy)
*   [Responsible disclosure policy](https://www.anthropic.com/responsible-disclosure-policy)
*   [Terms of service: Commercial](https://www.anthropic.com/legal/commercial-terms)
*   [Terms of service: Consumer](https://www.anthropic.com/legal/consumer-terms)
*   [Usage policy](https://www.anthropic.com/legal/aup)

Ask Docs