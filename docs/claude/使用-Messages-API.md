---
title: 使用 Messages API
url: https://platform.claude.com/docs/en/build-with-claude/working-with-messages
source_type: web
folder: claude
author: null
tags:
- Claude API
- Messages API
- 对话式 AI
- Anthropic
- 工具调用
summary: Claude Messages API 是 Anthropic 推荐的对话式接口，通过 messages 数组管理多轮会话，支持文本、图像、工具调用与流式输出。
fetched_at: '2026-09-20T02:16:56.127539+00:00'
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

[Console](https://platform.claude.com/)[Log in](https://platform.claude.com/login?returnTo=%2Fdocs%2Fen%2Fbuild-with-claude%2Fworking-with-messages)

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

[Messages](https://platform.claude.com/docs/en/intro)Building with Claude

# Using the Messages API

Copy page



Practical patterns and examples for using the Messages API effectively

Copy page



Anthropic offers two ways to build with Claude, each suited to different use cases:

|  | Messages API | Claude Managed Agents |
| --- | --- | --- |
| **What it is** | Direct model prompting access | Pre-built, configurable agent harness that runs in managed infrastructure |
| **Best for** | Custom agent loops and fine-grained control | Long-running tasks and asynchronous work |

This guide covers common patterns for working with the Messages API, including basic requests, multi-turn conversations, prefill techniques, and vision capabilities. For complete API specifications, see the [Messages API reference](https://platform.claude.com/docs/en/api/messages/create). For the managed agent harness instead, see the [Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview).



To learn how zero data retention (ZDR) applies to this feature, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

## Basic request and response



The `temperature`, `top_p`, and `top_k` sampling parameters are not supported on Claude 4.7 and later models and Claude Mythos Preview. Setting them to a non-default value returns a 400 error. Omit them from request payloads and use prompting to guide the model's behavior instead. See the [migration guide](https://platform.claude.com/docs/en/models/opus-5/migration-guide#migrating-from-claude-opus-47).

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
message = anthropic.Anthropic().messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
print(message)
```

Output



```
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello!"
    }
  ],
  "model": "claude-opus-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 12,
    "output_tokens": 6
  }
}
```

Refusal responses (`stop_reason: "refusal"`) also include a `stop_details` object identifying the policy category that triggered the refusal, on every model. See [Handling stop reasons](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#refusal-response) for the field reference and example handling code.

## Multiple conversational turns

The Messages API is stateless, which means that you always send the full conversational history to the API. You can use this pattern to build up a conversation over time. Earlier conversational turns don't necessarily need to actually originate from Claude. You can use synthetic `assistant` messages.

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
message = anthropic.Anthropic().messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"},
        {"role": "assistant", "content": "Hello!"},
        {"role": "user", "content": "Can you describe LLMs to me?"},
    ],
)
print(message)
```

Output



```
{
  "id": "msg_018gCsTGsXkYJVqYPxTgDHBU",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Sure, I'd be happy to provide..."
    }
  ],
  "model": "claude-opus-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 30,
    "output_tokens": 309
  }
}
```

### System role in messages

