---
title: Message Batches API：异步批量处理大模型请求
url: https://platform.claude.com/docs/en/build-with-claude/batch-processing
source_type: web
folder: claude
author: null
tags:
- Message Batches API
- 异步批处理
- 成本优化
- 大规模推理
- Anthropic Claude
summary: 用 Message Batches API 异步批量处理 Claude 请求，成本降低 50%、吞吐提升，适合无需即时响应的离线任务。
fetched_at: '2026-09-20T06:10:59.106515+00:00'
---

Process large volumes of Messages requests asynchronously with the Message Batches API, cutting costs by 50% and increasing throughput.

Batch processing is a powerful approach for handling large volumes of requests efficiently. Instead of processing requests one at a time with immediate responses, batch processing allows you to submit multiple requests together for asynchronous processing. This pattern is particularly useful when:

*   You need to process large volumes of data
*   Immediate responses are not required
*   You want to optimize for cost efficiency
*   You're running large-scale evaluations or analyses

The Message Batches API is Anthropic's first implementation of this pattern.

## Message Batches API

The Message Batches API is a powerful, cost-effective way to asynchronously process large volumes of [Messages](https://platform.claude.com/docs/en/api/messages/create) requests. This approach is well-suited to tasks that do not require immediate responses, with most batches finishing in less than 1 hour while reducing costs by 50% and increasing throughput.

You can [explore the API reference directly](https://platform.claude.com/docs/en/api/messages/batches/create), in addition to this guide.

## How the Message Batches API works

When you send a request to the Message Batches API:

1.   The system creates a new Message Batch with the provided Messages requests.
2.   The batch is then processed asynchronously, with each request handled independently.
3.   You can poll for the status of the batch and retrieve results when processing has ended for all requests.

This is especially useful for bulk operations that don't require immediate results, such as:

*   Large-scale evaluations: Process thousands of test cases efficiently.
*   Content moderation: Analyze large volumes of user-generated content asynchronously.
*   Data analysis: Generate insights or summaries for large datasets.
*   Bulk content generation: Create large amounts of text for various purposes (for example, product descriptions, article summaries).

### Batch limitations

*   A Message Batch is limited to either 100,000 Message requests or 256 MB in size, whichever is reached first.
*   The system processes each batch as fast as possible, with most batches completing within 1 hour. You can access batch results when all messages have completed or after 24 hours, whichever comes first. Batches expire if processing does not complete within 24 hours.
*   Batch results are available for 29 days after creation. After that, you may still view the Batch, but its results will no longer be available for download.
*   Batches are scoped to a [Workspace](https://platform.claude.com/settings/workspaces). You may view all batches (and their results) that were created within the Workspace your request runs in.
*   Rate limits apply to both Batches API HTTP requests and the number of requests within a batch waiting to be processed. See [Message Batches API rate limits](https://platform.claude.com/docs/en/api/rate-limits#message-batches-api). Additionally, processing may be slowed down based on current demand and your request volume. In that case, you may see more requests expiring after 24 hours.
*   Because of high throughput and concurrent processing, batches may go slightly over your Workspace's configured [spend limit](https://platform.claude.com/settings/billing).
*   Each batched request must have `max_tokens` of at least `1`. `max_tokens: 0` ([cache pre-warming](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#pre-warming-the-cache)) is not supported inside a batch, because an ephemeral cache entry written during batch processing would likely expire before the follow-up request runs.

### Supported models

All [active models](https://platform.claude.com/docs/en/models/overview) support the Message Batches API.

### What can be batched

Almost any request you can make to the Messages API can be included in a batch. This includes:

*   Vision
*   Tool use, including all [server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) (web search, web fetch, code execution, MCP connectors, advisor, and tool search)
*   System messages
*   Multi-turn conversations
*   Extended thinking
*   Most beta features

Because each request in the batch is processed independently, you can mix different types of requests within a single batch.

A small number of Messages API parameters are **not** supported in batch requests. Including any of these returns a validation error:

| Parameter | Why |
| --- | --- |
| `stream: true` | Batch results come back as a single file, not a stream. |
| `speed` ([Fast mode](https://platform.claude.com/docs/en/build-with-claude/fast-mode)) | Fast mode tunes synchronous latency, which doesn't apply to asynchronous batch processing. |
| `max_tokens: 0` | See [Batch limitations](https://platform.claude.com/docs/en/build-with-claude/batch-processing#batch-limitations). |

## Pricing

The Batches API offers significant cost savings. All usage is charged at 50% of the standard API prices.

| Model | Batch input | Batch output |
| --- | --- | --- |
| Claude Fable 5.1 | $5 / MTok | $25 / MTok |
| Claude Mythos 5.1 ([limited availability](https://anthropic.com/glasswing)) | $5 / MTok | $25 / MTok |
| Claude Fable 5 | $5 / MTok | $25 / MTok |
| Claude Mythos 5 ([limited availability](https://anthropic.com/glasswing)) | $5 / MTok | $25 / MTok |
| Claude Opus 5 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.8 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.7 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.6 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.5 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.1 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | $7.50 / MTok | $37.50 / MTok |
| Claude Opus 4 ([retired, except on Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | $7.50 / MTok | $37.50 / MTok |
| Claude Sonnet 5 | $1 / MTok | $5 / MTok |
| Claude Sonnet 4.6 | $1.50 / MTok | $7.50 / MTok |
| Claude Sonnet 4.5 | $1.50 / MTok | $7.50 / MTok |
| Claude Sonnet 4 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | $1.50 / MTok | $7.50 / MTok |
| Claude Haiku 4.5 | $0.50 / MTok | $2.50 / MTok |
| Claude Haiku 3.5 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | $0.40 / MTok | $2 / MTok |

## How to use the Message Batches API

### Prepare and create your batch

A Message Batch is composed of a list of requests to create a Message. The shape of an individual request comprises:

*   A unique `custom_id` for identifying the Messages request. Must be 1 to 64 characters and contain only alphanumeric characters, hyphens, and underscores (matching `^[a-zA-Z0-9_-]{1,64}$`).
*   A `params` object with the standard [Messages API](https://platform.claude.com/docs/en/api/messages/create) parameters

You can [create a batch](https://platform.claude.com/docs/en/api/messages/batches/create) by passing this list into the `requests` parameter:

In this example, two separate requests are batched together for asynchronous processing. Each request has a unique `custom_id` and contains the standard parameters you'd use for a Messages API call.

When a batch is first created, the response has a processing status of `in_progress`.

### Tracking your batch

The Message Batch's `processing_status` field indicates the stage of processing the batch is in. It starts as `in_progress`, then updates to `ended` once all the requests in the batch have finished processing, and results are ready. You can monitor the state of your batch by visiting the [Console](https://platform.claude.com/settings/workspaces/default/batches), or using the [retrieval endpoint](https://platform.claude.com/docs/en/api/retrieving-message-batches).

#### Polling for Message Batch completion

To poll a Message Batch, you'll need its `id`, which is provided in the response when creating a batch or by listing batches. You can implement a polling loop that checks the batch status periodically until processing has ended:

### Listing all Message Batches

You can list all Message Batches in your Workspace using the [list endpoint](https://platform.claude.com/docs/en/api/listing-message-batches). The API supports pagination, automatically fetching additional pages as needed:

### Retrieving batch results

Once batch processing has ended, each Messages request in the batch has a result. There are four result types:

| Result type | Description |
| --- | --- |
| `succeeded` | Request was successful. Includes the message result. |
| `errored` | Request encountered an error and a message was not created. Possible errors include invalid requests and internal server errors. You will not be billed for these requests. |
| `canceled` | User canceled the batch before this request could be sent to the model. You will not be billed for these requests. |
| `expired` | Batch reached its 24-hour expiration before this request could be sent to the model. You will not be billed for these requests. |

The batch's `request_counts` shows an overview of your results, indicating how many requests reached each of these four states.

Results of the batch are available for download at the `results_url` property on the Message Batch, and if the organization permission allows, in the Console. Because of the potentially large size of the results, it's recommended to [stream results](https://platform.claude.com/docs/en/api/messages/batches/results) back rather than download them all at once.

The results are in `.jsonl` format, where each line is a valid JSON object representing the result of a single request in the Message Batch. For each streamed result, you can do something different depending on its `custom_id` and result type. Here is an example set of results:

If your result has an error, its `result.error` will be set to the standard [error shape](https://platform.claude.com/docs/en/api/errors#error-shapes).

### Canceling a Message Batch

You can cancel a Message Batch that is currently processing using the [cancel endpoint](https://platform.claude.com/docs/en/api/canceling-message-batches). Immediately after cancellation, a batch's `processing_status` will be `canceling`. You can use the same polling technique described earlier to wait until cancellation is finalized. Canceled batches end up with a status of `ended` and may contain partial results for requests that were processed before cancellation.

The response shows the batch in a `canceling` state:

### Using prompt caching with Message Batches

The Message Batches API supports prompt caching, allowing you to potentially reduce costs and processing time for batch requests. The pricing discounts from prompt caching and Message Batches can stack, providing even greater cost savings when both features are used together. However, because batch requests are processed asynchronously and concurrently, cache hits are provided on a best-effort basis. Users typically experience cache hit rates ranging from 30% to 98%, depending on their traffic patterns.

To maximize the likelihood of cache hits in your batch requests:

1.   Include identical `cache_control` blocks in every Message request within your batch.
2.   Maintain a steady stream of requests to prevent cache entries from expiring after their 5-minute lifetime.
3.   Structure your requests to share as much cached content as possible.

Example of implementing prompt caching in a batch:

In this example, both requests in the batch include identical system messages and the full text of Pride and Prejudice marked with `cache_control` to increase the likelihood of cache hits.

### Server tools and the agentic loop

All [server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) (web search, web fetch, code execution, MCP connectors, advisor, and tool search) work in batch requests. The batch worker runs the same server-side agentic loop as the synchronous Messages API.

Because there is no open connection to maintain, the batch loop runs **more iterations per turn** than a synchronous request before it returns `stop_reason: "pause_turn"`. If a batch result comes back with `pause_turn`, the turn did not finish; you can continue it by submitting the paused assistant content in a follow-up request (batch or synchronous) exactly as shown in the [pause_turn continuation pattern](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#the-server-side-loop-and-pause-turn).

The batch worker additionally throttles `web_search` per organization so that highly concurrent batch processing does not exhaust your organization's web-search rate limit. The batch retries throttled requests automatically; you don't need to handle this yourself, but very large web-search batches might take longer to complete.

### Extended output (beta)

The `output-300k-2026-03-24` beta header raises the `max_tokens` cap to 300,000 for batch requests using Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, Claude Sonnet 5, or Claude Sonnet 4.6. Include the header to generate outputs far longer than the standard 128k `max_tokens` limit in a single turn.

Use extended output for long-form generation such as book-length drafts and technical documentation, exhaustive structured data extraction, large code-generation scaffolds, and long reasoning chains.

A single 300k-token generation can take over an hour to complete, so plan your batch submissions with the 24-hour processing window in mind. Standard batch pricing (50% of standard API prices) applies.

### Best practices for effective batching

To get the most out of the Batches API:

*   Monitor batch processing status regularly and implement appropriate retry logic for failed requests.
*   Use meaningful `custom_id` values to easily match results with requests, since order is not guaranteed.
*   Consider breaking very large datasets into multiple batches for better manageability.
*   Dry run a single request shape with the Messages API to avoid validation errors.

### Troubleshooting common issues

If experiencing unexpected behavior:

*   Verify that the total batch request size doesn't exceed 256 MB. If the request size is too large, you may get a 413 `request_too_large` error.
*   Check that you're using [supported models](https://platform.claude.com/docs/en/build-with-claude/batch-processing#supported-models) for all requests in the batch.
*   Ensure each request in the batch has a unique `custom_id`.
*   Ensure that it has been less than 29 days since batch `created_at` (not processing `ended_at`) time. If over 29 days have passed, results will no longer be viewable.
*   Confirm that the batch has not been canceled.

Note that the failure of one request in a batch does not affect the processing of other requests.

## Batch storage and privacy

*   **Workspace isolation**: Batches are isolated within the Workspace they are created in. They can only be accessed by API requests in that same Workspace, or users with permission to view Workspace batches in the Console.

*   **Result availability**: Batch results are available for 29 days after the batch is created, allowing ample time for retrieval and processing.

## Data retention

Batch processing stores request and response data for up to 29 days after batch creation. You can delete a message batch at any time after processing using the `DELETE /v1/messages/batches/{batch_id}` endpoint. To delete an in-progress batch, cancel it first. Asynchronous processing requires server-side storage of both inputs and outputs until batch completion and result retrieval.

For ZDR eligibility across all features, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

## FAQ

## Next steps

Enable natural citations for RAG applications by providing search results with source attribution.

Reduce cost and latency by caching prompt prefixes shared across requests in a batch.