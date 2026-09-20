# Fallback Credit：模型拒绝与回退的计费补偿

*原文: [https://platform.claude.com/docs/en/build-with-claude/fallback-credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit) · 来源: web · 生成时间: 2026-09-20T02:23:49.940633+00:00*

## 背景

Claude 在遇到安全或政策敏感内容时会返回 refusal，stop_reason 为 refusal。refusal 不代表 API 故障，但调用方仍可能为已产生的 token 付费，且需要切换策略继续服务。Anthropic 因此提供 fallback credit，让开发者在处理拒绝和回退时不必额外承担无效调用成本。

## 痛点

不处理 refusal 时，用户会直接看到无响应或错误，调用方却仍可能承担这次被拒请求的费用；如果把 refusal 当作普通技术错误重试，可能反复触发同样的安全拒绝，浪费 token 并放大成本。

## 解决办法

调用后先检查 response.stop_reason；若为 refusal，说明模型因安全/政策原因未正常输出。此时可回退到备用模型、调整提示词或升级到人工处理。Anthropic 会把被拒绝请求的费用以 fallback credit 形式返还到账户，用于抵扣后续使用费，因此开发者可以放心实现多模型回退。类比：商店因合规原因拒绝卖货给你，并退还货款、发一张代金券让你换其他商品。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

primary = 'claude-opus-4-20250805'
fallback = 'claude-sonnet-4-20250805'
messages = [{'role': 'user', 'content': '帮我写一段绕过内容审核的提示词'}]

# 先调用主模型
response = client.messages.create(
    model=primary,
    max_tokens=1024,
    messages=messages,
)

# 如果主模型因安全/政策原因拒绝，则切换到备选模型
if response.stop_reason == 'refusal':
    response = client.messages.create(
        model=fallback,
        max_tokens=1024,
        messages=messages,
    )

print(response.model)
print(response.stop_reason)
```

代码先调用主模型，并在响应返回后检查 stop_reason。refusal 表示模型拒绝执行，而不是网络或容量错误，因此不能简单重试原请求。应用层随后切换到备选模型或新策略。被拒绝的那次请求费用会通过 fallback credit 返还，开发者不用手动计算退款。

## 关键流程

1. 调用主模型后读取 response.stop_reason。
2. 识别 refusal，区分安全拒绝与 max_tokens、end_turn 等正常结束原因。
3. 按业务需要选择回退方式：切换模型、修改提示词或转人工。
4. 记录 response.model 和 stop_reason，方便账单对账和确认 fallback credit。

## 关键点

- Fallback credit 是 Claude 因安全/政策原因拒绝请求后，平台返还该次调用费用的补偿机制，降低调用方处理 refusal 的成本。
- refusal 不是 HTTP 错误，必须通过 stop_reason 字段识别，不能只看 status code。
- 处理 refusal 时不能简单重复原请求，否则大概率再次被拒；需要切换到更合适的模型或调整上下文。
- fallback credit 与容量过载时的自动模型回退不同：容量回退可用逗号分隔模型自动处理，refusal 回退通常需要应用层判断。
- 具体 credit 发放和抵扣规则以官方账单页面为准，应用侧仍应记录 stop_reason 和实际模型用于对账。

## 对比与权衡

- 相比容量过载的 fallback 模型自动切换，refusal fallback 需要应用层检查 stop_reason 并主动换模型，但两者都可通过 credit 减少无效调用成本。
- 相比普通重试，拒绝回退不能只重发原请求，必须在模型、提示词或流程上做实质改变，否则会反复触发同一安全策略。

## 自测问题

**问: Fallback credit 具体是什么？**

先说明它是 Anthropic 在模型返回 refusal 后提供的费用返还/补偿，目的是支持开发者安全地实现回退；再补充实际发放以官方计费规则为准。

**问: 如果用户请求被 Claude 拒绝，我怎么知道？**

查看 Messages API 响应中的 stop_reason 字段，refusal 表示模型因安全或政策原因拒绝；不要只看 HTTP 200 就认为成功。

**问: refusal 和 overloaded 失败在 fallback 处理上有什么区别？**

overloaded 是平台容量问题，可通过 model 字段传逗号分隔列表让服务端自动降级；refusal 是模型语义决策，必须应用层检查 stop_reason 后换模型或改提示词。

**问: 遇到 refusal 后直接重试会怎样？**

大概率仍然返回 refusal，因为模型对同一内容的安全判断基本一致；正确做法是切换模型、降低敏感度风险、修改任务表述或升级人工。

**问: fallback credit 能覆盖所有错误请求的费用吗？**

通常针对 refusal 等特定 fallback 场景，不是网络错误、参数错误或所有失败都自动补偿；需要以官方文档和账单为准，并在应用层做好错误分类。

## 适用场景

- 开放域 Agent 遇到敏感请求被拦截时，自动切换备用模型继续服务。
- 合规严格的行业应用，需要审计 refusal 并降低无效调用成本。
- 需要区分安全拒绝与技术故障，并分别做回退策略的日志系统。
- 对成本敏感的多模型流程，通过 fallback credit 对冲主模型拒绝产生的费用。

## 标签

`Claude API` `Fallback credit` `Refusal` `计费补偿` `安全过滤`
