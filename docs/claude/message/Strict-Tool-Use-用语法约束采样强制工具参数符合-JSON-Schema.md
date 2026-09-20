---
title: Strict Tool Use：用语法约束采样强制工具参数符合 JSON Schema
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use
source_type: web
folder: claude/message
author: null
tags:
- Strict Tool Use
- grammar-constrained sampling
- JSON Schema
- Claude Tools
- agent reliability
summary: 语法约束采样在解码层强制 Claude 工具输入匹配 JSON Schema，避免类型错误与缺字段，提升代理系统可靠性。
fetched_at: '2026-09-20T03:44:17.552307+00:00'
---

Enforce JSON Schema compliance on Claude's tool inputs with grammar-constrained sampling.

Setting `strict: true` on a tool definition guarantees Claude's tool inputs match your JSON Schema by constraining the model's token sampling to schema-valid outputs (a technique called grammar-constrained sampling). This page covers why strict mode matters for agents, how to enable it, and common use cases. For the supported JSON Schema subset, see [JSON Schema limitations](https://platform.claude.com/docs/en/build-with-claude/structured-outputs#json-schema-limitations). For non-strict schema guidance, see [Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools).

Strict tool use validates tool parameters, ensuring Claude calls your functions with correctly-typed arguments. Use strict tool use when you need to:

*   Validate tool parameters
*   Build agentic workflows
*   Ensure type-safe function calls
*   Handle complex tools with nested properties

## Why strict tool use matters for agents

Building reliable agentic systems requires guaranteed schema conformance. Without strict mode, Claude might return incompatible types (`"2"` instead of `2`) or omit required fields, breaking your functions and causing runtime errors.

Strict tool use guarantees type-safe parameters:

*   Functions receive correctly-typed arguments every time
*   No need to validate and retry tool calls
*   Production-ready agents that work consistently at scale

For example, suppose a booking system needs `passengers: int`. Without strict mode, Claude might provide `passengers: "two"` or `passengers: "2"`. With `strict: true`, the response always contains `passengers: 2`.

## Quick start

**Response format:** Tool use blocks with validated inputs in `response.content[x].input`

**Guarantees:**

*   Tool `input` strictly follows the `input_schema`
*   Tool `name` is always valid (from provided tools or server tools)

## How it works

1.   ### Define your tool schema

Create a JSON schema for your tool's `input_schema`. The schema uses standard JSON Schema format with some limitations (see [JSON Schema limitations](https://platform.claude.com/docs/en/build-with-claude/structured-outputs#json-schema-limitations)). 
2.   ### Add strict: true

Set `"strict": true` as a top-level property in your tool definition, alongside `name`, `description`, and `input_schema`. 
3.   ### Handle tool calls

When Claude uses the tool, the `input` field in the `tool_use` block strictly follows your `input_schema`, and the `name` is always valid. 

The [computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolset entries (`computer_toolset_20260801` and `browser_toolset_20260801`) don't accept `strict: true`; a request that sets it on either entry is rejected.

## Common use cases

## Data retention

Strict tool use compiles tool `input_schema` definitions into grammars using the same pipeline as [structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs). Tool schemas are temporarily cached for up to 24 hours since last use. Prompts and responses are not retained beyond the API response.

Strict tool use is HIPAA eligible, but **protected health information (PHI) must not be included in tool schema definitions**. The API caches compiled schemas separately from message content, and these cached schemas do not receive the same PHI protections as prompts and responses. Do not include PHI in `input_schema` property names, `enum` values, `const` values, or `pattern` regular expressions. PHI should only appear in message content (prompts and responses), where it is protected under HIPAA safeguards.

For ZDR and HIPAA eligibility across all features, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

## Next steps

Fetch and read content from specific URLs to bring live web content into Claude's context.

Cache tool definitions across turns to reduce cost and latency.

Get validated JSON responses using the same grammar-constrained sampling.

Specify tool schemas, write effective descriptions, and control when Claude calls your tools.