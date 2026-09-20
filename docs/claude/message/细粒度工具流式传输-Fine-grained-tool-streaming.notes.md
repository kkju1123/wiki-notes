# 细粒度工具流式传输（Fine-grained tool streaming）

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming](https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming) · 来源: web · 生成时间: 2026-09-20T04:16:41.930352+00:00*

## 背景

大模型工具调用的参数可能非常大，例如生成整个文件或大段代码。标准流式会等服务端把某个参数完整缓存并校验 JSON 后才下发，导致首字节延迟高。为满足低延迟、实时预览类应用，Claude 引入细粒度工具流式，让参数片段几乎在生成的同时就到达客户端。

## 痛点

开发者容易误以为开启 streaming 就已经是实时，实际上仍要等整个参数生成完毕才收到首个分片；同时若不知道 eager 模式可能返回非法 JSON，客户端会直接解析崩溃或执行无效工具输入。

## 解决办法

在目标工具定义上设置 eager_input_streaming: true，服务端不再缓冲和校验每个参数，而是持续下发 partial_json 字符串片段。客户端沿用标准 tool-use 累积协议：开始为空字符串，遇到 input_json_delta 就追加，等到 content_block_stop 再解析。由于服务端不保证合法 JSON，客户端必须安全解析；失败时把原始输入包装成 invalid_tool_input 字段，并通过 is_error=true 作为 tool result 返回给模型。类比终端逐字回显，而不是等命令执行完一次性打印。

## 关键代码示例

```javascript
const tools = [{
  name: 'make_file',
  input_schema: { type: 'object', properties: { content: { type: 'string' } } },
  eager_input_streaming: true,
}];

function accumulateToolInput(events) {
  let inputJson = '';
  for (const event of events) {
    if (event.type === 'content_block_start' && event.content_block.type === 'tool_use') {
      inputJson = '';
    } else if (event.type === 'content_block_delta' && event.delta.type === 'input_json_delta') {
      inputJson += event.delta.partial_json;
    } else if (event.type === 'content_block_stop') {
      try {
        return { ok: true, input: JSON.parse(inputJson) };
      } catch {
        return { ok: false, toolResult: JSON.stringify({ invalid_tool_input: inputJson }) };
      }
    }
  }
}
```

工具定义中 `eager_input_streaming: true` 让该工具跳过服务端缓冲与 JSON 校验；客户端在 `content_block_start` 初始化空串，把后续 `input_json_delta.partial_json` 逐段拼接；`content_block_stop` 时尝试解析，失败则返回 `{ invalid_tool_input: inputJson }`，可用于构造 `is_error: true` 的 tool result。

## 关键流程

1. 确认使用支持细粒度工具流式的模型和平台，并在请求中启用 streaming。
2. 在目标用户自定义工具定义中添加 eager_input_streaming: true；不要同时携带 legacy beta header。
3. 收到 content_block_start(type=tool_use) 时初始化 input_json 为空字符串，忽略 input: {} 这种占位对象。
4. 持续将 content_block_delta(type=input_json_delta) 的 partial_json 拼接到 input_json。
5. 收到 content_block_stop 时安全解析 input_json；如果解析失败或 stop_reason 为 max_tokens，不要执行工具。
6. 解析失败时构建 tool result，把原始输入包装为 invalid_tool_input 字段，并设置 is_error=true 返回给 Claude。

## 关键点

- eager_input_streaming 是工具级开关：true 开启细粒度流式，省略或 false 维持标准缓冲和校验，因此可以只对长参数工具开启。
- 细粒度流式跳过服务端 JSON 缓冲和校验，Claude 一旦开始生成参数就下发片段，大幅降低大参数的首分片延迟。
- 客户端累积协议与标准 tool-use streaming 一致：content_block_start 中 input: {} 只是占位符，真实输入由多个 input_json_delta.partial_json 拼接得到，停止后再解析。
- 因为服务端不再保证 JSON 有效性，客户端必须保护 JSON 解析，并准备把无效或截断的原始输入包装成 is_error=true 的 tool result 返回给模型。
- max_tokens 可能让响应停在参数中间，检查 stop_reason 是处理流式工具调用的必要步骤，避免把半截参数当完整输入执行。
- legacy beta header 已被 per-tool 字段取代，且不能与 computer use 或 browser use 工具集并存；新代码应使用 eager_input_streaming。

## 对比与权衡

- 相比标准缓冲式 tool streaming，细粒度 eager streaming 的首分片延迟更低，适合实时展示大参数生成；但代价是服务端不校验 JSON，客户端需要额外处理非法输入。
- 相比 legacy fine-grained-tool-streaming-2025-05-14 beta header 的请求级开关，per-tool eager_input_streaming 字段能精细控制单个工具，且显式 false 可以覆盖 header；但需要逐个工具配置。
- 相比 computer use 或 browser use 等托管工具集，用户自定义工具才支持 eager_input_streaming；托管工具集无法直接使用这一能力，并且旧 header 与它们互斥。

## 自测问题

**问: 为什么细粒度工具流式可以降低首个分片延迟？**

标准流式在服务端会缓冲并校验整个参数，校验通过后才把 input_json_delta 发出来；eager_input_streaming 让服务端省略这个等待，边生成边下发，所以大参数（长文档、代码块）的第一个片段更早到达客户端。

**问: 如果累积出来的工具输入不是合法 JSON，你会怎么处理？**

不能直接执行工具。应当安全解析，失败时构造 tool result，将原始字符串包装成 invalid_tool_input 字段，设置 is_error=true 返回给 Claude，让模型知道出了问题并重新生成或修复。

**问: content_block_start 里 input 是 {}，后面 delta 又是字符串，为什么类型不一致？**

{} 只是占位符，表示 content 数组中有一个 tool_use 块；partial_json 是流式字符串片段。两者职责不同，最终要把字符串片段拼接后再解析成对象，不能用 {} 直接合并。

**问: 如果工具调用因为 max_tokens 被截断怎么办？**

先检查 stop_reason，如果因 max_tokens 停住，参数可能不完整。可以调大 max_tokens 重试，或把已累积的原始片段按 invalid_json 返回给模型，让模型决定下一步。

**问: eager_input_streaming 和旧的 beta header 是什么关系？**

eager_input_streaming 是正式的工具级字段，用来替代 fine-grained-tool-streaming-2025-05-14 header。不设置或 false 维持缓冲；即使请求带旧 header，某工具显式 false 也会保持缓冲。旧 header 不能与 computer use 或 browser use 工具集组合。

## 适用场景

- 构建 AI 编程助手或其他需要实时展示工具生成的长文档、代码、配置的交互界面。
- 工具参数体积大且对首字节延迟敏感的场景，例如生成完整文件内容、长 JSON、设计稿代码。
- 自己实现客户端 SDK 或底层流式处理，需要精确控制 partial_json 累积与错误处理。
- 调试工具调用流式输出行为，分析片段边界或验证无效 JSON 的处理路径。

## 标签

`fine-grained tool streaming` `Claude API` `tool use` `streaming` `eager_input_streaming`
