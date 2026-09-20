---
title: LangGraph runtime - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/pregel
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:58:25.466029+00:00'
---

[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) implements LangGraph’s runtime, managing the execution of LangGraph applications.Compiling a [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) or creating an [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) produces a [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) instance that can be invoked with input.This guide explains the runtime at a high level and provides instructions for directly implementing applications with Pregel.

> **Note:** The [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) runtime is named after [Google’s Pregel algorithm](https://research.google/pubs/pub37252/), which describes an efficient method for large-scale parallel computation using graphs.

## Overview

In LangGraph, Pregel combines [**actors**](https://en.wikipedia.org/wiki/Actor_model) and **channels** into a single application. **Actors** read data from channels and write data to channels. Pregel organizes the execution of the application into multiple steps, following the **Pregel Algorithm**/**Bulk Synchronous Parallel** model.Each step consists of three phases:

*   **Plan**: Determine which **actors** to execute in this step. For example, in the first step, select the **actors** that subscribe to the special **input** channels; in subsequent steps, select the **actors** that subscribe to channels updated in the previous step.
*   **Execution**: Execute all selected **actors** in parallel, until all complete, or one fails, or a timeout is reached. During this phase, channel updates are invisible to actors until the next step.
*   **Update**: Update the channels with the values written by the **actors** in this step.

Repeat until no **actors** are selected for execution, or a maximum number of steps is reached.

## Actors

An **actor** is a `PregelNode`. It subscribes to channels, reads data from them, and writes data to them. It can be thought of as an **actor** in the Pregel algorithm. `PregelNodes` implement LangChain’s Runnable interface.

## Channels

Channels are used to communicate between actors (PregelNodes). Each channel has a value type, an update type, and an update function—which takes a sequence of updates and modifies the stored value. Channels can be used to send data from one chain to another, or to send data from a chain to itself in a future step.

### LastValue

[`LastValue`](https://reference.langchain.com/python/langgraph/channels/last_value/LastValue) is the default channel type. It stores the last value written to it, overwriting any previous value. Use it for input and output values, or for passing data from one step to the next.

### Topic

[`Topic`](https://reference.langchain.com/python/langgraph/channels/topic/Topic) is a configurable PubSub channel useful for sending multiple values between actors or accumulating output across steps. It can be configured to deduplicate values or to accumulate all values written during a run.

### BinaryOperatorAggregate

[`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/channels/binop/BinaryOperatorAggregate) stores a persistent value that is updated by applying a binary operator to the current value and each new update. Use it to compute running aggregates across steps.

### DeltaChannel

[`DeltaChannel`](https://reference.langchain.com/python/langgraph/channels/delta/DeltaChannel) stores only the incremental delta at each step rather than the full accumulated value. This is most useful for channels that are written frequently and accumulate large values over time—for example, a conversation message list in a long-running thread. Without delta storage, the full list is re-serialized into every checkpoint; with `DeltaChannel`, only the new messages written at each step are stored.

Use `DeltaChannel` in an `Annotated` type annotation the same way you would use a plain reducer:

#### Bulk reducer requirement

The `reducer` passed to `DeltaChannel` is a **bulk reducer**: it receives the current state and a _sequence_ of all writes from the current step in a single call—not pairwise like a standard reducer. This differs from the per-key reducers used with `Annotated` in a `StateGraph`, where the reducer is called once per update.

Here are bulk reducers for the two most common cases:

Both are associative: applying batches one at a time produces the same result as applying them together.

#### Use snapshot_frequency for bounded read latency

Without snapshots, reading a `DeltaChannel` value requires replaying the full write history—O(N) for a thread with N steps. Setting `snapshot_frequency=K` writes a full snapshot every K pregel steps, bounding read depth to at most K steps:

Higher values of `snapshot_frequency` reduce storage overhead but increase read latency. Lower values bound latency more tightly at the cost of larger checkpoints. `None` (the default) skips snapshots entirely—appropriate when reads are rare or threads are short.

#### Version compatibility and rollbacks

## Examples

While most users will interact with Pregel through the [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) API or the [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) decorator, it is possible to interact with Pregel directly.Below are a few different examples to give you a sense of the Pregel API.

*   Single node

*   Multiple nodes

*   Topic

*   BinaryOperatorAggregate

*   Cycle

This example demonstrates how to introduce a cycle in the graph, by having a chain write to a channel it subscribes to. Execution will continue until a `None` value is written to the channel.

## High-level API

LangGraph provides two high-level APIs for creating a Pregel application: the [StateGraph (Graph API)](https://docs.langchain.com/oss/python/langgraph/graph-api) and the [Functional API](https://docs.langchain.com/oss/python/langgraph/functional-api).

*   StateGraph (Graph API)

*   Functional API

The [StateGraph (Graph API)](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) is a higher-level abstraction that simplifies the creation of Pregel applications. It allows you to define a graph of nodes and edges. When you compile the graph, the StateGraph API automatically creates the Pregel application for you.

The compiled Pregel instance will be associated with a list of nodes and channels. You can inspect the nodes and channels by printing them.

You will see something like this:

You should see something like this

In the [Functional API](https://docs.langchain.com/oss/python/langgraph/functional-api), you can use an [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) to create a Pregel application. The `entrypoint` decorator allows you to define a function that takes input and returns output.

* * *