On Claude Fable 5.1, [Claude Mythos 5.1](https://anthropic.com/glasswing), Claude Fable 5, [Claude Mythos 5](https://anthropic.com/glasswing), Claude Opus 4.8, and Claude Opus 5, you can include messages with `"role": "system"` after a user turn (subject to [placement rules](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages#limitations)) to add a new system instruction partway through a conversation. A `system` message cannot be the first entry in `messages`. Use the top-level `system` field for instructions that apply from the start.

A mid-conversation system message has the same authority as the top-level `system` field, but because it is appended to the end of the message history, it does not invalidate any cached prefix that came before it. Use the top-level `system` field for instructions that should apply from the very first turn, and a mid-conversation system message for instructions that only become relevant later.

See [Mid-conversation system messages](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages) for the complete guide, including how to combine it with [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).

## Prefilling Claude's response

You can pre-fill part of Claude's response in the last position of the input messages list. Use this technique to shape Claude's response. The following example uses `"max_tokens": 1` to get a single multiple choice answer from Claude.



Prefilling is not supported on Claude 4.6 and later models and [Claude Mythos Preview](https://anthropic.com/glasswing). Requests using prefill with these models return a 400 error. Use [structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) on models that support it, or system prompt instructions, instead. See the [migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide) for migration patterns.

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
message = anthropic.Anthropic().messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1,
    messages=[
        {
            "role": "user",
            "content": "What is latin for Ant? (A) Apoidea, (B) Rhopalocera, (C) Formicidae",
        },
        {"role": "assistant", "content": "The answer is ("},
    ],
)
print(message)
```

Output



```
{
  "id": "msg_01Q8Faay6S7QPTvEUUQARt7h",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "C"
    }
  ],
  "model": "claude-sonnet-4-5",
  "stop_reason": "max_tokens",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 42,
    "output_tokens": 1
  }
}
```

## Vision

Claude can read both text and images in requests. You can supply images using the `base64`, `url`, or `file` source types. The `file` source type references an image uploaded through the [Files API](https://platform.claude.com/docs/en/build-with-claude/files). Supported media types are `image/jpeg`, `image/png`, `image/gif`, and `image/webp`. See the [vision guide](https://platform.claude.com/docs/en/build-with-claude/vision) for more details.

cURL CLI Python TypeScript C#Go Java PHP Ruby



```
import base64
import httpx2

# Option 1: Base64-encoded image
image_url = "https://platform.claude.com/docs/images/vision-example.jpg"
image_media_type = "image/jpeg"
image_data = base64.standard_b64encode(httpx2.get(image_url).content).decode("utf-8")

message = anthropic.Anthropic().messages.create(
    model="claude-opus-5",
    max_tokens=1024,
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
                {"type": "text", "text": "What is in the above image?"},
            ],
        }
    ],
)
print(message)

# Option 2: URL-referenced image
message_from_url = anthropic.Anthropic().messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "url",
                        "url": "https://platform.claude.com/docs/images/vision-example.jpg",
                    },
                },
                {"type": "text", "text": "What is in the above image?"},
            ],
        }
    ],
)
print(message_from_url)
```

Output



```
{
  "id": "msg_011CdKmWtV3oFx1C5yUbf5CY",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "This image is a beautiful minimalist/flat-design illustration of a sunset landscape. Here's what it contains:\n\n**Sky & Sun:**\n- A warm gradient sky transitioning from golden-yellow at the top to deep orange toward the horizon\n- A large pale yellow sun positioned in the upper-right area\n\n**Birds:**\n- Three small silhouetted birds flying in the upper-left portion of the sky, depicted as simple \"M\" or \"v\" shapes\n\n**Mountains:**\n- Multiple layered mountain peaks in purple and maroon tones\n- The mountains overlap to create depth, with varying shades of dusty purple and deep burgundy\n\n**Water:**\n- A dark purple body of water at the bottom of the image\n- A reflection of the sun shown as horizontal cream/peach colored lines in the center-bottom area\n\nThe overall style is clean, geometric, and uses a warm sunset color palette (oranges, yellows, purples, and maroons), giving it a peaceful, serene aesthetic typical of modern vector/flat design artwork."
    }
  ],
  "model": "claude-opus-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 1030,
    "output_tokens": 350
  }
}
```

## Next steps



[Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)

Handle each `stop_reason` value and decide what to do when a response ends.



[Tool use with Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

Give Claude tools to call external services and APIs from within the Messages API.



[Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)

Control desktop computer environments with the Messages API.

[Browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool)

Let Claude navigate, read, and interact with webpages in a browser you run.



[Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)

Get guaranteed, schema-validated JSON output from Claude.



[Task budgets](https://platform.claude.com/docs/en/build-with-claude/task-budgets)

Set an advisory token budget across a full agentic loop with `output_config.task_budget`.

Was this page helpful?



*   [Basic request and response](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#basic-request-and-response)
*   [Multiple conversational turns](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#multiple-conversational-turns)
*   [System role in messages](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#system-role-in-messages)
*   [Prefilling Claude's response](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#putting-words-in-claudes-mouth)
*   [Vision](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#vision)
*   [Next steps](https://platform.claude.com/docs/en/build-with-claude/working-with-messages#next-steps)

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