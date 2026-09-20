---
title: Agent Chat UI - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/ui
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:57:13.503848+00:00'
---

> ## Documentation Index
> 
> 
> Fetch the complete documentation index at:[/llms.txt](https://docs.langchain.com/llms.txt)
> 
> 
> Use this file to discover all available pages before exploring further.

[Skip to main content](https://docs.langchain.com/oss/python/langgraph/ui#content-area)

Interrupt is coming to NYC and London this fall. Join the builders, engineers, and teams shaping what's next for agents. [Get your tickets →](https://interrupt.langchain.com/)

[Docs by LangChain home page![Image 1: light logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-dark-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=5babf1a1962208fd7eed942fa2432ecb)![Image 2: dark logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-light-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=0bcd2a1f2599ed228bcedf0f535b45b1)](https://docs.langchain.com/)Build

Search...

Ctrl K

*   [Ask AI](https://chat.langchain.com/)
*   [GitHub](https://github.com/langchain-ai)
*   [Try LangSmith](https://smith.langchain.com/)
*   [Try LangSmith](https://smith.langchain.com/)

Search...

Navigation

Production

Agent Chat UI

[Overview](https://docs.langchain.com/build-overview)[Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview)[Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview)[LangChain](https://docs.langchain.com/oss/python/langchain/overview)[LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)[OpenWiki](https://docs.langchain.com/oss/openwiki/overview)[Integrations](https://docs.langchain.com/oss/python/integrations/providers/overview)[Learn](https://docs.langchain.com/oss/python/learn)[Reference](https://docs.langchain.com/oss/python/reference/overview)[Contribute](https://docs.langchain.com/oss/python/contributing/overview)

Python

*   [Overview](https://docs.langchain.com/oss/python/langgraph/overview)

### Get started

*   [Install](https://docs.langchain.com/oss/python/langgraph/install)
*   [Quickstart](https://docs.langchain.com/oss/python/langgraph/quickstart)
*   [Local server](https://docs.langchain.com/oss/python/langgraph/local-server)
*   [Changelog](https://docs.langchain.com/oss/python/releases/changelog)
*   [Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph)
*   [Workflows + agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)

### Capabilities

*   [Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
*   [Checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers)
*   [Stores](https://docs.langchain.com/oss/python/langgraph/stores)
*   [Fault tolerance](https://docs.langchain.com/oss/python/langgraph/fault-tolerance)
*   [Event streaming](https://docs.langchain.com/oss/python/langgraph/event-streaming)
*   [Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)
*   [Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
*   [Time travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel)
*   [Memory](https://docs.langchain.com/oss/python/langgraph/add-memory)
*   [Subgraphs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs)

### Production

*   [Application structure](https://docs.langchain.com/oss/python/langgraph/application-structure)
*   [Test](https://docs.langchain.com/oss/python/langgraph/test)
*   [Backward compatibility](https://docs.langchain.com/oss/python/langgraph/backward-compatibility)
*   [LangSmith Studio](https://docs.langchain.com/oss/python/langgraph/studio)
*   [Agent Chat UI](https://docs.langchain.com/oss/python/langgraph/ui)
*   [Deployment](https://docs.langchain.com/oss/python/langgraph/deploy)
*   [LangSmith Observability](https://docs.langchain.com/oss/python/langgraph/observability)

### Frontend

*   [Overview](https://docs.langchain.com/oss/python/langgraph/frontend/overview)
*   [Graph execution](https://docs.langchain.com/oss/python/langgraph/frontend/graph-execution)
*   [Custom stream channels](https://docs.langchain.com/oss/python/langgraph/frontend/custom-stream-channels)

### LangGraph APIs

*   Graph API  
*   Functional API  
*   [Runtime](https://docs.langchain.com/oss/python/langgraph/pregel)

*   [Studio](https://docs.langchain.com/langsmith/studio)

## On this page

*   [Quick start](https://docs.langchain.com/oss/python/langgraph/ui#quick-start)
*   [Local development](https://docs.langchain.com/oss/python/langgraph/ui#local-development)
*   [Connect to your agent](https://docs.langchain.com/oss/python/langgraph/ui#connect-to-your-agent)

[Production](https://docs.langchain.com/oss/python/langgraph/application-structure)

# Agent Chat UI

Copy page Copy page

Copy page Copy page

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) is a Next.js application that provides a conversational interface for interacting with any LangChain agent. It supports real-time chat, tool visualization, and advanced features like time-travel debugging and state forking. Agent Chat UI works seamlessly with agents created using [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) and provides interactive experiences for your agents with minimal setup, whether you’re running locally or in a deployed context (such as [LangSmith](https://docs.langchain.com/langsmith/observability)).Agent Chat UI is open source and can be adapted to your application needs.

[Video 2](https://www.youtube.com/watch?v=lInrwVnZ83o)

You can use generative UI in the Agent Chat UI. For more information, see [Implement generative user interfaces with LangGraph](https://docs.langchain.com/langsmith/generative-ui-react).

### [​](https://docs.langchain.com/oss/python/langgraph/ui#quick-start)

Quick start

The fastest way to get started is using the hosted version:
1.   **Visit [Agent Chat UI](https://agentchat.vercel.app/)**
2.   **Connect your agent** by entering your deployment URL or local server address
3.   **Start chatting** - the UI will automatically detect and render tool calls and interrupts

### [​](https://docs.langchain.com/oss/python/langgraph/ui#local-development)

Local development

For customization or local development, you can run Agent Chat UI locally:

Use npx

Clone repository

```
# Create a new Agent Chat UI project
npx create-agent-chat-app --project-name my-chat-ui
cd my-chat-ui

# Install dependencies and start
pnpm install
pnpm dev
```

```
# Clone the repository
git clone https://github.com/langchain-ai/agent-chat-ui.git
cd agent-chat-ui

# Install dependencies and start
pnpm install
pnpm dev
```

### [​](https://docs.langchain.com/oss/python/langgraph/ui#connect-to-your-agent)

Connect to your agent

Agent Chat UI can connect to both [local](https://docs.langchain.com/oss/python/langgraph/studio#set-up-local-agent-server) and [deployed agents](https://docs.langchain.com/oss/python/langgraph/deploy).After starting Agent Chat UI, you’ll need to configure it to connect to your agent:
1.   **Graph ID**: Enter your graph name (find this under `graphs` in your `langgraph.json` file)
2.   **Deployment URL**: Your Agent server’s endpoint (e.g., `http://localhost:2024` for local development, or your deployed agent’s URL)
3.   **LangSmith API key (optional)**: Add your LangSmith API key (not required if you’re using a local Agent server)

Once configured, Agent Chat UI will automatically fetch and display any interrupted threads from your agent.

Agent Chat UI has out-of-the-box support for rendering tool calls and tool result messages. To customize what messages are shown, see [Hiding Messages in the Chat](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat).

* * *

[Connect these docs](https://docs.langchain.com/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

[Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/ui.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).

Was this page helpful?

Yes No

[LangSmith Studio Previous](https://docs.langchain.com/oss/python/langgraph/studio)[Deployment Next](https://docs.langchain.com/oss/python/langgraph/deploy)

[Docs by LangChain home page![Image 3: light logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-dark-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=5babf1a1962208fd7eed942fa2432ecb)![Image 4: dark logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-light-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=0bcd2a1f2599ed228bcedf0f535b45b1)](https://docs.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

Resources

[Forum](https://forum.langchain.com/)[Changelog](https://changelog.langchain.com/)[LangChain Academy](https://academy.langchain.com/)[Contact Sales](https://www.langchain.com/contact-sales)

Company

[Home](https://langchain.com/)[Trust Center](https://trust.langchain.com/)[Careers](https://langchain.com/careers)[Blog](https://blog.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

## Chat LangChain

[](https://chat.langchain.com/ "Open chat.langchain.com in a new tab")

![Image 6](https://t.co/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=6184575e-10ce-4d33-90d7-b578ff671af7&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=4e0aa72b-4b7e-4f35-92b8-a9580df96b1d&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fui&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894396568-729720796&tw_session_start=1&twpid=tw.1789894396568.98406378991693619&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 7](https://analytics.twitter.com/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=6184575e-10ce-4d33-90d7-b578ff671af7&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=4e0aa72b-4b7e-4f35-92b8-a9580df96b1d&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fui&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894396568-729720796&tw_session_start=1&twpid=tw.1789894396568.98406378991693619&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 8](https://t.co/i/adsct?bci=4&cv=100%261&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=2&event_id=aaf352f5-4a50-49d0-9d92-7b6241b41cd2&events=%5B%5B%22auto_long_site_dwell%22%2C%7B%7D%5D%5D&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=4e0aa72b-4b7e-4f35-92b8-a9580df96b1d&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fui&tw_engaged_ms=10001&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894396568-729720796&twpid=tw.1789894396568.98406378991693619&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 9](https://analytics.twitter.com/i/adsct?bci=4&cv=100%261&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=2&event_id=aaf352f5-4a50-49d0-9d92-7b6241b41cd2&events=%5B%5B%22auto_long_site_dwell%22%2C%7B%7D%5D%5D&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=4e0aa72b-4b7e-4f35-92b8-a9580df96b1d&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fui&tw_engaged_ms=10001&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894396568-729720796&twpid=tw.1789894396568.98406378991693619&txn_id=qr5t6&type=javascript&version=2.4.11)