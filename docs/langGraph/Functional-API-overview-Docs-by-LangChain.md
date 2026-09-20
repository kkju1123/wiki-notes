---
title: Functional API overview - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/functional-api
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:58:20.596537+00:00'
---

The **Functional API** allows you to add LangGraph’s key features ([persistence](https://docs.langchain.com/oss/python/langgraph/persistence), [memory](https://docs.langchain.com/oss/python/langgraph/add-memory), [human-in-the-loop](https://docs.langchain.com/oss/python/langgraph/interrupts), and [streaming](https://docs.langchain.com/oss/python/langgraph/streaming)) to your applications with minimal changes to your existing code.It is designed to integrate these features into existing code that may use standard language primitives for branching and control flow, such as `if` statements, `for` loops, and function calls. Unlike many data orchestration frameworks that require restructuring code into an explicit pipeline or DAG, the Functional API allows you to incorporate these capabilities without enforcing a rigid execution model.The Functional API uses two key building blocks:

*   **`@entrypoint`**: Marks a function as the starting point of a workflow, encapsulating logic and managing execution flow, including handling long-running tasks and interrupts.
*   **[`@task`](https://reference.langchain.com/python/langgraph/func/task)**: Represents a discrete unit of work, such as an API call or data processing step, that can be executed asynchronously within an entrypoint. Tasks return a future-like object that can be awaited or resolved synchronously.

This provides a minimal abstraction for building workflows with state management and streaming.

## Functional API vs. Graph API

For users who prefer a more declarative approach, LangGraph’s [Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api) allows you to define workflows using a Graph paradigm. Both APIs share the same underlying runtime, so you can use them together in the same application.Here are some key differences:

*   **Control flow**: The Functional API does not require thinking about graph structure. You can use standard Python constructs to define workflows. This will usually trim the amount of code you need to write.
*   **Short-term memory**: The **GraphAPI** requires declaring a [**State**](https://docs.langchain.com/oss/python/langgraph/graph-api#state) and may require defining [**reducers**](https://docs.langchain.com/oss/python/langgraph/graph-api#reducers) to manage updates to the graph state. `@entrypoint` and `@tasks` do not require explicit state management as their state is scoped to the function and is not shared across functions.
*   **Checkpointing**: Both APIs generate and use checkpoints. In the **Graph API** a new checkpoint is generated after every [superstep](https://docs.langchain.com/oss/python/langgraph/graph-api). In the **Functional API**, when tasks are executed, their results are saved to an existing checkpoint associated with the given entrypoint instead of creating a new checkpoint.
*   **Visualization**: The Graph API makes it easy to visualize the workflow as a graph which can be useful for debugging, understanding the workflow, and sharing with others. The Functional API does not support visualization as the graph is dynamically generated during runtime.

## Example

Below we demonstrate a simple application that writes an essay and [interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) to request human review.

Detailed Explanation

This workflow will write an essay about the topic “cat” and then pause to get a review from a human. The workflow can be interrupted for an indefinite amount of time until a review is provided.When the workflow is resumed, it executes from the very start, but because the result of the `writeEssay` task was already saved, the task result will be loaded from the checkpoint instead of being recomputed.

An essay has been written and is ready for review. Once the review is provided, we can resume the workflow:

The workflow has been completed and the review has been added to the essay.

## Entrypoint

The [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) decorator can be used to create a workflow from a function. It encapsulates workflow logic and manages execution flow, including handling _long-running tasks_ and [interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts).

### Definition

An **entrypoint** is defined by decorating a function with the `@entrypoint` decorator.The function **must accept a single positional argument**, which serves as the workflow input. If you need to pass multiple pieces of data, use a dictionary as the input type for the first argument.Decorating a function with an `entrypoint` produces a [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream) instance which helps to manage the execution of the workflow (e.g., handles streaming, resumption, and checkpointing).You will usually want to pass a **checkpointer** to the `@entrypoint` decorator to enable persistence and use features like **human-in-the-loop**.

*   Sync

*   Async

### Injectable parameters

When declaring an `entrypoint`, you can request access to additional parameters that will be injected automatically at runtime. These parameters include:

| Parameter | Description |
| --- | --- |
| **previous** | Access the state associated with the previous `checkpoint` for the given thread. See [short-term-memory](https://docs.langchain.com/oss/python/langgraph/functional-api#short-term-memory). |
| **store** | An instance of [BaseStore][langgraph.store.base.BaseStore]. Useful for [long-term memory](https://docs.langchain.com/oss/python/langgraph/use-functional-api#long-term-memory). |
| **writer** | Use to access the StreamWriter when working with Async Python < 3.11. See [streaming with functional API for details](https://docs.langchain.com/oss/python/langgraph/use-functional-api#streaming). |
| **config** | For accessing run time configuration. See [RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) for information. |

Requesting Injectable Parameters

### Executing

Using the [`@entrypoint`](https://docs.langchain.com/oss/python/langgraph/functional-api#entrypoint) yields a [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream) object that can be executed using the `invoke`, `ainvoke`, `stream`, and `astream` methods.

*   Invoke

*   Async Invoke

*   Stream

*   Async Stream

### Resuming

Resuming an execution after an [interrupt](https://reference.langchain.com/python/langgraph/types/interrupt) can be done by passing a **resume** value to the [`Command`](https://reference.langchain.com/python/langgraph/types/Command) primitive.

*   Invoke

*   Async Invoke

*   Stream

*   Async Stream

**Resuming after an error**To resume after an error, run the `entrypoint` with a `None` and the same **thread id** (config).This assumes that the underlying **error** has been resolved and execution can proceed successfully.

*   Invoke

*   Async Invoke

*   Stream

*   Async Stream

### Short-term memory

When an `entrypoint` is defined with a `checkpointer`, it stores information between successive invocations on the same **thread id** in [checkpoints](https://docs.langchain.com/oss/python/langgraph/checkpointers#checkpoints).This allows accessing the state from the previous invocation using the `previous` parameter.By default, the `previous` parameter is the return value of the previous invocation.

#### `entrypoint.final`

[`entrypoint.final`](https://reference.langchain.com/python/langgraph/func/entrypoint/final) is a special primitive that can be returned from an entrypoint and allows **decoupling** the value that is **saved in the checkpoint** from the **return value of the entrypoint**.The first value is the return value of the entrypoint, and the second value is the value that will be saved in the checkpoint. The type annotation is `entrypoint.final[return_type, save_type]`.

## Task

A **task** represents a discrete unit of work, such as an API call or data processing step. It has two key characteristics:

*   **Asynchronous Execution**: Tasks are designed to be executed asynchronously, allowing multiple operations to run concurrently without blocking.
*   **Checkpointing**: Task results are saved to a checkpoint, enabling resumption of the workflow from the last saved state. (See [persistence](https://docs.langchain.com/oss/python/langgraph/persistence) for more details).

### Definition

Tasks are defined using the `@task` decorator, which wraps a regular Python function.

### Execution

**Tasks** can only be called from within an **entrypoint**, another **task**, or a [state graph node](https://docs.langchain.com/oss/python/langgraph/graph-api#nodes).Tasks _cannot_ be called directly from the main application code.When you call a **task**, it returns _immediately_ with a future object. A future is a placeholder for a result that will be available later.To obtain the result of a **task**, you can either wait for it synchronously (using `result()`) or await it asynchronously (using `await`).

*   Synchronous Invocation

*   Asynchronous Invocation

## When to use a task

**Tasks** are useful in the following scenarios:

*   **Checkpointing**: When you need to save the result of a long-running operation to a checkpoint, so you don’t need to recompute it when resuming the workflow.
*   **Human-in-the-loop**: If you’re building a workflow that requires human intervention, you MUST use **tasks** to encapsulate any randomness (e.g., API calls) to ensure that the workflow can be resumed correctly. See the [determinism](https://docs.langchain.com/oss/python/langgraph/functional-api#determinism) section for more details.
*   **Parallel Execution**: For I/O-bound tasks, **tasks** enable parallel execution, allowing multiple operations to run concurrently without blocking (e.g., calling multiple APIs).
*   **Observability**: Wrapping operations in **tasks** provides a way to track the progress of the workflow and monitor the execution of individual operations using [LangSmith](https://docs.langchain.com/langsmith/observability).
*   **Retryable Work**: When work needs to be retried to handle failures or inconsistencies, **tasks** provide a way to encapsulate and manage the retry logic.

## Serialization

There are two key aspects to serialization in LangGraph:

1.   `entrypoint` inputs and outputs must be JSON-serializable.
2.   `task` outputs must be JSON-serializable.

These requirements are necessary for enabling checkpointing and workflow resumption. Use python primitives like dictionaries, lists, strings, numbers, and booleans to ensure that your inputs and outputs are serializable.Serialization ensures that workflow state, such as task results and intermediate values, can be reliably saved and restored. This is critical for enabling human-in-the-loop interactions, fault tolerance, and parallel execution.Providing non-serializable inputs or outputs will result in a runtime error when a workflow is configured with a checkpointer.

## Determinism

When you resume a workflow run, the code does **NOT** resume from the **same line of code** where execution stopped. Execution returns to a checkpoint boundary, and the workflow **replays** forward until it reaches the pause again.For the Functional API, replay starts at the beginning of the **entrypoint** while LangGraph restores completed [**task**](https://docs.langchain.com/oss/python/langgraph/functional-api#task) and [**subgraph**](https://docs.langchain.com/oss/python/langgraph/use-subgraphs) results from the checkpointer instead of recomputing them. That preserves the recorded order of steps across pauses, including for long-running or non-deterministic **task** outputs.To use features like **human-in-the-loop**, you must place non-deterministic work (for example, random values) and side effects (for example, file writes or API calls) in [**tasks**](https://docs.langchain.com/oss/python/langgraph/functional-api#task).Different runs of a workflow can produce different results, but resuming a **specific** thread should replay the same persisted **task** and **subgraph** results.To ensure that your workflow is deterministic and can be consistently replayed, follow these guidelines:

*   **Avoid repeating work**: In an **entrypoint**, if you chain several side effects (for example, logging, file writes, or network calls), give each its own **task** so resume restores their outputs from the checkpointer instead of running them again.
*   **Encapsulate non-deterministic operations**: Keep values that can change between attempts (for example, random numbers or wall-clock reads) inside **tasks**, so replay lines up with what was checkpointed.
*   **Use idempotent operations**: For partial task failures and retries, see [Idempotency](https://docs.langchain.com/oss/python/langgraph/functional-api#idempotency).

## Idempotency

Idempotency ensures that running the same operation multiple times produces the same result. This helps prevent duplicate API calls and redundant processing if a step is rerun due to a failure. Always place API calls inside **tasks** functions for checkpointing, and design them to be idempotent in case of re-execution. This is particularly important for operations that result in data writes. When a workflow resumes, LangGraph replays completed **task** results from the checkpoint. A **task** that started but did not finish may run again on that resume, so design side effects to be idempotent. Use idempotency keys or verify existing results to avoid unintended duplication.

## Common pitfalls

### Handling side effects

Encapsulate side effects (e.g., writing to a file, sending an email) in tasks to ensure they are not executed multiple times when resuming a workflow.

*   Incorrect

*   Correct

In this example, a side effect (writing to a file) is directly included in the workflow, so it will be executed a second time when resuming the workflow.

In this example, the side effect is encapsulated in a task, ensuring consistent execution upon resumption.

### Non-deterministic control flow

Operations that might give different results each time (like getting current time or random numbers) should be encapsulated in tasks to ensure that on resume, the same result is returned.

*   In a task: Get random number (5) → interrupt → resume → (returns 5 again) → …
*   Not in a task: Get random number (5) → interrupt → resume → get new random number (7) → …

This is especially important when using **human-in-the-loop** workflows with multiple interrupt calls. LangGraph keeps a list of resume values for each task/entrypoint. When an interrupt is encountered, it’s matched with the corresponding resume value. This matching is strictly **index-based**, so the order of the resume values should match the order of the interrupts.If order of execution is not maintained when resuming, one [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) call may be matched with the wrong `resume` value, leading to incorrect results.Please read the section on [determinism](https://docs.langchain.com/oss/python/langgraph/functional-api#determinism) for more details.

*   Incorrect

*   Correct

In this example, the workflow uses the current time to determine which task to execute. This is non-deterministic because the result of the workflow depends on the time at which it is executed.

In this example, the workflow uses the input `t0` to determine which task to execute. This is deterministic because the result of the workflow depends only on the input.

## Learn more

*   [How to use the Functional API](https://docs.langchain.com/oss/python/langgraph/use-functional-api)
*   [Graph API conceptual overview](https://docs.langchain.com/oss/python/langgraph/graph-api)
*   [Choosing between Graph API and Functional API](https://docs.langchain.com/oss/python/langgraph/choosing-apis)

* * *