# Claude Managed Agents 会话事件流与增量预览

*原文: [https://platform.claude.com/docs/en/managed-agents/events-and-streaming](https://platform.claude.com/docs/en/managed-agents/events-and-streaming) · 来源: web · 生成时间: 2026-09-20T07:32:44.675947+00:00*

## 背景

LLM 生成响应时间较长且需要可观测、可中断和可重定向，因此 Managed Agents 采用事件驱动架构，用统一事件流传递用户指令与状态。默认情况下 agent 文本以完整 buffer 的 agent.message 事件返回，虽然正确但实时性不足。为同时兼顾实时展示与最终一致性，引入了可选的 event deltas 预览机制。

## 痛点

如果只等待 buffered agent.message，客户端在模型请求结束前看不到任何内容，长回复或思考过程会让用户感到卡顿。直接使用增量数据而不与权威事件协调，又会因预览片段可能被丢弃而渲染出错误或不完整的结果。

## 解决办法

在流连接上通过 event_deltas[] 参数 opt-in 开启预览，服务端为 agent.message 发送 event_start 宣告即将到来的事件 id，随后发送 event_delta 携带增量文本和 content block index。客户端把预览当作草稿，按 (event_id, index) 聚合增量文本；当权威的 agent.message 到达时按 id 匹配并替换为完整内容。每个 model request 结束时用 span.model_request_end 作为边界清理未闭环的预览。这样预览只负责低延迟展示，持久化记录仍以 buffered event 为准。

## 关键代码示例

```python
from collections import defaultdict

seen_ids = set()
previews = defaultdict(str)  # (event_id, delta_index) -> preview_text

def handle_event(ev):
    if ev.type == "event_start":
        # 记录接下来会有这个 event_id 的预览片段
        seen_ids.add(ev.event.id)
    elif ev.type == "event_delta":
        key = (ev.event_id, ev.delta.index)
        previews[key] += ev.delta.content.text
        render_running_text(previews[key])
    elif ev.type == "agent.message":
        # 权威事件到达，删除该 id 的所有草稿并渲染最终内容
        for key in [k for k in previews if k[0] == ev.id]:
            del previews[key]
        render_final(ev.content)
    elif ev.type == "span.model_request_end":
        # 当前 model request 结束，清理任何未闭环的预览
        cleanup_stale_previews(seen_ids)
```

这段代码展示手动累加器模式：event_start 只记录被预览的事件 id；event_delta 按 (event_id, index) 追加文本，key 由 delta 自带 event_id 和 index 组成；agent.message 到达时用 id 清掉所有相关草稿并渲染权威内容；span.model_request_end 是请求边界，用于处理可能因错误/中断而缺失 buffered event 的情况。

## 关键流程

1. 在 stream URL 上添加 event_deltas[]=agent.message（可重复添加 agent.thinking 开启思考预览），shell 中注意转义或给 URL 加引号。
2. 连接 session-level stream 或 thread stream 后按顺序读取事件。
3. 遇到 event_start 时记录其 event.id，它是要预览的 buffered event 的唯一标识。
4. 遇到 event_delta 时以 (event_id, delta.index) 为 key 追加 delta.content.text 并渲染运行文本。
5. 收到 buffered agent.message 时按 id 丢弃预览并渲染权威内容。
6. 收到 span.model_request_end 时清理当前请求内未闭环的预览。

## 关键点

- 事件流是双向的：user.* 和 system.message 由客户端发送，session.*、span.*、agent.* 用于可观测性。
- event_deltas 是 opt-in 的流式预览，默认的 buffered agent.message 才是持久化且权威的记录。
- event_start 和 event_delta 没有自己的 id 或 processed_at，只引用被预览事件的 id，因此不能单独用于事件历史。
- 累加器按 (event_id, index) 聚合增量文本，保证得到的是 content[index].text 的前缀，但不保证完整。
- 每个连接对同一 event_id 最多发送一个 event_start，且该 id 的 buffered event 是该连接最后交付的事件。
- 多智能体线程的预览只在对应 thread stream 上发送，路径是 /threads/{thread_id}/stream，而不是 /events/stream。

## 对比与权衡

- 相比默认的 buffered agent.message 模式，event deltas 提供了更低的展示延迟，但可能在高负载下丢弃部分片段，因此不适合作为持久化记录。
- 相比 WebSocket 双向长连接方案，基于 HTTP stream 的事件 delta 更容易穿透代理和基础设施，但客户端需要自己处理事件顺序与累加协调。
- 相比使用 SDK 内置的 accumulator helper，手动累加器更灵活、可定制，但需要自己维护 (event_id, index) 书签，容易在并发或错误路径下出错。

## 自测问题

**问: 为什么 event_deltas 需要 opt-in 而不是默认开启？**

默认 buffered agent.message 已经能提供完整正确的事件流；预览是 best-effort，增加带宽和客户端复杂度。opt-in 可让旧客户端保持兼容，也避免不需要实时预览的客户端为可能丢片段的数据付费。

**问: event_start/event_delta 没有自己的 id 和 processed_at，会带来哪些影响？**

它们不是持久化事件，不会出现在事件历史查询中；生命周期完全依附于被预览的 buffered event，所以必须按 event.id 做关联和清理。

**问: 如果因为网络问题丢失部分 event_delta，客户端如何保证最终展示正确？**

预览只是草稿，任何丢失都会使本地聚合文本成为最终 content 的前缀；当权威 agent.message 到达时按 id 替换为完整内容即可，最终一致性不受影响。

**问: agent.thinking 预览和 agent.message 预览有何区别？**

agent.thinking 只发送 event_start，不发送 event_delta，因此不能看到思考内容，只能作为进度信号；而 agent.message 会通过 event_delta 携带增量文本。

**问: 在多智能体 session 中，如何获取 subagent 的流式预览？**

每个 session thread 都有自己的 stream endpoint：/v1/sessions/{session_id}/threads/{thread_id}/stream，并且接受相同的 event_deltas[] 参数；子代理的预览只会出现在它自己的 thread stream 上，不是 session-level stream。

## 适用场景

- 构建需要逐 token 或逐片段渲染的聊天/Agent 界面。
- 多智能体编排中监控子智能体的生成进度和文本输出。
- 在语音合成、实时翻译等下游消费场景中，需要低延迟预览但以最终消息为准。
- 调试事件流顺序、生命周期以及 stream 与持久化事件关系时。

## 标签

`事件流` `增量预览` `Claude Managed Agents` `流式渲染` `多智能体`
