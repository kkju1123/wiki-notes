---
title: Compaction：服务端上下文压缩机制
url: https://platform.claude.com/docs/en/build-with-claude/compaction
source_type: web
folder: claude
author: null
tags:
- 上下文管理
- Compaction
- Claude API
- 长对话
- 摘要压缩
summary: Claude API 的服务端上下文压缩机制，在接近上下文窗口阈值时自动摘要旧内容，延长有效上下文并保持响应质量。
fetched_at: '2026-09-20T02:59:19.705750+00:00'
---

Server-side context compaction for managing long conversations that approach context window limits.

Compaction extends the effective context length for long-running conversations and tasks by automatically summarizing older context when approaching the context window limit. It also keeps the active context small: as a conversation grows, response quality degrades, so compaction replaces older content with a concise summary.

This is ideal for:

*   Chat-based, multi-turn conversations where you want users to use one chat for a long period of time
*   Task-oriented prompts that require a lot of follow-up work (often tool use) that might exceed the context window

## How compaction works

When compaction is enabled, Claude automatically summarizes your conversation when it reaches the configured token threshold. The API:

1.   Detects when input tokens reach your specified trigger threshold.
2.   Generates a summary of the current conversation.
3.   Creates a `compaction` block containing the summary.
4.   Continues the response with the compacted context.

On subsequent requests, append the response to your messages. The API automatically drops all content blocks prior to the `compaction` block, continuing the conversation from the summary.

