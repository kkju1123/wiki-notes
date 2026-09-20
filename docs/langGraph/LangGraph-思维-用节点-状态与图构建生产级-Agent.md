---
title: LangGraph 思维：用节点、状态与图构建生产级 Agent
url: https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph
source_type: web
folder: langGraph
author: null
tags:
- LangGraph
- 智能体
- 工作流编排
- 状态管理
- 人机协同
summary: 以客服邮件 Agent 为例，讲透 LangGraph 的节点拆分、状态设计和错误处理思维，把复杂流程建成可恢复、可观测的图。
fetched_at: '2026-09-20T00:54:48.182651+00:00'
---

When you build an agent with LangGraph, you will first break it apart into discrete steps called **nodes**. Then, you will describe the different decisions and transitions from each of your nodes. Finally, you connect nodes together through a shared **state** that each node can read from and write to.In this walkthrough, we’ll guide you through the thought process of building a customer support email agent with LangGraph.

## Start with the process you want to automate

Imagine that you need to build an AI agent that handles customer support emails. Your product team has given you these requirements:

To implement an agent in LangGraph, you will usually follow the same five steps.

## Step 1: Map out your workflow as discrete steps

Start by identifying the distinct steps in your process. Each step will become a **node** (a function that does one specific thing). Then, sketch how these steps connect to each other.

The arrows in this diagram show possible paths, but the actual decision of which path to take happens inside each node.Now that we’ve identified the components in our workflow, let’s understand what each node needs to do:

*   `Read Email`: Extract and parse the email content
*   `Classify Intent`: Use an LLM to categorize urgency and topic, then route to appropriate action
*   `Doc Search`: Query your knowledge base for relevant information
*   `Bug Track`: Create or update issue in tracking system
*   `Draft Reply`: Generate an appropriate response
*   `Human Review`: Escalate to human agent for approval or handling
*   `Send Reply`: Dispatch the email response

## Step 2: Identify what each step needs to do

For each node in your graph, determine what type of operation it represents and what context it needs to work properly.

### LLM steps

When a step needs to understand, analyze, generate text, or make reasoning decisions:

### Data steps

When a step needs to retrieve information from external sources:

### Action steps

When a step needs to perform an external action:

### User input steps

When a step needs human intervention:

## Step 3: Design your state

