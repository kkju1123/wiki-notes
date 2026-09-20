---
title: Graph execution - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/frontend/graph-execution
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:57:47.554819+00:00'
---

LangGraph agents aren’t black boxes. Every graph is composed of **named nodes** that execute in sequence or in parallel: classify, research, analyze, synthesize. Graph execution cards make this pipeline visible by rendering a card for each node, showing its status, streaming its content in real time, and tracking completion across the entire workflow. Users see exactly what the agent is doing, which step it’s on, and what each step produced.This pattern is especially useful for production agents because it turns graph structure into product UX. Instead of treating the run as a single assistant response, you can expose the same checkpoints, node names, state keys, and stream metadata that LangGraph uses internally.

## How graph nodes map to UI cards

A LangGraph graph defines a series of nodes, each responsible for a specific task. For example, a research pipeline might have:

1.   **Classify**: categorize the user’s query
2.   **Research**: gather relevant information
3.   **Analyze**: draw conclusions from the research
4.   **Synthesize**: produce a final, polished response

Each node writes its output to a specific key in the graph’s state. On the frontend, you don’t need to hardcode that mapping as [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) discovers each node as it runs via `stream.subgraphs` and exposes a [`SubgraphDiscoverySnapshot`](https://reference.langchain.com/javascript/langchain-react/SubgraphDiscoverySnapshot) for every observed step:

Use `node.nodeName` for labels in the progress bar and card headers. Pass each snapshot to `useMessages(stream, node)` to render node-scoped streaming content without coupling the UI to graph state key names.This mapping becomes the contract between your graph and your UI. Backend authors can add, rename, or reorder nodes intentionally, while frontend authors decide how each state key should be visualized: a status badge, markdown panel, table, chart, trace view, or approval card.

## Setting up `useStream`

Wire up [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) as usual. The key properties you’ll use are `messages` (for the conversation) and `subgraphs` (for the graph nodes discovered in the current run). Pass each discovered subgraph snapshot to a selector to read the messages scoped to that node.

## Routing streaming tokens to nodes

As the graph streams, each discovered subgraph snapshot identifies the node it belongs to. Pass that snapshot to a selector hook or composable to read the messages scoped to that node:

The first mounted selector opens a scoped subscription for that node namespace. When the node card unmounts, the subscription is released automatically.

## Determining node status

Each discovered node carries its current status. Use `node.status` directly; the discovery snapshot reports `"pending"`, `"running"`, `"complete"`, or `"error"`:

## Building the pipeline progress bar

A horizontal progress bar at the top gives users a bird’s-eye view of the entire pipeline. Each step is a labeled segment that fills in as nodes complete:

## Building collapsible NodeCard components

Each node gets its own card that shows the status badge, content (streaming or final), and a collapsible body for long outputs:

## Streaming vs. completed content

The node card reads scoped messages for both streaming and final content. This avoids assuming that a graph node name matches the state key it writes to (for example, `do_research` writes to `research` in the playground graph):

| Source | When to use |
| --- | --- |
| `useMessages(stream, node)` | Render node-scoped streaming and final messages |
| `stream.values` | Read whole-graph state such as the final `synthesis` field, using the actual state key |

The pattern is: show the most recent scoped AI message in the node card, and use `stream.values` only when you intentionally need a graph state field.Because scoped messages are tied to the producing node, the UI can support parallel graph paths without guessing from message order. Each card updates from the stream events that belong to its node, and completed values remain available through `stream.values`.

## Putting it all together

Here’s the full card list that combines routing, status detection, and card rendering:

## Use cases

Graph execution cards work well for any multi-step pipeline where visibility matters:

*   **Research pipelines**: classify → gather sources → analyze → synthesize a report
*   **Content generation**: outline → draft → fact-check → edit → publish
*   **Data processing**: ingest → validate → transform → aggregate → export
*   **Code generation**: understand requirements → plan architecture → write code → review → test
*   **Decision workflows**: gather context → evaluate options → score alternatives → recommend

## Handling dynamic pipelines

Not all graphs have a fixed set of nodes. Some pipelines add or skip nodes based on the input. The discovery map contains only nodes observed for the current thread:

This ensures your UI only shows cards for nodes that are relevant to the current execution, avoiding empty placeholder cards.

## Best practices

*   **Discover nodes from the stream**. Render cards from `stream.subgraphs` rather than hardcoding expected nodes; conditional or skipped steps won’t appear until they run.
*   **Treat state keys as UI contracts**. Decide which graph outputs should be stable enough for the frontend to render, and keep those keys documented next to the graph definition.
*   **Use scoped messages for node cards**. They work while a node is streaming and after it completes, without coupling UI cards to state key names.
*   **Auto-collapse completed nodes**.In long pipelines, auto-collapse finished cards so users can focus on the currently active step.
*   **Show estimated timing**. If you have historical data on how long each node takes, display a time estimate to set user expectations.
*   **Add a global progress indicator**. Complement per-node cards with an overall progress bar (e.g., “Step 2 of 4”) at the top of the pipeline view.
*   **Handle errors per node**. If a node fails, show the error in its card without collapsing the entire pipeline. Other nodes may still complete successfully.

* * *