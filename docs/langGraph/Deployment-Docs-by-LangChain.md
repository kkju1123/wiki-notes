---
title: Deployment - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/deploy
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:57:22.060709+00:00'
---

> ## Documentation Index
> 
> 
> Fetch the complete documentation index at:[/llms.txt](https://docs.langchain.com/llms.txt)
> 
> 
> Use this file to discover all available pages before exploring further.

[Skip to main content](https://docs.langchain.com/oss/python/langgraph/deploy#content-area)

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

Deployment

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

*   [LangSmith Cloud](https://docs.langchain.com/oss/python/langgraph/deploy#langsmith-cloud)
    *   [Prerequisites](https://docs.langchain.com/oss/python/langgraph/deploy#prerequisites)
    *   [Deploy your agent](https://docs.langchain.com/oss/python/langgraph/deploy#deploy-your-agent)
    *   [1. Create a repository on GitHub](https://docs.langchain.com/oss/python/langgraph/deploy#1-create-a-repository-on-github)
    *   [2. Deploy to LangSmith](https://docs.langchain.com/oss/python/langgraph/deploy#2-deploy-to-langsmith)
    *   [3. Test your application in Studio](https://docs.langchain.com/oss/python/langgraph/deploy#3-test-your-application-in-studio)
    *   [4. Get the API URL for your deployment](https://docs.langchain.com/oss/python/langgraph/deploy#4-get-the-api-url-for-your-deployment)
    *   [5. Test the API](https://docs.langchain.com/oss/python/langgraph/deploy#5-test-the-api)

[Production](https://docs.langchain.com/oss/python/langgraph/application-structure)

# Deployment

Copy page Copy page

Deploy LangGraph agents to production with LangSmith Cloud or JavaScript frameworks and hosting platforms.

Copy page Copy page

When you are ready to deploy your LangGraph agent to production, choose a hosting model that fits your stack. **[LangSmith Cloud](https://docs.langchain.com/langsmith/deploy-to-cloud)** provides fully managed infrastructure for stateful, long-running agents with persistent state and background execution.

LangSmith offers multiple deployment options beyond Cloud, including [hybrid](https://docs.langchain.com/langsmith/hybrid), [standalone servers](https://docs.langchain.com/langsmith/deploy-standalone-server), and [self-hosted with control plane](https://docs.langchain.com/langsmith/deploy-with-control-plane). For more information, see the [LangSmith Deployment overview](https://docs.langchain.com/langsmith/deployment).

## [​](https://docs.langchain.com/oss/python/langgraph/deploy#langsmith-cloud)

LangSmith Cloud

This section walks through deploying your agent to LangSmith Cloud from a GitHub repository. LangSmith handles infrastructure, scaling, and operational concerns.
### [​](https://docs.langchain.com/oss/python/langgraph/deploy#prerequisites)

Prerequisites

Before you begin, ensure you have the following:
*   A [GitHub account](https://github.com/)
*   A [LangSmith account](https://smith.langchain.com/?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-deploy) (free to sign up)

### [​](https://docs.langchain.com/oss/python/langgraph/deploy#deploy-your-agent)

Deploy your agent

#### [​](https://docs.langchain.com/oss/python/langgraph/deploy#1-create-a-repository-on-github)

1. Create a repository on GitHub

Your application’s code must reside in a GitHub repository to be deployed on LangSmith. Both public and private repositories are supported. For this quickstart, first make sure your app is LangGraph-compatible by following the [local server setup guide](https://docs.langchain.com/oss/python/langgraph/studio#set-up-local-agent-server). Then, push your code to the repository.
#### [​](https://docs.langchain.com/oss/python/langgraph/deploy#2-deploy-to-langsmith)

2. Deploy to LangSmith

1

Navigate to LangSmith Deployment

Log in to [LangSmith](https://smith.langchain.com/?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-deploy). In the left sidebar, select **Deployments**.

2

Create new deployment

Click the **+ New Deployment** button. A pane will open where you can fill in the required fields.

3

Link repository

If you are a first time user or adding a private repository that has not been previously connected, click the **Add new account** button and follow the instructions to connect your GitHub account.

4

Deploy repository

Select your application’s repository. Click **Submit** to deploy. This may take about 15 minutes to complete. You can check the status in the **Deployment details** view.

#### [​](https://docs.langchain.com/oss/python/langgraph/deploy#3-test-your-application-in-studio)

3. Test your application in Studio

Once your application is deployed:
1.   Select the deployment you just created to view more details.
2.   Click the **Studio** button in the top right corner. Studio will open to display your graph.

#### [​](https://docs.langchain.com/oss/python/langgraph/deploy#4-get-the-api-url-for-your-deployment)

4. Get the API URL for your deployment

1.   In the **Deployment details** view in LangGraph, click the **API URL** to copy it to your clipboard.
2.   Click the `URL` to copy it to the clipboard.

#### [​](https://docs.langchain.com/oss/python/langgraph/deploy#5-test-the-api)

5. Test the API

You can now test the API:

*   Python 
*   Rest API 

1.   Install LangGraph SDK:

```
pip install langgraph-sdk
```

1.   Send a message to the agent:

```
from langgraph_sdk import get_sync_client # or get_client for async

client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

for chunk in client.runs.stream(
    None,    # Threadless run
    "agent", # Name of agent. Defined in langgraph.json.
    input={
        "messages": [{
            "role": "human",
            "content": "What is LangGraph?",
        }],
    },
    stream_mode="updates",
):
    print(f"Receiving new event of type: {chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

```
curl -s --request POST \
    --url <DEPLOYMENT_URL>/runs/stream \
    --header 'Content-Type: application/json' \
    --header "X-Api-Key: <LANGSMITH API KEY> \
    --data "{
        \"assistant_id\": \"agent\", `# Name of agent. Defined in langgraph.json.`
        \"input\": {
            \"messages\": [
                {
                    \"role\": \"human\",
                    \"content\": \"What is LangGraph?\"
                }
            ]
        },
        \"stream_mode\": \"updates\"
    }"
```

* * *

[Connect these docs](https://docs.langchain.com/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

[Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/deploy.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).

Was this page helpful?

Yes No

[Agent Chat UI Previous](https://docs.langchain.com/oss/python/langgraph/ui)[LangSmith Observability Next](https://docs.langchain.com/oss/python/langgraph/observability)

[Docs by LangChain home page![Image 3: light logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-dark-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=5babf1a1962208fd7eed942fa2432ecb)![Image 4: dark logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-light-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=0bcd2a1f2599ed228bcedf0f535b45b1)](https://docs.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

Resources

[Forum](https://forum.langchain.com/)[Changelog](https://changelog.langchain.com/)[LangChain Academy](https://academy.langchain.com/)[Contact Sales](https://www.langchain.com/contact-sales)

Company

[Home](https://langchain.com/)[Trust Center](https://trust.langchain.com/)[Careers](https://langchain.com/careers)[Blog](https://blog.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

## Chat LangChain

[](https://chat.langchain.com/ "Open chat.langchain.com in a new tab")

![Image 6](https://t.co/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=f79c68e2-4811-48fb-896f-9ffbcbe15b74&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=26495540-c653-4b03-abf9-4e03132e52b6&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fdeploy&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894413556-685189162&tw_session_start=1&twpid=tw.1789894413556.43012456378491145&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 7](https://analytics.twitter.com/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=f79c68e2-4811-48fb-896f-9ffbcbe15b74&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=26495540-c653-4b03-abf9-4e03132e52b6&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fdeploy&tw_engaged_ms=2&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894413556-685189162&tw_session_start=1&twpid=tw.1789894413556.43012456378491145&txn_id=qr5t6&type=javascript&version=2.4.11)