State is the shared [memory](https://docs.langchain.com/oss/python/concepts/memory) accessible to all nodes in your agent. Think of it as the notebook your agent uses to keep track of everything it learns and decides as it works through the process.

### What belongs in state?

Ask yourself these questions about each piece of data:

For our email agent, we need to track:

*   The original email and sender info (can’t reconstruct these later)
*   Classification results (needed by multiple later/downstream nodes)
*   Search results and customer data (expensive to re-fetch)
*   The draft response (needs to persist through review)
*   Execution metadata (for debugging and recovery)

### Keep state raw, format prompts on-demand

This separation means:

*   Different nodes can format the same data differently for their needs
*   You can change prompt templates without modifying your state schema
*   Debugging is clearer—you see exactly what data each node received
*   Your agent can evolve without breaking existing state

Let’s define our state:

Notice that the state contains only raw data—no prompt templates, no formatted strings, no instructions. The classification output is stored as a single dictionary, straight from the LLM.

## Step 4: Build your nodes

Now we implement each step as a function. A node in LangGraph is just a Python function that takes the current state and returns updates to it.

### Handle errors appropriately

Different errors need different handling strategies:

| Error Type | Who Fixes It | Strategy | When to Use |
| --- | --- | --- | --- |
| Transient errors (network issues, rate limits) | System (automatic) | Retry policy | Temporary failures that usually resolve on retry |
| LLM-recoverable errors (tool failures, parsing issues) | LLM | Store error in state and loop back | LLM can see the error and adjust its approach |
| User-fixable errors (missing information, unclear instructions) | Human | Pause with `interrupt()` | Need user input to proceed |
| Recoverable failure after retries | Developer (declarative) | `error_handler` | Run a compensation/recovery branch after retry exhaustion |
| Unexpected errors | Developer | Let them bubble up | Unknown issues that need debugging |

*   Transient errors 
*   LLM-recoverable 
*   User-fixable 
*   Unexpected 
*   Saga / compensation 

Add a retry policy to automatically retry network issues and rate limits.Combine with `timeout=` to cap each attempt. See [Fault tolerance](https://docs.langchain.com/oss/python/langgraph/fault-tolerance) for the full lifecycle.

Store the error in state and loop back so the LLM can see what went wrong and try again:

Pause and collect information from the user when needed (like account IDs, order numbers, or clarifications):

Let them bubble up for debugging. Don’t catch what you can’t handle:

After retries are exhausted, run a recovery function that updates state and routes to a compensation branch.See [Fault tolerance](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling) for the full pattern.

To apply the same `retry_policy`, `timeout`, or `error_handler` to every node in a graph without repeating them on each `add_node`, use `StateGraph.set_node_defaults(...)`. Per-node values still take precedence. See [Fault tolerance](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graph-defaults).

### Implementing our email agent nodes

We’ll implement each node as a simple function. Remember: nodes take state, do work, and return updates.

## Step 5: Wire it together

Now we connect our nodes into a working graph. Since our nodes handle their own routing decisions, we only need a few essential edges.To enable [human-in-the-loop](https://docs.langchain.com/oss/python/langgraph/interrupts) with `interrupt()`, we need to compile with a [checkpointer](https://docs.langchain.com/oss/python/langgraph/persistence) to save state between runs:

Graph compilation code

The graph structure is minimal because routing happens inside nodes through [`Command`](https://reference.langchain.com/python/langgraph/types/Command) objects. Each node declares where it can go using type hints like `Command[Literal["node1", "node2"]]`, making the flow explicit and traceable.

### Try out your agent

Let’s run our agent with an urgent billing issue that needs human review:

Testing the agent

The graph pauses when it hits `interrupt()`, saves everything to the checkpointer, and waits. It can resume days later, picking up exactly where it left off. The `thread_id` ensures all state for this conversation is preserved together.

## Summary and next steps

### Key Insights

Building this email agent has shown us the LangGraph way of thinking:

### Advanced considerations

Node granularity trade-offs

You might wonder: why not combine `Read Email` and `Classify Intent` into one node?Or why separate Doc Search from Draft Reply?The answer involves trade-offs between resilience and observability.**The resilience consideration:** LangGraph’s [persistence layer](https://docs.langchain.com/oss/python/langgraph/persistence) creates checkpoints at node boundaries. When a workflow resumes after an interruption or failure, it starts from the beginning of the node where execution stopped. Smaller nodes mean more frequent checkpoints, which means less work to repeat if something goes wrong. If you combine multiple operations into one large node, a failure near the end means re-executing everything from the start of that node.Why we chose this breakdown for the email agent:

*   **Isolation of external services:** Doc Search and Bug Track are separate nodes because they call external APIs. If the search service is slow or fails, we want to isolate that from the LLM calls. We can add retry policies to these specific nodes without affecting others.
*   **Intermediate visibility:** Having `Classify Intent` as its own node lets us inspect what the LLM decided before taking action. This is valuable for debugging and monitoring—you can see exactly when and why the agent routes to human review.
*   **Different failure modes:** LLM calls, database lookups, and email sending have different retry strategies. Separate nodes let you configure these independently.
*   **Reusability and testing:** Smaller nodes are easier to test in isolation and reuse in other workflows.

A different valid approach: You could combine `Read Email` and `Classify Intent` into a single node. You’d lose the ability to inspect the raw email before classification and would repeat both operations on any failure in that node. For most applications, the observability and debugging benefits of separate nodes are worth the trade-off.Application-level concerns: The caching discussion in Step 2 (whether to cache search results) is an application-level decision, not a LangGraph framework feature. You implement caching within your node functions based on your specific requirements—LangGraph doesn’t prescribe this.Performance considerations: More nodes doesn’t mean slower execution. LangGraph writes checkpoints in the background by default ([async durability mode](https://docs.langchain.com/oss/python/langgraph/checkpointers#durability-modes)), so your graph continues running without waiting for checkpoints to complete. This means you get frequent checkpoints with minimal performance impact. You can adjust this behavior if needed—use `"exit"` mode to checkpoint only at completion, or `"sync"` mode to block execution until each checkpoint is written.

### Where to go from here

This was an introduction to thinking about building agents with LangGraph. You can extend this foundation with:

* * *