---
title: Choosing between Graph and Functional APIs - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/choosing-apis
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:58:07.624594+00:00'
---

> ## Documentation Index
> 
> 
> Fetch the complete documentation index at:[/llms.txt](https://docs.langchain.com/llms.txt)
> 
> 
> Use this file to discover all available pages before exploring further.

[Skip to main content](https://docs.langchain.com/oss/python/langgraph/choosing-apis#content-area)

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

Graph API

Choosing between Graph and Functional APIs

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
    *   [Choosing APIs](https://docs.langchain.com/oss/python/langgraph/choosing-apis)
    *   [Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
    *   [Use the graph API](https://docs.langchain.com/oss/python/langgraph/use-graph-api)

*   Functional API  
*   [Runtime](https://docs.langchain.com/oss/python/langgraph/pregel)

*   [Studio](https://docs.langchain.com/langsmith/studio)

## On this page

*   [Quick decision guide](https://docs.langchain.com/oss/python/langgraph/choosing-apis#quick-decision-guide)
*   [Detailed comparison](https://docs.langchain.com/oss/python/langgraph/choosing-apis#detailed-comparison)
    *   [When to use the Graph API](https://docs.langchain.com/oss/python/langgraph/choosing-apis#when-to-use-the-graph-api)
    *   [When to use the Functional API](https://docs.langchain.com/oss/python/langgraph/choosing-apis#when-to-use-the-functional-api)

*   [Combining both APIs](https://docs.langchain.com/oss/python/langgraph/choosing-apis#combining-both-apis)
*   [Migration between APIs](https://docs.langchain.com/oss/python/langgraph/choosing-apis#migration-between-apis)
    *   [From Functional to Graph API](https://docs.langchain.com/oss/python/langgraph/choosing-apis#from-functional-to-graph-api)
    *   [From Graph to Functional API](https://docs.langchain.com/oss/python/langgraph/choosing-apis#from-graph-to-functional-api)

*   [Summary](https://docs.langchain.com/oss/python/langgraph/choosing-apis#summary)

[LangGraph APIs](https://docs.langchain.com/oss/python/langgraph/choosing-apis)

[Graph API](https://docs.langchain.com/oss/python/langgraph/choosing-apis)

# Choosing between Graph and Functional APIs

Copy page Copy page

Copy page Copy page

LangGraph provides two different APIs to build agent workflows: the **Graph API** and the **Functional API**. Both APIs share the same underlying runtime and can be used together in the same application, but they are designed for different use cases and development preferences.This guide will help you understand when to use each API based on your specific requirements.
## [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#quick-decision-guide)

Quick decision guide

Use the **Graph API** when you need:
*   **Complex workflow visualization** for debugging and documentation
*   **Explicit state management** with shared data across multiple nodes
*   **Conditional branching** with multiple decision points
*   **Parallel execution paths** that need to merge later
*   **Team collaboration** where visual representation aids understanding

Use the **Functional API** when you want:
*   **Minimal code changes** to existing procedural code
*   **Standard control flow** (if/else, loops, function calls)
*   **Function-scoped state** without explicit state management
*   **Rapid prototyping** with less boilerplate
*   **Linear workflows** with simple branching logic

## [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#detailed-comparison)

Detailed comparison

### [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#when-to-use-the-graph-api)

When to use the Graph API

The [Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api) uses a declarative approach where you define nodes, edges, and shared state to create a visual graph structure.**1. Complex decision trees and branching logic**When your workflow has multiple decision points that depend on various conditions, the Graph API makes these branches explicit and easy to visualize.

```
# Graph API: Clear visualization of decision paths
from langgraph.graph import StateGraph
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    current_tool: str
    retry_count: int

def should_continue(state):
    if state["retry_count"] > 3:
        return "end"
    elif state["current_tool"] == "search":
        return "process_search"
    else:
        return "call_llm"

workflow = StateGraph(AgentState)
workflow.add_node("call_llm", call_llm_node)
workflow.add_node("process_search", search_node)
workflow.add_conditional_edges("call_llm", should_continue)
```

**2. State management across multiple components**When you need to share and coordinate state between different parts of your workflow, the Graph API’s explicit state management is beneficial.

```
# Multiple nodes can access and modify shared state
class WorkflowState(TypedDict):
    user_input: str
    search_results: list
    generated_response: str
    validation_status: str

def search_node(state):
    # Access shared state
    results = search(state["user_input"])
    return {"search_results": results}

def validation_node(state):
    # Access results from previous node
    is_valid = validate(state["generated_response"])
    return {"validation_status": "valid" if is_valid else "invalid"}
```

**3. Parallel processing with synchronization**When you need to run multiple operations in parallel and then combine their results, the Graph API handles this naturally.

```
# Parallel processing of multiple data sources
workflow.add_node("fetch_news", fetch_news)
workflow.add_node("fetch_weather", fetch_weather)
workflow.add_node("fetch_stocks", fetch_stocks)
workflow.add_node("combine_data", combine_all_data)

# All fetch operations run in parallel
workflow.add_edge(START, "fetch_news")
workflow.add_edge(START, "fetch_weather")
workflow.add_edge(START, "fetch_stocks")

# Combine waits for all parallel operations to complete
workflow.add_edge("fetch_news", "combine_data")
workflow.add_edge("fetch_weather", "combine_data")
workflow.add_edge("fetch_stocks", "combine_data")
```

**4. Team development and documentation**The visual nature of the Graph API makes it easier for teams to understand, document, and maintain complex workflows.

```
# Clear separation of concerns - each team member can work on different nodes
workflow.add_node("data_ingestion", data_team_function)
workflow.add_node("ml_processing", ml_team_function)
workflow.add_node("business_logic", product_team_function)
workflow.add_node("output_formatting", frontend_team_function)
```

### [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#when-to-use-the-functional-api)

When to use the Functional API

The [Functional API](https://docs.langchain.com/oss/python/langgraph/functional-api) uses an imperative approach that integrates LangGraph features into standard procedural code.**1. Existing procedural code**When you have existing code that uses standard control flow and want to add LangGraph features with minimal refactoring.

```
# Functional API: Minimal changes to existing code
from langgraph.func import entrypoint, task

@task
def process_user_input(user_input: str) -> dict:
    # Existing function with minimal changes
    return {"processed": user_input.lower().strip()}

@entrypoint(checkpointer=checkpointer)
def workflow(user_input: str) -> str:
    # Standard Python control flow
    processed = process_user_input(user_input).result()

    if "urgent" in processed["processed"]:
        response = handle_urgent_request(processed).result()
    else:
        response = handle_normal_request(processed).result()

    return response
```

**2. Linear workflows with simple logic**When your workflow is primarily sequential with straightforward conditional logic.

```
@entrypoint(checkpointer=checkpointer)
def essay_workflow(topic: str) -> dict:
    # Linear flow with simple branching
    outline = create_outline(topic).result()

    if len(outline["points"]) < 3:
        outline = expand_outline(outline).result()

    draft = write_draft(outline).result()

    # Human review checkpoint
    feedback = interrupt({"draft": draft, "action": "Please review"})

    if feedback == "approve":
        final_essay = draft
    else:
        final_essay = revise_essay(draft, feedback).result()

    return {"essay": final_essay}
```

**3. Rapid prototyping**When you want to quickly test ideas without the overhead of defining state schemas and graph structures.

```
@entrypoint(checkpointer=checkpointer)
def quick_prototype(data: dict) -> dict:
    # Fast iteration - no state schema needed
    step1_result = process_step1(data).result()
    step2_result = process_step2(step1_result).result()

    return {"final_result": step2_result}
```

**4. Function-scoped state management**When your state is naturally scoped to individual functions and doesn’t need to be shared broadly.

```
@task
def analyze_document(document: str) -> dict:
    # Local state management within function
    sections = extract_sections(document)
    summaries = [summarize(section) for section in sections]
    key_points = extract_key_points(summaries)

    return {
        "sections": len(sections),
        "summaries": summaries,
        "key_points": key_points
    }

@entrypoint(checkpointer=checkpointer)
def document_processor(document: str) -> dict:
    analysis = analyze_document(document).result()
    # State is passed between functions as needed
    return generate_report(analysis).result()
```

## [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#combining-both-apis)

Combining both APIs

You can use both APIs together in the same application. This is useful when different parts of your system have different requirements.

```
from langgraph.graph import StateGraph
from langgraph.func import entrypoint

# Complex multi-agent coordination using Graph API
coordination_graph = StateGraph(CoordinationState)
coordination_graph.add_node("orchestrator", orchestrator_node)
coordination_graph.add_node("agent_a", agent_a_node)
coordination_graph.add_node("agent_b", agent_b_node)

# Simple data processing using Functional API
@entrypoint()
def data_processor(raw_data: dict) -> dict:
    cleaned = clean_data(raw_data).result()
    transformed = transform_data(cleaned).result()
    return transformed

# Use the functional API result in the graph
def orchestrator_node(state):
    processed_data = data_processor.invoke(state["raw_data"])
    return {"processed_data": processed_data}
```

## [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#migration-between-apis)

Migration between APIs

### [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#from-functional-to-graph-api)

From Functional to Graph API

When your functional workflow grows complex, you can migrate to the Graph API:

```
# Before: Functional API
@entrypoint(checkpointer=checkpointer)
def complex_workflow(input_data: dict) -> dict:
    step1 = process_step1(input_data).result()

    if step1["needs_analysis"]:
        analysis = analyze_data(step1).result()
        if analysis["confidence"] > 0.8:
            result = high_confidence_path(analysis).result()
        else:
            result = low_confidence_path(analysis).result()
    else:
        result = simple_path(step1).result()

    return result

# After: Graph API
class WorkflowState(TypedDict):
    input_data: dict
    step1_result: dict
    analysis: dict
    final_result: dict

def should_analyze(state):
    return "analyze" if state["step1_result"]["needs_analysis"] else "simple_path"

def confidence_check(state):
    return "high_confidence" if state["analysis"]["confidence"] > 0.8 else "low_confidence"

workflow = StateGraph(WorkflowState)
workflow.add_node("step1", process_step1_node)
workflow.add_conditional_edges("step1", should_analyze)
workflow.add_node("analyze", analyze_data_node)
workflow.add_conditional_edges("analyze", confidence_check)
# ... add remaining nodes and edges
```

### [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#from-graph-to-functional-api)

From Graph to Functional API

When your graph becomes overly complex for simple linear processes:

```
# Before: Over-engineered Graph API
class SimpleState(TypedDict):
    input: str
    step1: str
    step2: str
    result: str

# After: Simplified Functional API
@entrypoint(checkpointer=checkpointer)
def simple_workflow(input_data: str) -> str:
    step1 = process_step1(input_data).result()
    step2 = process_step2(step1).result()
    return finalize_result(step2).result()
```

## [​](https://docs.langchain.com/oss/python/langgraph/choosing-apis#summary)

Summary

Choose the **Graph API** when you need explicit control over workflow structure, complex branching, parallel processing, or team collaboration benefits.Choose the **Functional API** when you want to add LangGraph features to existing code with minimal changes, have simple linear workflows, or need rapid prototyping capabilities.Both APIs provide the same core LangGraph features (persistence, streaming, human-in-the-loop, memory) but package them in different paradigms to suit different development styles and use cases.

* * *

[Connect these docs](https://docs.langchain.com/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

[Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/choosing-apis.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).

Was this page helpful?

Yes No

[Custom stream channels Previous](https://docs.langchain.com/oss/python/langgraph/frontend/custom-stream-channels)[Graph API overview Next](https://docs.langchain.com/oss/python/langgraph/graph-api)

[Docs by LangChain home page![Image 3: light logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-dark-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=5babf1a1962208fd7eed942fa2432ecb)![Image 4: dark logo](https://mintcdn.com/langchain-5e9cc07a/nQm-sjd_MByLhgeW/images/brand/langchain-docs-light-blue.png?fit=max&auto=format&n=nQm-sjd_MByLhgeW&q=85&s=0bcd2a1f2599ed228bcedf0f535b45b1)](https://docs.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

Resources

[Forum](https://forum.langchain.com/)[Changelog](https://changelog.langchain.com/)[LangChain Academy](https://academy.langchain.com/)[Contact Sales](https://www.langchain.com/contact-sales)

Company

[Home](https://langchain.com/)[Trust Center](https://trust.langchain.com/)[Careers](https://langchain.com/careers)[Blog](https://blog.langchain.com/)

[github](https://github.com/langchain-ai)[x](https://x.com/LangChain)[linkedin](https://www.linkedin.com/company/langchain)[youtube](https://www.youtube.com/@LangChain)

## Chat LangChain

[](https://chat.langchain.com/ "Open chat.langchain.com in a new tab")

![Image 6](https://t.co/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=e6984b05-c59e-49ae-babb-03a08ce4098a&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=541286d4-067c-45f3-ab72-170893cc6b85&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fchoosing-apis&tw_engaged_ms=1&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894459211-271657084&tw_session_start=1&twpid=tw.1789894459210.445460455345390724&txn_id=qr5t6&type=javascript&version=2.4.11)![Image 7](https://analytics.twitter.com/1/i/adsct?bci=4&dv=UTC%26en-US%2Cen%26Google%20Inc.%26Linux%20x86_64%26255%261280%261280%2610%2624%261280%261280%260%26na&eci=3&event=%7B%7D&event_id=e6984b05-c59e-49ae-babb-03a08ce4098a&integration=gtm&p_id=Twitter&p_user_id=0&pl_id=541286d4-067c-45f3-ab72-170893cc6b85&tw_ch_fvl=Google%20Chrome%2F153.0.8010.47%2CNot_A%20Brand%2F8.0.0.0%2CChromium%2F153.0.8010.47&tw_document_href=https%3A%2F%2Fdocs.langchain.com%2Foss%2Fpython%2Flanggraph%2Fchoosing-apis&tw_engaged_ms=1&tw_iframe_status=0&tw_pid_src=1&tw_session_count=1&tw_session_id=1789894459211-271657084&tw_session_start=1&twpid=tw.1789894459210.445460455345390724&txn_id=qr5t6&type=javascript&version=2.4.11)