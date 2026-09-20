---
title: Claude 拒绝响应与降级回退机制（Refusals and Fallback）
url: https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback
source_type: web
folder: claude
author: null
tags:
- Claude API
- 安全分类器
- fallback
- stop_reason
- LLM 安全
summary: Claude Fable/Opus 对安全拒绝返回 stop_reason=refusal，可通过服务端或客户端自动回退到其他模型，兼顾安全与可用性。
fetched_at: '2026-09-20T02:21:09.367181+00:00'
---

How Claude Fable and Claude Opus models return classifier refusals and how to retry refused requests on a fallback model.

Claude Fable 5.1, Claude Fable 5, and Claude Opus 5 include safety classifiers that can decline a request. When that happens, you receive a normal response, not an error, with `stop_reason: "refusal"`. Its `stop_details.category` names the policy area (see [What a refusal looks like](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#refusal-response)). You can usually still get an answer by sending the same request to another Claude model. This page shows you how to recognize a refusal and how to set up that retry.

Read this page when you build on any of these models and want declined requests to fall through to another model automatically. It also applies when you have seen `"refusal"` in a response and want to know what to do next.

Related pages:

*   [Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons): the full list of `stop_reason` values.
*   [Fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit): how to avoid paying the prompt-cache cost twice when you build the retry yourself.
*   [SDK middleware](https://platform.claude.com/docs/en/cli-sdks-libraries/middleware): the SDK helper that wraps all of this.
*   [Fallback and billing cookbook](https://platform.claude.com/cookbook/fable-5-fallback-billing-guide): a worked end-to-end example.

The simplest setup, in beta on the Claude API: set `fallbacks` to `"default"`, and the API retries a declined request on the fallback model Anthropic recommends for its refusal category. For categories with no recommended fallback, the refusal stands.

The following sections cover what a refusal response contains, when to use server-side or client-side fallback, and how each is billed.

## What a refusal looks like

A refusal is a successful HTTP 200 response with `stop_reason: "refusal"`:

The `stop_details` object explains the decline:

*   **`category`:** names the policy area that triggered the classifier.
*   **`explanation`:** a human-readable description. The text is not stable, so display it rather than parse it.
*   **`recommended_model`:** present only on requests that set `fallbacks` ([server-side fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback), beta). It names a model to retry directly when the API skipped the fallback attempt (for example, the fallback model was rate limited), and is `null` otherwise. It's a hint, not a guarantee.
*   `category` and `explanation` are both `null` when the refusal does not map to a named category. That `null` is a normal, permanent value, not a placeholder.
*   `stop_details` itself is `null` for every stop reason other than `refusal`.

| `category` | What it means |
| --- | --- |
| `"cyber"` | The request could enable cyber harm, such as malware or exploit development. Benign cybersecurity work can also trigger this category. |
| `"bio"` | The request could enable biological harm, such as dangerous lab methods. Beneficial life sciences work can also trigger this category. |
| `"frontier_llm"` | The request could assist the development of competing AI models, which is restricted under [Anthropic's commercial terms](https://www.anthropic.com/legal/commercial-terms). Benign machine learning work can also trigger this category. |
| `"reasoning_extraction"` | The request asks the model to reproduce its internal reasoning in the response text. To get reasoning in a structured form instead, use [adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/thinking). |
| `"general_harms"` | The request falls under a usage-policy area outside the four named categories. Benign work can also trigger this category. |

A refusal can arrive before any output, or mid-stream after partial output. In either case, treat any partial output as incomplete and discard it.

## Picking a fallback approach

There are three ways to retry a refused request on another model. The right one depends on where you are running and how much control you need.

| Your situation | Use | Why |
| --- | --- | --- |
| Claude API, simplest setup | [Server-side fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback) | One request, one response. The API handles the retry. |
| Any platform, using an Anthropic SDK | [The SDK middleware](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#client-side-fallback) | Configure once on the client. Retries happen automatically. |
| Raw HTTP or custom retry logic | [A manual retry](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#manual-retry) with [fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit) | Full control. Fallback credit keeps the cost down. |

Server-side fallback and the SDK middleware apply fallback credit for you. You only need the [Fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit) page when you build the retry yourself.

## Server-side fallback

Server-side fallback retries a refused request inside a single API call. In the default mode, when the primary model declines and the refusal category has a recommended fallback, the API runs the same request on the model Anthropic recommends for that category. You can instead [name up to three fallback models of your own](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#naming-your-own-fallback-models). Either way, you get back one response that names the model that answered, so your user gets an answer in one round trip.

### Making the request

Set the `fallbacks` parameter to the string `"default"` and send the `server-side-fallback-2026-07-01` beta header. The API then applies the requested model's server-defined default routing, which selects a recommended fallback model based on the refusal category the classifier reports, so refused requests are served without you maintaining a model list as recommendations change.

Default routing never draws the up-front [oversized-image rejection](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#oversized-image-error) for models you did not choose: a routed model that would resize an image marked `"oversized_image": "error"` is dropped from the routing instead, so a marked image is never served resized.

Anthropic sets safeguards for each model individually and for each policy category, in line with the model's capability: depending on the category, a flagged request may fall back to a less capable model or be declined. The `"default"` mode encodes these per-model, per-category recommendations for you, so a refused request is retried on the model Anthropic recommends for that category. Fallbacks are visible either way: the response names the model that served it, and the `fallback` content block marks the handoff.

The routing is applied server-side and is not published per model on the [Models API](https://platform.claude.com/docs/en/api/models/list). To see which model served a refused request, check the response's top-level `model` field and look for a `fallback_message` entry in `usage.iterations`, as this page's samples do.

Only a safety classifier decline triggers the fallback. A rate limit, overload, or server error on the requested model is returned to you as-is.

### Naming your own fallback models

Instead of default routing, you can set `fallbacks` to a list of up to three models. When the requested model declines, the API runs the next model in the chain on the same request. Use this form when you want to control exactly which models serve refused requests, such as pinning a model your application has qualified.

Named fallback models count toward the [oversized-image check](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#oversized-image-error): a request whose image block sets `"oversized_image": "error"` is checked up front against the requested model and every named fallback, is rejected if any of them would resize that image, and the rejection's reported rescale target fits them all.

The highlighted lines are the only difference from the default-routing request.

A few rules apply to the `fallbacks` list:

*   Entries are tried in order. Each must be distinct from the other entries and from the requested model.
*   Each entry must be one of the requested model's permitted targets. With the beta header set, that list is published as `allowed_fallback_models` on the model's entry in the [Models API](https://platform.claude.com/docs/en/api/models/list).
*   Each entry names a `model` and can override `max_tokens`, `thinking`, `output_config`, and `speed` for that attempt only.
*   The request must be valid as a direct request to every model named. If a fallback model does not support a feature the request uses, the API rejects the request up front.
*   As with the default mode, only a safety classifier decline triggers the fallback. A rate limit, overload, or server error on the requested model is returned to you as-is.
*   If a fallback model is rate limited or overloaded, the fallback attempt is not made and the preceding refusal is returned instead. The refusal's `stop_details.recommended_model` then names a model to retry directly. Size the fallback model's rate limits for the refusal volume you expect, or fallbacks degrade to refusals under load.

The response has the same shape in both modes: the model that served the turn appears in the top-level `model` field, a `fallback` content block marks the handoff, and `usage.iterations` records each attempt.

### What the response contains

The response looks like any other message, with two additions:

*   The top-level `model` field reports the model that produced the returned message, whether that is the requested model or a fallback.
*   A `fallback` content block marks each point in `content` where one model's output gives way to the next: `{"type": "fallback", "from": {"model": ...}, "to": {"model": ...}}`.
    *   `from.model` echoes the model string you sent when the declining hop is the requested model.
    *   `to.model` is always the resolved ID of the model that continues.

On a refusal before any output, the `fallback` block is the first content block. For example, when default routing selects Claude Opus 4.8 for the refusal's category:

The `usage.iterations` array records every attempt. A model that declined appears as an ordinary `message` entry, and the model that served the turn appears as a `fallback_message` entry. If every model in the chain declines, the response is the last model's refusal, with a `message` entry for each earlier hop and a `fallback_message` entry for the last.

[Sticky routing](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#sticky-routing) can send a later turn straight to the fallback model. Such a turn carries no `fallback` content block, because no model declined that turn. Identify it by the `fallback_message` entry in `usage.iterations`, the absence of a `message` entry for the requested model, and the response's `model` field.

### Continuing the conversation

On the next turn, send the assistant content back as you received it. After a mid-output fallback, `content` can include block types the declining model produced before the handoff. The following table covers which to keep and which to drop when you echo the turn.

| Block type | On the next turn |
| --- | --- |
| `fallback` | Keep it exactly where it appeared. The API uses its position to validate the thinking blocks around it, so a request that echoes thinking blocks from both sides of the boundary is rejected if the block is omitted or moved. |
| `text` | Keep. |
| Any block after the final `fallback` block | Keep. |
| `thinking`, `redacted_thinking`, or `connector_text` before the final `fallback` block | Drop. |
| Client-side `tool_use` before the final `fallback` block | Drop. |
| `server_tool_use` before the final `fallback` block | Keep when paired with its result. Drop when it has no matching result. |

### Streaming

On a streaming request, the retry happens on the same stream, and nothing you have already received is invalidated. What you see depends on when the decline happens.

**When the decline happens before any output:**

*   `message_start` names the fallback model, and the `fallback` block is the first content block.
*   Because `message_start` waits for the fallback attempt to start, time to first byte includes the declined attempt.

**When the decline happens mid-output:**

*   The open content block closes, and the `fallback` block (an ordinary `content_block_start` and `content_block_stop` pair with no deltas) marks the boundary.
*   The fallback model continues from the partial output. Only the partial output's `text` blocks are passed to the fallback model as context. Other block types remain in `content`.
*   `message_start` already named the requested model, so read the serving model from the `fallback` block's `to.model` and from the `fallback_message` entry in the final `message_delta`'s `usage.iterations`.

### Non-streaming responses

On a non-streaming request, a mid-output decline behaves differently: the response omits the declined model's partial output, and the fallback model answers from scratch. The result looks like a decline before any output, with the `fallback` block first. The declined attempt and its output tokens still appear in `usage.iterations`.

### Billing and rate limits

An attempt that declined before producing any output is not billed: its tokens are reported on its `usage.iterations` entry but not charged. Every attempt that produced output, including one that declined partway through its response, is billed separately at the rates of the model that ran it. The `usage.iterations` array is the per-attempt record of what you're billed. The top-level `usage` counts describe only the attempt that produced the returned message. Tokens from different models are never summed into one field.

Every attempt that runs, including one that declined, counts against its own model's rate limits.

### Sticky routing

After a conversation falls back, the API records which model served it. Later requests for that conversation that include `fallbacks` go directly to that fallback model, without running the requested model. This avoids paying for an attempt that would predictably be declined again on every turn.

A few properties of the routing decision:

*   It is retained for approximately 1 hour and is scoped to your organization.
*   It is stored as a content hash of the conversation prefix plus the model that served it. The message content itself is not stored.
*   It is best-effort, so your code must handle the requested model being tried again at any time.

Sticky routing applies to both streaming and non-streaming requests. On a streaming request, the routing decision is made before the stream opens, so the `message_start` event's `model` field already carries the fallback model's ID.

## Client-side fallback with the SDK middleware

Every Anthropic SDK includes a refusal-fallback middleware. You configure it once on the client with your list of fallback models. Calls through `client.beta.messages` then retry refused requests automatically, on any platform. The middleware also sends the `fallback-credit-2026-07-01` beta header on every request it handles, so retries are repriced without per-request setup.

### Setting it up

Pass the middleware to the client constructor, and share one `BetaFallbackState` instance across the requests of a conversation.

### How it behaves

*   Retries walk your fallback list in order. A fallback model that itself refuses passes the request to the next entry.
*   When every model in the list has declined, the middleware returns the final refusal (the last model's refusal response) rather than raising an error.
*   Thinking blocks from Claude Fable 5.1 or Claude Fable 5 pass through unchanged. Each retry re-sends your original request body, and the only blocks the middleware removes from conversation history on later requests are the `fallback` boundary blocks it added itself. The fallback model can't read Claude Fable 5.1 blocks, which are [preserved only for that model or a newer one](https://platform.claude.com/docs/en/build-with-claude/thinking#preserved-for-model), so the API drops them.
*   Responses served through the middleware include a `fallback` content block at each model boundary, the same as server-side fallback responses. The middleware manages those blocks for you on later requests.
*   The model that accepted is recorded in `BetaFallbackState`, so follow-up requests that share the state stay pinned to it rather than re-asking a model that refused.

## Writing the retry yourself

Over raw HTTP or with custom retry logic, implement the pattern the middleware wraps:

1.   ### Detect the refusal

Check the response for `stop_reason: "refusal"`. 
2.   
### Re-send on a fallback model

Send the same request with `model` set to a fallback model, such as Claude Opus 4.8. Another model can normally serve a request that Claude Fable 5.1 or Claude Fable 5 declines. How you handle the conversation history depends on whether you redeem a [fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit):

    *   **Not redeeming a credit:** you can leave the earlier `thinking` and `redacted_thinking` blocks in place or strip them to save input tokens. The fallback model cannot use them either way: it ignores Claude Fable 5 blocks, and Claude Fable 5.1 blocks are [preserved only for that model or a newer one](https://platform.claude.com/docs/en/build-with-claude/thinking#preserved-for-model), so the API drops them.
    *   **Redeeming a credit:** send the body unchanged, because redemption requires an exact match. The server handles the earlier model's thinking blocks on a redemption, so do not strip them (see [Fields that must match the refused request](https://platform.claude.com/docs/en/build-with-claude/fallback-credit#reference)).

3.   ### Stay on the fallback model

For multi-turn conversations, keep using the fallback model for subsequent turns rather than switching back. 

A manual retry writes the fallback model's prompt cache from scratch, which costs more than reading an existing cache. [Fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit) refunds that cost; redeem it on every retry you build yourself.

## Refusals in Message Batches

A refused request in a [Message Batch](https://platform.claude.com/docs/en/build-with-claude/batch-processing) comes back as `result.type: "succeeded"` with `stop_reason: "refusal"`. Batch results carry the same `stop_details` object as synchronous responses, so you can detect refusals through either `stop_reason` or `stop_details.type`. One difference: batch refusals don't mint fallback credits, so `stop_details` on a batch result never includes a `fallback_credit_token`.

Server-side fallback is not available for batches (a batch request that includes `fallbacks` produces a per-item errored result). To retry refused batch items:

1.   Collect the refused items from the results.
2.   Strip the Claude Fable 5.1 or Claude Fable 5 thinking blocks from any multi-turn histories.
3.   Resubmit them on a fallback model as a new batch or as direct requests.

## Common pitfalls

*   **Retry on a different model.** Re-sending a refused request to the same model usually earns another refusal. Point the retry at the fallback model.
*   **Budget retries per request, not per turn or per session.** A single turn can produce several refusals, for example an agent plus its sub-agents.
*   **Configure fallback on every request path.** Retry handlers, error-recovery branches, and background workers all need it. A handler that re-issues a request without fallback loses the protection on exactly the requests most likely to need it.
*   **Give sub-agent calls their own fallback.** The `fallbacks` parameter does not propagate into model calls made from inside tool execution.
*   **Make fallback a property of the request, not of ambient state.** A shared flag, cached config value, or global toggle can drift out of sync and silently leave a request unprotected. When you cannot confirm fallback is active, configure it rather than assume it is on.
*   **Instrument refusals as their own signal.** A refusal is an HTTP 200, so monitoring built on error rates or 5xx responses never sees it. Emit one event per refusal and one per fallback-served response (the `fallback_message` entry in `usage.iterations` marks the latter), then alert on the gap between the two counts.
*   **Branch on `stop_reason` or `stop_details.type`, not on `content` or the inner `stop_details` fields.** The `stop_details` object is always present on a refusal, but its `category` and `explanation` fields can be `null`. Check for `stop_reason` equal to `"refusal"` directly.

## Next steps

Avoid paying the prompt-cache cost twice when you build the retry yourself.

Every `stop_reason` value and how to handle it.

How SDK middleware works, including the refusal-fallback helper.

Move an existing application to Claude Fable 5.1.