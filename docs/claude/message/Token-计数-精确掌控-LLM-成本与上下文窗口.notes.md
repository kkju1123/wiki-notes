# Token 计数：精确掌控 LLM 成本与上下文窗口

*原文: [https://platform.claude.com/docs/en/build-with-claude/token-counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) · 来源: web · 生成时间: 2026-09-20T03:10:05.014056+00:00*

## 背景

LLM 以 token 而非字符或单词为单位处理文本，并按 token 计费、以 token 限制上下文窗口。开发者需要在调用前后准确知道 token 数，才能做成本预估、越界保护和 prompt 设计。Claude 等平台因此提供专用计数接口，避免开发者自行猜测或引入不一致的分词器。

## 痛点

不懂 token 计数会把 token 数简单按单词或字符估算，导致费用超预期、请求因超出上下文窗口被拒绝、或 prompt 被截断。缺少精确预检还会让缓存和批量任务难以优化。

## 解决办法

LLM 使用 BPE 等分词器把文本切成子词 token，常见词可能一个 token，长词或罕见词可能多个。Claude 通过 count_tokens API/SDK 在调用前计算输入 token；调用后通过 response.usage 返回实际 input_tokens/output_tokens，部分模型还返回缓存读写 token。类比：像寄快递前先称重，而不是按包裹大小或页数猜运费。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

# 调用前：只计算 token，不触发模型推理
estimate = client.messages.count_tokens(
    model='claude-sonnet-4-5',
    messages=[{'role': 'user', 'content': 'Explain token counting.'}],
)
print('estimated input tokens:', estimate.input_tokens)

# 调用后：从响应中读取真实用量
resp = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=256,
    messages=[{'role': 'user', 'content': 'Explain token counting.'}],
)
print(resp.usage.input_tokens, resp.usage.output_tokens)
```

第一段调用 count_tokens，只对 messages 做分词计算，返回预估输入 token，适合请求前成本与窗口预检。第二段通过 create 发起真实推理，usage 返回实际的 input_tokens 和 output_tokens，是计费和监控的权威数据。两处都指定同一个 model，确保 tokenizer 与计费模型一致。

## 关键流程

1. 确定目标模型和完整的 messages 结构
2. 调用 count_tokens 或对应 API 预检输入 token 数
3. 将输入 token 与模型上下文窗口比较，预留输出 token 空间
4. 实际发起模型调用，并从 response.usage 读取真实 input/output token
5. 按模型单价分别计算常规输入、输出和缓存相关 token 的成本

## 关键点

- token 是 BPE 子词单元，不是字符或单词；同样文本在不同模型下 token 数可能不同，因此必须用目标模型对应的计数方式。
- count_tokens 在不产生推理费用的情况下返回输入 token 数，适合请求前预检和上下文窗口保护。
- 实际调用后的 usage.input_tokens 与 output_tokens 是计费与监控的权威数据，应始终记录。
- 启用 prompt caching 时，usage 可能包含 cache_creation 和 cache_read 相关 token，它们按不同单价计费。
- 不要用固定比例（如 1 token = 0.75 词）做精确成本决策，只适合粗略估计。

## 对比与权衡

- 相比本地 tiktoken 等分词库，Claude 的 count_tokens API 能保证与官方计费一致的模型 tokenizer，但需要网络请求并可能受 API 波动影响。
- 相比直接使用 create 后的 usage 数据，预调用 count_tokens 可以在请求前发现超限风险，但不能反映输出 token，也无法替代实际 usage。
- 相比 OpenAI 的 tiktoken/Token API，Claude 的 count_tokens 集成在 Messages API 路径中，使用上更针对对话消息，但模型覆盖范围相对更聚焦 Claude。

## 自测问题

**问: token 计数为什么不能按字符或单词估算？**

现代 LLM 使用 BPE/WordPiece 等子词分词，高频短词可能 1 token，复杂或罕见词可能被拆成多个子词；不同模型词表不同。精确计数影响费用、窗口和截断判断，因此必须调用官方接口或对应 tokenizer。

**问: 如何在不调用模型的情况下知道请求是否会超过上下文窗口？**

使用 count_tokens 或类似端点传入完整 messages，返回 input_tokens；与模型 context_window 比较，还要预留 max_tokens 输出空间。若接近上限，可压缩历史、摘要或使用 prompt caching。

**问: usage 里 input_tokens 和 cache_creation_input_tokens 有什么区别？**

input_tokens 是常规未缓存输入；cache_creation_input_tokens 是首次写入缓存的 token，通常比普通输入单价更高；cache_read_input_tokens 是命中缓存的 token，单价远低于普通输入。需要按字段分别计费。

**问: 不同模型 tokenizer 相同吗？为什么 count_tokens 必须指定 model？**

不同模型可能使用不同词表和分词规则，虽然同厂商可能共享大部分，但版本升级会变化。指定 model 确保计算与计费模型一致。

**问: 怎样设计一个成本监控系统？**

在每次请求前调用 count_tokens 记录预估，请求后记录 usage 的各 token 字段，按模型单价聚合；设置预算告警和上下文窗口保护；对于长会话主动触发压缩或摘要。

## 适用场景

- 构建对话产品时，在用户输入提交前估算 token 数，防止超过模型上下文窗口。
- 批量推理或数据处理任务中，按 token 预估总成本并选择合适模型。
- 长文档问答应用中，诊断 prompt 是否过大，决定是否切块或压缩。
- API 成本监控与多租户计费系统，按 usage 字段精确记账。

## 标签

`LLM` `Token 计数` `Anthropic Claude` `API 成本` `上下文管理`
