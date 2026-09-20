# Cache diagnostics：定位提示缓存未命中的根因

*原文: [https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics) · 来源: web · 生成时间: 2026-09-20T03:08:08.353885+00:00*

## 背景

Prompt caching 能大幅降低延迟和成本，但前提是请求前缀必须与最近一次请求逐字节一致。实际系统中，一个工具顺序调整、系统提示里的时间戳、早期消息被编辑，都会让缓存静默失效。缓存诊断由此出现，为这类不可见问题提供可观测性。

## 痛点

没有缓存诊断时，开发者只能看到 usage.cache_read_input_tokens 变为零，却不知道是 system、tools、messages 还是 model 发生了变化。修复只能靠猜或人工逐字 diff，多轮对话中尤其痛苦。

## 解决办法

开启 beta header 后，API 为每个请求保存轻量指纹，包含结构哈希和 token 估计值，不保存原始内容。下一次请求通过 diagnostics.previous_message_id 传入上一响应 id，API 会比较两次请求的 model、system、tools、messages，并返回第一个分歧点。它的比较对象是请求结构，而不是缓存是否命中，因此需要结合 usage.cache_read_input_tokens 判断：请求没变但缓存没了，是缓存过期；请求变了且缓存没命中，才是你的 bug。可以类比为 git diff，但它只看文件结构变化，不暴露源码内容。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
previous_message_id = None
messages = []

for turn, user_text in enumerate(["第一轮", "第二轮"]):
    messages.append({"role": "user", "content": user_text})
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        system="You are a helpful assistant.",  # 必须保持字节稳定
        messages=messages,
        extra_headers={"anthropic-beta": "cache-diagnostics"},
        extra_body={"diagnostics": {"previous_message_id": previous_message_id}},
    )
    previous_message_id = response.id
    diag = getattr(response, "diagnostics", None)
    cache_read = response.usage.cache_read_input_tokens
    print(f"turn={turn} diag={diag} cache_read={cache_read}")

```

首轮请求把 previous_message_id 设为 null，表示只开启诊断、暂不比较。后续每一轮都传入上一轮响应 id，API 会对比两轮请求结构。代码同时打印 diagnostics 和 cache_read_input_tokens：前者说明请求结构是否变化，后者说明缓存是否实际命中。system 保持常量是关键，否则很容易在诊断中被标记为 system_changed。

## 关键流程

1. 每轮请求都携带 cache diagnostics 对应的 beta header，确保 API 保存并比较指纹。
2. 第一轮将 diagnostics.previous_message_id 设为 null，只开启诊断，不进行对比。
3. 后续每一轮把上一轮响应对象的 id 作为 previous_message_id 传入。
4. 从响应中的 diagnostics 字段读取比较结果；流式响应中它出现在 message_start 事件上。
5. 结合 usage.cache_read_input_tokens 和诊断类型矩阵，判断是请求变化、缓存过期还是正常工作。

## 关键点

- diagnostics 比较的是请求结构差异，并不直接说明缓存是否命中；必须和 usage.cache_read_input_tokens 一起看。
- cache_miss_reason 只报告最早的分歧点，后续分歧可能被隐藏，修复第一个后需要重试才能暴露下一个。
- 指纹只包含哈希和 token 估算值，不保存原始 prompt 内容，且限制在组织/工作区范围内并短期保留。
- 保持 system、tools 和 messages 前缀稳定是提高缓存命中率的核心手段，动态数据应放到 user message 或 cache_control breakpoint 之后。
- 该功能目前是 Beta、仅限 Claude API，Bedrock 和 Google Cloud 上不可用。

## 对比与权衡

- 相比仅监控 usage.cache_read_input_tokens，cache diagnostics 把“缓存没命中”细化为“哪个字段变了”，减少猜测，但它是 Beta 且指纹保留时间短。
- 相比人工 diff 原始请求，自动诊断不需要接触原始 prompt 内容，更快且更安全，但它的比较粒度到顶层结构，不能直接告诉你具体哪个 message 字符变了。

## 自测问题

**问: Prompt caching 命中条件是什么？为什么把时间戳放 system prompt 会导致 cache miss？**

缓存按前缀逐字节匹配，system 在 tools 和 messages 之前。system 里任何动态值都会让整个前缀变化，导致后续所有缓存失效。应把动态内容移到第一条 user message 或 cache_control breakpoint 之后。

**问: diagnostics 返回 null 但 cache_read_input_tokens 为零，可能是什么情况？**

说明请求结构一致，但缓存条目不可用，可能因为 TTL 过期、缓存被驱逐或连续请求间隔太长。首轮 previous_message_id 为 null 时也有同样表现，属于正常写入缓存阶段。

**问: 为什么 cache_miss_reason 只报告最早的分歧点？**

API 在比较过程中发现第一处不同就停止返回，后续差异会被遮蔽。修复第一个分歧后重新请求，才能看到下一个问题。这避免了在多个字段变化时一次性淹没开发者。

**问: tool_choice 或 thinking 参数变了，diagnostics 会怎么报？**

这类参数不在 model/system/tools/messages 的直接比较范围内，通常会返回 unavailable。它表示诊断信息不可用，但请求本身正常处理。应保持所有会影响 prompt 结构的参数在缓存对话期间恒定。

**问: 多轮 agent 如何设计才能稳定命中 prompt cache？**

system 和 tools 字节稳定，messages 保持 append-only，历史 assistant 内容和 tool_result 原样回传，动态数据尽量放后面；连续请求间隔缩短，并用 diagnostics 定位偶发 miss。

## 适用场景

- 多轮对话应用缓存命中率突然下降，需要快速定位是哪类请求参数变化导致。
- 系统提示中插入了时间戳、请求 ID 等动态内容，导致缓存频繁失效时排查根因。
- 工具定义由代码动态生成、顺序不稳定或无确定性序列化时，定位 tools_changed 问题。
- 在回归测试或 CI 中自动检查多轮对话前缀是否保持稳定。

## 标签

`Prompt Caching` `Cache Diagnostics` `Claude API` `LLM 性能优化` `可观测性`