The previous steps describe threshold compaction, which most of this page covers. With the `compact-2026-09-04` beta header, you can instead request a summary on demand. That request is separate from your conversation turns and returns only the summary, so it can run in the background. When the block arrives, you swap it in for the messages it summarizes. See [Compact on demand with the `compaction` parameter](https://platform.claude.com/docs/en/build-with-claude/compaction#compact-on-demand-with-the-compaction-parameter).

## Basic usage

Enable compaction by adding the `compact_20260112` strategy to `context_management.edits` in your Messages API request.

## Parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `type` | string | Required | Must be `"compact_20260112"` |
| `trigger` | object | `{"type": "input_tokens", "value": 150000}` | When to trigger compaction. `input_tokens` is the only supported trigger type. `value` must be at least 50,000 tokens. |
| `pause_after_compaction` | boolean | `false` | Whether to pause after generating the compaction summary |
| `instructions` | string | `null` | Custom summarization prompt. Completely replaces the default prompt when provided. |

### Trigger configuration

Configure when compaction triggers using the `trigger` parameter:

### Custom summarization instructions

The default summarization prompt varies by model. Each default instructs Claude to write a summary inside `<summary></summary>` tags with the information needed to continue the task in a future context window. For example, some models use the following prompt:

You can provide custom instructions through the `instructions` parameter. Custom instructions don't supplement the default prompt. They replace it completely:

On Claude Fable 5.1 and Claude Mythos 5.1, a request with custom `instructions` summarizes from the visible conversation only: earlier thinking blocks are not part of the summarizer's input.

### Pausing after compaction

Use `pause_after_compaction` to pause the API after generating the compaction summary. This allows you to add additional content blocks (such as preserving recent messages or specific instruction-oriented messages) before the API continues with the response.

When enabled, the API returns a message with the `compaction` stop reason after generating the compaction block:

#### Enforcing a total token budget

When a model works on long tasks with many tool-use iterations, total token consumption can grow significantly. You can combine `pause_after_compaction` with a compaction counter to estimate cumulative usage and gracefully wrap up the task once a budget is reached.

This example appears in the SDK languages only: its value is the budget-tracking logic around the request. The raw request combines the `trigger` from [Trigger configuration](https://platform.claude.com/docs/en/build-with-claude/compaction#trigger-configuration) with `pause_after_compaction` from [Pausing after compaction](https://platform.claude.com/docs/en/build-with-claude/compaction#pausing-after-compaction).

## Working with compaction blocks

When compaction is triggered, the API returns a `compaction` block at the start of the assistant response.

A long-running conversation might result in multiple compactions. The last compaction block reflects the final state of the prompt, replacing content prior to it with the generated summary.

### Passing compaction blocks back

You must pass the `compaction` block back to the API on subsequent requests to continue the conversation with the shortened prompt. The simplest approach is to append the entire response content to your messages:

When the API receives a `compaction` block, all content blocks before it are ignored. You can either:

*   Keep the original messages in your list and let the API handle removing the compacted content
*   Manually drop the compacted messages and only include the compaction block onwards

On Claude Fable 5.1 and Claude Mythos 5.1, thinking blocks from before a `compaction` block aren't carried forward, so the summary is all the model has of that earlier work. If you write your own `instructions`, tell the model what the summary must retain; see [Tell the model what to preserve in compaction summaries](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#tell-the-model-what-to-preserve-in-compaction-summaries).

### Streaming

The compaction block streams differently from text blocks. You receive a `content_block_start` event, followed by a single `content_block_delta` with the complete summary content (no intermediate streaming), and then a `content_block_stop` event.

### Prompt caching

Compaction works well with [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching). You can add a `cache_control` breakpoint on compaction blocks to cache the summarized content.

#### Maximizing cache hits with system prompts

When compaction occurs, the summary becomes new content that needs to be written to the cache. Without additional cache breakpoints, this would also invalidate any cached system prompt, requiring it to be re-cached along with the compaction summary.

To maximize cache hit rates, add a `cache_control` breakpoint at the end of your system prompt. This keeps the system prompt cached separately from the conversation, so when compaction occurs:

*   The system prompt cache remains valid and is read from cache
*   Only the compaction summary needs to be written as a new cache entry

This keeps long system prompts cached across multiple compaction events throughout a conversation.

## Understanding usage

Compaction requires an additional sampling step, which contributes to rate limits and billing. The API returns detailed usage information in the response:

The `iterations` array shows usage for each sampling iteration. When compaction occurs, you'll see a `compaction` iteration followed by the main `message` iteration. The top-level `input_tokens` and `output_tokens` match the `message` iteration exactly in this example because there is only one non-compaction iteration. The final iteration's token counts reflect the effective context size after compaction.

## Combining with other features

### Server tools

When using server tools (such as web search), the compaction trigger is checked at the start of each sampling iteration. Compaction might occur multiple times within a single request depending on your trigger threshold and the amount of output generated.

### Token counting

The token counting endpoint (`/v1/messages/count_tokens`) applies existing `compaction` blocks in your prompt but does not trigger new compactions. Use it to check your effective token count after previous compactions:

## Examples

Here's a complete example of a long-running conversation with compaction:

On Claude Fable 5.1, remove the `thinking` and `redacted_thinking` blocks from any assistant turn you re-insert after the compaction block, or send `thinking.block_binding.prefix_mismatch_behavior: "drop_block"` with the `thinking-binding-controls-2026-08-01`[beta header](https://platform.claude.com/docs/en/api/beta-headers). Those blocks were produced when the full history was present, so they no longer pass the [conversation check](https://platform.claude.com/docs/en/build-with-claude/thinking#preserved-in-conversation). Where the check is enforced, the continuation request is rejected with a 400 error. The preserved text and tool blocks can stay as they are. Letting the API summarize everything, without re-inserting earlier turns, avoids this.

Here's an example that uses `pause_after_compaction` to preserve the prior exchange and the current user message (three messages total) verbatim instead of summarizing them:

## Current limitations

*   **Same model for summarization:** The model specified in your request is used for summarization. There is no option to use a different (for example, cheaper) model for the summary.

*   **Compaction might fail when tools are defined:** When your request includes `tools`, the model occasionally calls a tool during the internal summarization step instead of writing a summary. When this occurs, the response contains a `compaction` block with `content: null`. To prevent this, set [`instructions`](https://platform.claude.com/docs/en/build-with-claude/compaction#custom-summarization-instructions) to a prompt that explicitly tells the model not to call tools, for example:

## Compact on demand with the `compaction` parameter

The `compact-2026-09-04` beta adds a second way to compact. Threshold compaction summarizes partway through a request once the threshold you set is reached. With this beta, you instead send the top-level `compaction` parameter on a request of your choosing. The response contains a single signed `compaction` block and no reply. From then on, send that block first in `messages`, in place of the messages it summarizes, followed by any turns taken since. Claude sees the summary where those messages were. Everything after the summary reaches Claude unchanged. A threshold compaction block follows the messages it summarizes, but a signed block replaces them. Leaving the summarized messages in front of a signed block is a 400 error.

Compacting this way gives you three things. First, you decide when to compact. Second, the summarization request can run in the background while the conversation continues on its full history, and you swap the block in when it arrives. This is often called async or background compaction. Third, you can keep recent turns word for word after the summary, which is often called keep-tail compaction. Models with [preserved thinking](https://platform.claude.com/docs/en/build-with-claude/preserved-thinking) check earlier thinking blocks against the conversation. On those models, the thinking in turns that follow the summary, from either pattern, can stay valid after the swap, under the conditions in [Continue from the summary](https://platform.claude.com/docs/en/build-with-claude/compaction#continue-from-the-summary). That lets a long-running agent keep its train of thought. Use threshold compaction when you want the API to manage context inside ordinary requests. Use the `compaction` parameter when your application needs to control when compaction happens, can't pause while a summary is written, or must keep recent turns and their thinking after the summary.

Send the `compact-2026-09-04` beta header on the request that asks for the summary and on every later request that carries the signed block. On-demand compaction is available on the Claude API but not on Amazon Bedrock or Google Cloud. It works on Claude Fable 5.1, Claude Mythos 5.1, Claude Fable 5, Claude Mythos 5, Claude Mythos Preview, Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, Claude Sonnet 5, and Claude Sonnet 4.6. You can also call the [Models API](https://platform.claude.com/docs/en/api/beta/models/list) with the beta header and read each model's `capabilities.compaction`. You can't combine `compaction` with `context_management` on one request.

### Request a summary

Send the conversation as it stands with `"compaction": {"type": "summarize"}`. The API summarizes every message in the request once, generates no reply after it, and returns the block alone with `stop_reason``"compaction"`. Send the same `system` prompt and `tools` that you use for the rest of the conversation. The summarizer reads them, and on models with preserved thinking, the turns you keep stay valid only if they match. The conversation in this example has no `system` prompt or tools, so the request sends neither:

The summarization call uses the request's model, `system`, `tools`, thinking settings, and `max_tokens`. The summarizer reads the tool definitions but never runs a tool, and the response carries no thinking. `max_tokens` caps the whole call, including any thinking the model does before it writes the summary, so allow several thousand tokens. It is billed and rate-limited like any other request, and `usage.iterations` reports it as the `compaction` entry. The top-level `input_tokens` and `output_tokens` are zero because no reply was generated.

If the last `assistant` turn ends in a tool call with no result yet, the API rejects the request. Send that turn's tool results first. Also leave out `stop_sequences`, structured-output `output_config.format`, and a `tool_choice` of type `any` or `tool`. They would do nothing on a summarization call, and the API rejects them. The conversation must still fit the model's context window, so compact before you outgrow it, not after.

When you stream the response, the block arrives whole. You get one `content_block_start` event carrying the complete block, then `content_block_stop`, with no `content_block_delta` events. `ping` events can arrive before or between them.

### Continue from the summary

In your history, replace the messages you sent with the returned assistant message. Keep the `compaction` block exactly as the API returned it, including its `signature`. Send it first on every later request, with the beta header:

Here the second `assistant` message is the reply to the last summarized `user` turn. It arrived while the summary was being written, so it was not among the messages summarized. Two `assistant` messages in a row are fine here, because the block still comes first.

The API puts the summary where the block stands and passes every later message to Claude unchanged. Follow these rules:

*   Put the block first in `messages`, either as an `assistant` message of its own or as the first content block of the first message, whether that is a `user` or `assistant` message.
*   Remove the summarized messages. If any remain in front of the block, the request returns a 400 error (`compaction_block_misplaced`). If any remain after it, the API doesn't reject the request for that reason and sends them to the model again.
*   Send exactly one `compaction` block per request, on every later request. A request without the block reaches Claude without the summary.

To keep a tail of recent turns word for word, leave those turns out of the compaction request. The API summarizes every message it is sent, so send only the older turns, then put the block in front of the turns you kept.

If the conversation took more turns while a background summary request ran, drop exactly the messages you sent in the compaction request from the front of your history. Put the returned message in their place, and keep everything appended since:

Don't edit your history between sending the compaction request and making the swap, and make the swap on the first request after the block arrives. That way, thinking produced while the summary was being written stays valid.

On models with preserved thinking, the thinking blocks in the kept turns stay valid as long as both of these conditions hold:

*   The kept turns directly followed the summarized messages.
*   The `system` parameter and the `tools` not marked `defer_loading: true` are unchanged from the compaction request.

The first condition also rules out a first kept message that the API would merge into the last summarized message: one with the same role as the last summarized message, or a `role: "system"` message. The simplest way to meet it is to compact exactly the `messages` of a request you already made. To change `system` or `tools` without invalidating any kept thinking, compact the whole conversation first, so no turns are kept. Then change them on the next request.

A later request can use a different model, `system`, or `tools` than the compaction request, and the API still accepts the block. Such a change can invalidate the thinking in the kept turns, but it has no other effect.

To compact a conversation that already starts with a block, send `compaction` again. The new block summarizes the old summary and everything after it. From then on, send only the newest block.

### Write your own summarization prompt

Without `instructions`, the API uses its own summarization prompt. A non-blank `instructions` string (up to 16,384 characters) replaces that prompt entirely, as it does for threshold compaction (see [Custom summarization instructions](https://platform.claude.com/docs/en/build-with-claude/compaction#custom-summarization-instructions)). For example:

The summarizer reads the whole conversation, earlier thinking included, with or without `instructions`. That differs from threshold compaction on Claude Fable 5.1 and Claude Mythos 5.1, where custom `instructions` leave earlier thinking out. In your `instructions`, say what the summary must retain and tell the model not to call tools. The summarization call runs under the same safeguards as any other request.

### When no summary comes back

A summary is produced only when the summarization call ends normally with text and no tool call. Otherwise, the response is still a 200 with empty `content`. The call is still billed and reported in `usage.iterations`, with zero usage when no call could be made. The `stop_reason` is the one the summarization call ended with:

*   `"max_tokens"`: the summary was cut off.
*   `"model_context_window_exceeded"`: there was no room for the summarization prompt.
*   `"refusal"`: the request was declined. It is subject to the same safeguards as your other requests, and [`stop_details`](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons#refusal) identifies the policy category behind it.
*   `"tool_use"`: the model called a tool instead of writing the summary.
*   `"end_turn"`: the call returned no text.

Resend with a larger `max_tokens` after `"max_tokens"`, with shorter `instructions` or fewer messages after `"model_context_window_exceeded"`, or with `instructions` that tell the model not to call tools after `"tool_use"`. You can also continue without a summary.

A transient server problem while producing a block, or while reading one you sent back, returns a retryable 529 `overloaded_error` with `error.details.error_code` set to `compaction_unavailable`. Retry the request. Other rejections specific to this beta are 400 errors, and most have a message that says what to remove or resend. The exception is a request that leaves out the beta header: it fails with a generic validation error, such as `compaction: Extra inputs are not permitted`, that doesn't mention the header. Some also carry an `error.details.error_code` that starts with `compaction_`, mostly the errors about the block itself: an altered, misplaced, or duplicated block, or a request with nothing left to summarize. Parameter errors, such as a field that can't be combined with `compaction`, carry the message only.

### How it fits with the rest of the API

*   **Threshold compaction and context editing.** You can't send `compaction` and `context_management` on the same request. Threshold compaction (`compact_20260112`) can't run on a request that carries a signed block.
*   **Prompt caching.**`cache_control` on the block places a breakpoint after the summary.
*   **Mid-conversation system messages and tool changes.**`role: "system"` messages inside the summarized range are summarized. What they declared stops applying once the block replaces them. If an instruction or a [tool change](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes) still matters, state it again in a `role: "system"` message. Send that message right after your next new `user` turn, which comes after the kept turns, and leave it in your history from then on. Don't put it between the block and the kept turns, because that breaks the kept turns' thinking.
*   **Task budgets.** Don't send the `remaining` value of a [task budget](https://platform.claude.com/docs/en/build-with-claude/task-budgets) (`output_config.task_budget.remaining`) with `compaction` or on requests that carry the block. Doing so returns a 400 error.
*   **Token counting.** The [token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) endpoint ignores the `compaction` parameter.
*   **Content the summary can't carry.** Images, documents, `container_upload` blocks, and fetched URLs inside the summarized messages are gone once the block replaces them. Restate or re-upload anything a later turn still needs.

## Next steps

Automatically manage conversation context as it grows with context editing.

Learn about context window sizes and management strategies.

Explore a practical implementation that manages long-running conversations with instant session memory compaction using background threading and prompt caching.

## Compatibility

| Supported models | * Fable 5 and 5.1 * Mythos 5, 5.1, and Preview * Opus 4.6, 4.7, 4.8, and 5 * Sonnet 4.6 and 5 |
| --- |
| Supported platforms | * Claude API Beta * Claude Platform on AWS Beta * Amazon Bedrock Beta * Google Cloud Beta * Microsoft Foundry Beta |