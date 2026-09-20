---
title: Run a local server - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/local-server
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:55:04.841160+00:00'
---

> ## Documentation Index
> 
> 
> Fetch the complete documentation index at:[/llms.txt](https://docs.langchain.com/llms.txt)
> 
> 
> Use this file to discover all available pages before exploring further.

[Skip to main content](https://docs.langchain.com/oss/python/langgraph/local-server#content-area)

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

Get started

Run a local server

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

*   [Prerequisites](https://docs.langchain.com/oss/python/langgraph/local-server#prerequisites)
*   [1. Install the LangGraph CLI](https://docs.langchain.com/oss/python/langgraph/local-server#1-install-the-langgraph-cli)
*   [2. Create a LangGraph app](https://docs.langchain.com/oss/python/langgraph/local-server#2-create-a-langgraph-app)
*   [3. Install dependencies](https://docs.langchain.com/oss/python/langgraph/local-server#3-install-dependencies)
*   [4. Create a .env file](https://docs.langchain.com/oss/python/langgraph/local-server#4-create-a-env-file)
*   [5. Launch Agent server](https://docs.langchain.com/oss/python/langgraph/local-server#5-launch-agent-server)
*   [6. Test your application in Studio](https://docs.langchain.com/oss/python/langgraph/local-server#6-test-your-application-in-studio)
*   [7. Test the API](https://docs.langchain.com/oss/python/langgraph/local-server#7-test-the-api)
*   [Next steps](https://docs.langchain.com/oss/python/langgraph/local-server#next-steps)

[Get started](https://docs.langchain.com/oss/python/langgraph/install)

# Run a local server

Copy page Copy page

Copy page Copy page

This guide shows you how to run a LangGraph application locally.
## [​](https://docs.langchain.com/oss/python/langgraph/local-server#prerequisites)

Prerequisites

Before you begin, ensure you have the following:
*   An API key for [LangSmith](https://smith.langchain.com/settings) - free to sign up

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#1-install-the-langgraph-cli)

1. Install the LangGraph CLI

pip

uv

```
# Python >= 3.11 is required.
pip install -U "langgraph-cli[inmem]"
```

```
# Python >= 3.11 is required.
uv add "langgraph-cli[inmem]"
```

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#2-create-a-langgraph-app)

2. Create a LangGraph app

Create a new app from the [`new-langgraph-project-python` template](https://github.com/langchain-ai/new-langgraph-project). This template demonstrates a single-node application you can extend with your own logic.

```
langgraph new path/to/your/app --template new-langgraph-project-python
```

**Additional templates** If you use `langgraph new` without specifying a template, you will be presented with an interactive menu that will allow you to choose from a list of available templates.

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#3-install-dependencies)

3. Install dependencies

In the root of your new LangGraph app, install the dependencies in `edit` mode so your local changes are used by the server:

pip

uv

```
cd path/to/your/app
pip install -e .
```

```
cd path/to/your/app
uv sync
```

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#4-create-a-env-file)

4. Create a `.env` file

You will find a `.env.example` in the root of your new LangGraph app. Create a `.env` file in the root of your new LangGraph app and copy the contents of the `.env.example` file into it, filling in the necessary API keys:

```
LANGSMITH_API_KEY=lsv2...
```

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#5-launch-agent-server)

5. Launch Agent server

Start the LangGraph API server locally:

```
langgraph dev
```

Sample output:

```
INFO:langgraph_api.cli:

        Welcome to

╦  ┌─┐┌┐┌┌─┐╔═╗┬─┐┌─┐┌─┐┬ ┬
║  ├─┤││││ ┬║ ╦├┬┘├─┤├─┘├─┤
╩═╝┴ ┴┘└┘└─┘╚═╝┴└─┴ ┴┴  ┴ ┴

- 🚀 API: http://127.0.0.1:2024
- 🎨 Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
- 📚 API Docs: http://127.0.0.1:2024/docs

This in-memory server is designed for development and testing.
For production use, please use LangSmith Deployment.
```

The `langgraph dev` command starts Agent Server in an in-memory mode. This mode is suitable for development and testing purposes. For production use, deploy Agent Server with access to a persistent storage backend. For more information, see the [Platform setup overview](https://docs.langchain.com/langsmith/platform-setup).
## [​](https://docs.langchain.com/oss/python/langgraph/local-server#6-test-your-application-in-studio)

6. Test your application in Studio

[Studio](https://docs.langchain.com/langsmith/studio) is a specialized UI that you can connect to LangGraph API server to visualize, interact with, and debug your application locally. Test your graph in Studio by visiting the URL provided in the output of the `langgraph dev` command:

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

For an Agent Server running on a custom host/port, update the `baseUrl` query parameter in the URL. For example, if your server is running on `http://myhost:3000`:

```
https://smith.langchain.com/studio/?baseUrl=http://myhost:3000
```

Safari compatibility

Use the `--tunnel` flag with your command to create a secure tunnel, as Safari has limitations when connecting to localhost servers:

```
langgraph dev --tunnel
```

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#7-test-the-api)

7. Test the API

*   Python SDK (async) 
*   Python SDK (sync) 
*   Rest API 

1.   Install the LangGraph Python SDK:  ```
pip install langgraph-sdk
```      
2.   Send a message to the assistant (threadless run):  ```
from langgraph_sdk import get_client
import asyncio

client = get_client(url="http://localhost:2024")

async def main():
    async for chunk in client.runs.stream(
        None,  # Threadless run
        "agent", # Name of assistant. Defined in langgraph.json.
        input={
        "messages": [{
            "role": "human",
            "content": "What is LangGraph?",
            }],
        },
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")

asyncio.run(main())
```        

1.   Install the LangGraph Python SDK:  ```
pip install langgraph-sdk
```      
2.   Send a message to the assistant (threadless run):  ```
from langgraph_sdk import get_sync_client

client = get_sync_client(url="http://localhost:2024")

for chunk in client.runs.stream(
    None,  # Threadless run
    "agent", # Name of assistant. Defined in langgraph.json.
    input={
        "messages": [{
            "role": "human",
            "content": "What is LangGraph?",
        }],
    },
    stream_mode="messages-tuple",
):
    print(f"Receiving new event of type: {chunk.event}...")
    print(chunk.data)
    print("\n\n")
```      

```
curl -s --request POST \
    --url "http://localhost:2024/runs/stream" \
    --header 'Content-Type: application/json' \
    --data "{
        \"assistant_id\": \"agent\",
        \"input\": {
            \"messages\": [
                {
                    \"role\": \"human\",
                    \"content\": \"What is LangGraph?\"
                }
            ]
        },
        \"stream_mode\": \"messages-tuple\"
    }"
```

## [​](https://docs.langchain.com/oss/python/langgraph/local-server#next-steps)

Next steps

Now that you have a LangGraph app running locally, take your journey further by exploring deployment and advanced features:
*   [Deployment quickstart](https://docs.langchain.com/langsmith/deployment-quickstart): Deploy your LangGraph app using LangSmith.
*   [LangSmith](https://docs.langchain.com/langsmith/observability): Learn about foundational LangSmith concepts.
*   [SDK Reference](https://reference.langchain.com/python/langsmith/deployment/sdk/): Explore the SDK API Reference.

* * *

[Connect these docs](https://docs.langchain.com/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

[Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/local-server.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).

Was this page helpful?

Yes No

[Quickstart Previous](https://docs.langchain.com/oss/python/langgraph/quickstart)[Changelog Next](https://docs.langchain.com/oss/python/langgraph/changelog-py)

[Docs by LangChain home page![Image 3: light logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-dark-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=5babf1a1962208fd7eed942fa2432ecb)![Image 4: dark logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-light-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=0bcd2a1f2599ed228bcedf0f535b45b1)](https://docs.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

Resources

[Forum](https://forum.langchain.com/)[Changelog](https://changelog.langchain.com/)[LangChain Academy](https://academy.langchain.com/)[Contact Sales](https://www.langchain.com/contact-sales)

Company

[Home](https://langchain.com/)[Trust Center](https://trust.langchain.com/)[Careers](https://langchain.com/careers)[Blog](https://blog.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

## Chat LangChain

[](https://chat.langchain.com/ "Open chat.langchain.com in a new tab")

![Image 6](https://t.co/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=caa5d006-9695-433f-919f-8f4e75d524f3&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=f5655382-a9bf-4370-a818-e12bc748f583&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Flocal-server&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894276886-804940130&tw_session_start=1&twpid=tw.1789894276886.983002308560088032&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 7](https://analytics.twitter.com/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=caa5d006-9695-433f-919f-8f4e75d524f3&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=f5655382-a9bf-4370-a818-e12bc748f583&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Flocal-server&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894276886-804940130&tw_session_start=1&twpid=tw.1789894276886.983002308560088032&txn_id=qr5t6&type=javascript&version=2.4.11)