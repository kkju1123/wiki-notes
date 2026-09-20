# 流式消息：用 Server-Sent Events 增量返回 Claude 响应

*原文: [https://platform.claude.com/docs/en/build-with-claude/streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) · 来源: web · 生成时间: 2026-09-20T02:32:55.776200+00:00*

## 背景

LLM 本质上是逐 token 生成内容，传统一次性 HTTP 响应会让用户长时间等待首字，且长生成过程可能触发客户端、代理或网关超时。Server-Sent Events 是建立在 HTTP/1.1 长连接上的单向文本流协议，适合把生成内容逐块从服务端推送到客户端。Anthropic Messages API 因此在创建 Message 时允许开启 stream:true。

## 痛点

如果不用流式或不懂事件结构，长回答、工具调用和思考过程都会表现为长时间无响应，首字延迟高；大 max_tokens 请求还可能被 HTTP 连接超时中断。直接集成时，如果把 partial JSON、未知事件、错误事件处理错，很容易解析崩溃或让流卡死。

## 解决办法

客户端创建 Message 时设置 stream:true，服务端按固定事件流推送：message_start → 多个 content_block 的 start/delta/stop → message_delta → message_stop。文本通过 text_delta 直接增量输出；tool_use 的 input 使用 partial JSON string 增量下发，需要累积到 content_block_stop 后解析成对象；thinking 通过 thinking_delta 下发，并在 block 结束前发送 signature_delta 做完整性校验。SDK 把这些事件封装成 text_stream、get_final_message/finalMessage、累加器等接口，既能逐字处理，也能内部流式但返回完整 Message，避免长请求超时。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    messages=[{'role': 'user', 'content': '解释 SSE 和 WebSocket 的区别'}],
) as stream:
    for text in stream.text_stream:
        print(text, end='', flush=True)

final = stream.get_final_message()
print('\nstop_reason:', final.stop_reason)

```

这段代码用 Python SDK 的 .stream() 隐式开启 SSE 流，text_stream 会把多个 text content_block_delta 的文本增量按顺序串起来，适合实时 UI；flush=True 保证终端立即刷新。循环结束后，get_final_message() 在内部累积所有事件并返回完整 Message，等价于非流式 create 的返回结果，同时仍然能拿到 stop_reason 等顶层字段。

## 关键流程

1. 创建 Message 请求时开启流式：SDK 可使用 .stream() 或 create(..., stream=True)。
2. 收到 message_start 后初始化空 Message 和 content 数组。
3. 对每个 content_block_start 创建对应内容块，并用 index 跟踪它在最终 content 数组中的位置。
4. 收到 content_block_delta 时按 delta 类型处理：text 直接拼接，tool_use 累积 partial JSON，thinking 追加思考文本。
5. 收到 content_block_stop 后，如果是 tool_use 则将累积 JSON 解析为对象；如果是 thinking 则记录 signature。
6. 处理 message_delta 更新 stop_reason、usage 等顶层字段，最终收到 message_stop 结束整个流。

## 关键点

- 每个 SSE 事件都有具名 event type 和 JSON data，而且 data 中也有 type 字段，客户端应按 type 分派处理，而不是只靠固定顺序。
- 内容块按 content_block_start、多个 content_block_delta、content_block_stop 三阶段推进，index 决定它对应最终 Message.content 的哪个位置。
- tool_use 的 input delta 是 partial JSON string，不是完整对象；必须累积到 content_block_stop 后再解析，或使用 SDK/Pydantic 的 partial JSON 能力。
- thinking 流会额外发送 signature_delta，用于校验 thinking block 完整性；display=omitted 时不会流式思考文本，只会发送空 delta 和签名后关闭。
- SDK 提供 text_stream、get_final_message/finalMessage、累加器等封装，既支持逐字处理，也能内部流式但返回完整 Message 以规避 HTTP 超时。
- 流中可能出现 ping、overloaded_error 以及未来新增事件，健壮的客户端必须忽略未知事件并妥善处理流内错误。

## 对比与权衡

- 相比非流式 Messages.create，stream=true 首字延迟更低、长响应更不容易被连接超时打断，但需要维护事件循环和连接状态，代码复杂度更高。
- 相比 WebSocket 双向长连接，SSE 是单向文本流，基于 HTTP/1.1 更容易兼容代理和负载均衡，但不适合在同一连接上进行客户端到服务端的复杂双向交互。
- 相比 Webhook 回调，SSE 不需要暴露公网回调地址，实时性更好；但连接生命周期与本次请求绑定，不适合完全异步的离线任务通知。

## 自测问题

**问: 为什么 LLM API 要使用 SSE 而不是普通 JSON 响应？**

LLM 是逐 token 生成，SSE 允许服务端生成一部分就推送一部分，显著降低 TTFB；对长输出可以避免 HTTP 或代理超时。SSE 基于 HTTP 单向文本流，协议简单，客户端可用 fetch 手动解析；注意 EventSource 通常不支持 POST 和自定义 header，所以 SDK 一般用 fetch 流读取。

**问: stream 事件流的典型顺序是什么？**

message_start → 一个或多个 content_block_start/delta/stop → 一个或多个 message_delta → message_stop，中间可能夹杂 ping 或 error。每个 block 都有 index，对应最终 Message.content 中的位置。

**问: tool_use 的 input delta 为什么是 partial JSON string？应该怎么解析？**

为了最大化增量粒度，模型会分块输出 input 内容；客户端应累积这些字符串，在 content_block_stop 后再进行 JSON 解析，或用 Pydantic partial JSON、SDK 工具解析辅助。注意最终 tool_use.input 是一个对象，不能把单个 delta 当完整 JSON 解析。

**问: 想要完整 Message 对象但不想手动拼接事件，该怎么做？**

可以使用 SDK 的 get_final_message()/finalMessage()/accumulator 等接口。它们底层仍会走 SSE 流，但 SDK 负责累积所有事件并返回与 create 等价的完整 Message。这种模式特别适合 max_tokens 很大时避免非流式请求超时。

**问: thinking stream 里 signature_delta 是做什么的？display 参数如何影响输出？**

signature_delta 用于验证 thinking block 完整性，确认推理内容由模型产生且未被篡改，它通常出现在 content_block_stop 前。display=omitted 时不会流式返回思考文本；display=updates 下只有部分模型在工具调用间产生的 progress update 会以 thinking_delta 下发文本。

## 适用场景

- 聊天类产品需要逐字显示 Claude 回复，降低用户感知等待。
- Agent 平台需要在工具调用过程中实时展示模型填写的参数或执行状态。
- 长文本生成、深度思考或推理任务中边生成边处理，而不是等全部完成。
- 直接集成 Claude HTTP API 的框架或运行时，需要自行实现 SSE 事件解析、容错与恢复。

## 标签

`Anthropic Claude` `Server-Sent Events` `流式 API` `Tool Use` `Thinking`
