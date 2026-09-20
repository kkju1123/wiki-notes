---
title: Stores - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/stores
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:55:28.058867+00:00'
---

Stores let agents persist information across threads, including user preferences, accumulated knowledge, and facts that should survive beyond a single conversation. Unlike [checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers), which save the full graph state scoped to one thread, stores hold arbitrary key-value data accessible from any thread.![Image 1: Model of shared state](https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=354526fb48c5eb11b4b2684a2df40d6c)

## Basic usage

The following code snippet shows the [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) in isolation without using LangGraph:

Memories are namespaced by a `tuple`, which is `(<user_id>, "memories")` in the following example. The namespace can be any length and represent anything, does not have to be user specific.

Use the `store.put` method to save memories to the namespace in the store. Specify the namespace, as defined above, and a key-value pair for the memory: the key is simply a unique identifier for the memory (`memory_id`) and the value (a dictionary) is the memory itself.

Read out memories from your namespace using the `store.search` method, which returns memories for a given user as a list, up to the `limit` argument (default `10`). With `InMemoryStore`, items are returned in insertion order, so the most recent memory is last in the list; other backends may order memories differently (see [Listing items in a namespace](https://docs.langchain.com/oss/python/langgraph/stores#listing-items-in-a-namespace)).

Each memory type is a Python class ([`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item)) with certain attributes. We can access it as a dictionary by converting with `.dict`.The attributes it has are:

*   `value`: The value (itself a dictionary) of this memory
*   `key`: A unique key for this memory in this namespace
*   `namespace`: A tuple of strings, the namespace of this memory type
*   `created_at`: Timestamp for when this memory was created
*   `updated_at`: Timestamp for when this memory was updated

## Listing items in a namespace

Calling [`store.search`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.search) (or the async [`store.asearch`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.asearch)) with no `query` and no `filter` returns the items stored under `namespace_prefix`, up to `limit`. Use this to enumerate everything in a namespace when you don’t need semantic ranking.

Three behaviors to keep in mind:

*   **`namespace_prefix` matches by prefix, not exactly.**`("alice",)` also returns items under `("alice", "memories")`, `("alice", "preferences")`, and so on. To restrict to a single level, pass the full namespace or filter the returned items client-side on `item.namespace`.
*   **Results past `limit` are silently truncated.** There is no overflow signal—set `limit` above your expected maximum, or paginate with `offset`.
*   **Default ordering depends on the store backend.**`PostgresStore` and `AsyncPostgresStore` return results ordered by `updated_at` descending (most recently updated first). `InMemoryStore` returns results in insertion order (most recently inserted last). Do not rely on a specific order across implementations; sort client-side on `item.updated_at` if order matters.

To page through a large namespace:

To discover which namespaces exist (for example, to iterate over every user before listing their memories), use [`store.list_namespaces`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.list_namespaces) or [`store.alist_namespaces`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.alist_namespaces):

## Semantic search

Beyond simple retrieval, the store also supports semantic search, allowing you to find memories based on meaning rather than exact matches. To enable this, configure the store with an embedding model:

Now when searching, you can use natural language queries to find relevant memories:

You can control which parts of your memories get embedded by configuring the `fields` parameter or by specifying the `index` parameter when storing memories:

## Using in LangGraph

The store works hand-in-hand with the checkpointer: the checkpointer saves state to threads, as discussed above, and the store allows you to store arbitrary information for access _across_ threads. Compile the graph with both the checkpointer and the store as follows.

Then invoke the graph with a `thread_id`, as before, and also with a `user_id`, which serves as the namespace for memories for this particular user as before.

You can access the store and the `user_id` from _any node_ by using the `Runtime` object. The `Runtime` is automatically injected by LangGraph when you add it as a parameter to your node function. You can use it to save memories:

You can also access the store from any node and use the `store.search` method to get memories. Memories are returned as a list of objects that can be converted to a dictionary.

You access the memories and use them in model calls.

If you create a new thread, you can still access the same memories so long as the `user_id` is the same.

When you use LangSmith locally (e.g., in [Studio](https://docs.langchain.com/langsmith/studio)) or [hosted](https://docs.langchain.com/langsmith/platform-setup), the base store is available to use by default and you do not need to specify it during graph compilation. To enable semantic search, however, you **do** need to configure the indexing settings in your `langgraph.json` file. For example:

See the [deployment guide](https://docs.langchain.com/langsmith/semantic-search) for more details and configuration options.

## Build a custom store

To use a storage backend other than the built-in implementations, subclass [BaseStore](https://reference.langchain.com/python/langchain-core/stores/BaseStore) and implement its required methods. The built-in [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) is the simplest reference implementation.

### Base contract

All five async methods are required. Sync counterparts (`put`, `get`, `delete`, `search`, `list_namespaces`) are optional but recommended for compatibility with sync graph execution.

| Method | Description |
| --- | --- |
| `aput(namespace, key, value, index=None)` | Store or overwrite a single item |
| `aget(namespace, key)` | Retrieve a single item by key; return `None` if missing |
| `adelete(namespace, key)` | Delete a single item |
| `asearch(namespace_prefix, *, query=None, filter=None, limit=10, offset=0)` | Search items under a namespace prefix; optionally by semantic query |
| `alist_namespaces(*, prefix=None, suffix=None, max_depth=None, limit=100, offset=0)` | List namespaces matching a prefix/suffix pattern |

Look up exact signatures before implementing:

### Namespace design

Namespaces are tuples of strings, e.g. `("user_id", "memories")`. Store implementations must support:

*   **Prefix matching**: `asearch(("alice",))` returns items under `("alice",)`, `("alice", "memories")`, and any other sub-namespace.
*   **Exact key lookup**: `aget(("alice", "memories"), "some-key")` must be O(1) or close to it.

For SQL backends, a common schema:

### Serialization

Store values are plain Python dicts — no special serializer is required. Serialize with `json.dumps` / `json.loads` or a JSONB column directly. Do not store raw Python objects that are not JSON-serializable.

### Semantic search support

If your backend supports vector search, implement the `query` parameter on `asearch`:

*   Accept a `query: str | None` argument.
*   When `query` is not `None`, embed it and rank results by cosine similarity.
*   Results should include a `score` field on each `Item` when `query` is provided.

If your backend does not support vector search, raise `NotImplementedError` when `query` is passed.

### Testing

There is currently no conformance suite for custom stores. Test against [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) as the reference:

### Next steps

*   [Add a custom store to Agent Server](https://docs.langchain.com/langsmith/custom-store) — deploying your implementation
*   [Checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers) — thread-scoped state persistence

* * *