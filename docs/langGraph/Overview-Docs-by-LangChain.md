---
title: Overview - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/frontend/overview
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:57:40.279240+00:00'
---

Build frontends that visualize LangGraph pipelines in real time. These patterns show how to render multi-step graph execution with per-node status and streaming content from custom `StateGraph` workflows.LangGraph’s frontend advantage is that the UI can follow the same structure as the graph. Nodes, state keys, checkpoints, interrupts, subgraphs, and streamed messages are all visible runtime concepts, so you can build interfaces that explain what the system is doing instead of hiding execution behind one assistant message.

## Architecture

LangGraph graphs are composed of named nodes connected by edges. Each node executes a step (classify, research, analyze, synthesize) and writes output to a specific state key. On the frontend, the SDK stream handle provides reactive access to node outputs, streaming tokens, and discovered subgraphs so you can map each node to a UI card.

On the frontend, [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) exposes `stream.subgraphs` for graph-node discovery and selector helpers such as `useMessages(stream, node)` for node-scoped streaming content. `stream.values` still holds the full graph state when you need fields such as the final `synthesis`. Angular uses the same stream API shape through [`injectStream`](https://reference.langchain.com/javascript/langchain-angular/injectStream).

## What makes this different from a chat stream

Custom graphs often power product workflows: research pipelines, approval flows, data pipelines, data enrichment, code review, planning, and multi-step analysis. The frontend SDK lets you render these workflows using graph-native signals:

| Runtime concept | Frontend UX |
| --- | --- |
| **Named nodes** | One card, timeline step, or status badge per graph node. |
| **State keys** | Dedicated UI regions for typed outputs such as classification, sources, analysis, and final synthesis. |
| **Streaming metadata** | Route partial messages to the node that produced them. |
| **Checkpoints** | Inspect or resume from prior graph states for debugging and auditability. |
| **Interrupts** | Pause a node for human input, approval, or correction, then continue. |
| **Subgraphs** | Reveal nested execution only when the user needs more detail. |

Because the SDK exposes these concepts directly, you can scale from a simple chat panel to a full workflow debugger without changing the backend protocol.

## Patterns

The [LangChain frontend patterns](https://docs.langchain.com/oss/python/langchain/frontend/overview)—markdown messages, tool calling, human-in-the-loop, resumable streams, and time travel—work with any LangGraph graph. The stream API provides the same core data model whether you use `createAgent`, `createDeepAgent`, or a custom `StateGraph`.

* * *