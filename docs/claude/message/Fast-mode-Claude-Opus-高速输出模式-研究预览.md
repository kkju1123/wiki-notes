---
title: Fast mode：Claude Opus 高速输出模式（研究预览）
url: https://platform.claude.com/docs/en/build-with-claude/fast-mode
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Fast Mode
- 推理加速
- API 限流
- LLM 性能优化
summary: Claude Opus 5/4.8 的付费高速模式，通过更快推理配置将输出 tokens 每秒提升最高 2.5 倍，不改变模型能力。
fetched_at: '2026-09-20T02:28:51.938750+00:00'
---

Get up to 2.5x higher output tokens per second from supported Claude Opus models.

Fast mode delivers up to 2.5x higher output tokens per second from Claude Opus 5 and Claude Opus 4.8 at premium pricing. Set `speed: "fast"` with the `fast-mode-2026-02-01` beta header on your request to opt in.

## Supported models

Fast mode is supported on the following models:

*   Claude Opus 5 ()
*   Claude Opus 4.8 ()

## How fast mode works

Fast mode runs the same model with a faster inference configuration. There is no change to intelligence or capabilities.

*   Up to 2.5x higher output tokens per second compared to standard speed
*   Speed benefits are focused on output tokens per second (OTPS), not time to first token (TTFT)
*   Same model weights and behavior (not a different model)
*   Compatible with [streaming](https://platform.claude.com/docs/en/build-with-claude/streaming), where the OTPS gain is most visible

## Basic usage

## Pricing

Fast mode is priced at a multiplier on standard rates across the full context window, including requests over 200k input tokens. The following table shows fast mode pricing for the supported models:

| Model | Input | Output |
| --- | --- | --- |
| Claude Opus 5 / Claude Opus 4.8 | $10 USD / MTok | $50 USD / MTok |

Fast mode pricing stacks with other pricing modifiers:

*   [Prompt caching multipliers](https://platform.claude.com/docs/en/about-claude/pricing#prompt-caching) apply on top of fast mode pricing
*   [Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency) multipliers apply on top of fast mode pricing

For complete pricing details, see the [Pricing](https://platform.claude.com/docs/en/about-claude/pricing#fast-mode-pricing) page.

## Rate limits

Fast mode has a dedicated rate limit that is separate from standard Opus rate limits. When your fast mode rate limit is exceeded, the API returns a `429` error with a `retry-after` header indicating when capacity will be available.

The response includes headers that indicate your fast mode rate limit status:

| Header | Description |
| --- | --- |
| `anthropic-fast-input-tokens-limit` | Maximum fast mode input tokens per minute |
| `anthropic-fast-input-tokens-remaining` | Remaining fast mode input tokens |
| `anthropic-fast-input-tokens-reset` | Time when the fast mode input token limit resets |
| `anthropic-fast-output-tokens-limit` | Maximum fast mode output tokens per minute |
| `anthropic-fast-output-tokens-remaining` | Remaining fast mode output tokens |
| `anthropic-fast-output-tokens-reset` | Time when the fast mode output token limit resets |

For tier-specific rate limits, see the [Rate limits](https://platform.claude.com/docs/en/api/rate-limits) page.

## Checking which speed was used

The response `usage` object includes a `speed` field that indicates which speed was used, either `"fast"` or `"standard"`. Requesting `speed: "fast"` on a [model that doesn't support fast mode](https://platform.claude.com/docs/en/build-with-claude/fast-mode#supported-models) returns an error, and so does exceeding fast mode's rate limits or capacity (a `429` or `529`). When a request with `speed: "fast"` succeeds, `usage.speed` is `"fast"`. If you are using Claude Opus 4.6 and request fast mode, its behavior is unique. Instead of returning an error like other models that don't support fast mode, it silently switches to standard speed. Though there is no error with Opus 4.6, the `speed` field accurately shows `"standard"`.

To track fast mode usage and costs across your organization, see the [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api).

## Retries and fallback

### Automatic retries

When fast mode rate limits are exceeded, the API returns a `429` error with a `retry-after` header. The Anthropic SDKs automatically retry these requests up to 2 times by default (configurable with `max_retries`), waiting for the server-specified delay before each retry. Because fast mode uses continuous token replenishment, the `retry-after` delay is typically short and requests succeed once capacity is available.

### Falling back to standard speed

If you'd prefer to fall back to standard speed rather than wait for fast mode capacity, catch the rate limit error and retry without `speed: "fast"`. Set `max_retries` to `0` on the initial fast request to skip automatic retries and fail immediately on rate limit errors.

Because setting `max_retries` to `0` also disables retries for other transient errors (overloaded, internal server errors), the following examples reissue the original request with default retries for those cases.

## Considerations

*   **Prompt caching:** Switching between fast and standard speed invalidates the prompt cache. Requests at different speeds do not share cached prefixes.
*   **Supported models:** Fast mode is supported on Claude Opus 5 and Claude Opus 4.8. See [Supported models](https://platform.claude.com/docs/en/build-with-claude/fast-mode#supported-models).
*   **TTFT:** Fast mode's benefits are focused on output tokens per second (OTPS), not time to first token (TTFT).
*   **Batch API:** Fast mode is not available with the [Batch API](https://platform.claude.com/docs/en/build-with-claude/batch-processing).
*   **Priority Tier:** Fast mode is not available with a [Priority Tier](https://platform.claude.com/docs/en/api/service-tiers) commitment.
*   **Claude Platform on AWS:** Fast mode is not currently available on [Claude Platform on AWS](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws).

## Next steps

Get validated JSON results from agent workflows.

Learn about Anthropic's pricing structure for models and features.

Control how many tokens Claude uses when responding with the effort parameter, trading off between response thoroughness and token efficiency.

Stream Messages API responses incrementally with server-sent events, including text, tool use, and extended thinking deltas.