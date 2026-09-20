# Claude 拒绝响应与降级回退机制（Refusals and Fallback）

*原文: [https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback) · 来源: web · 生成时间: 2026-09-20T02:21:09.367181+00:00*

## 背景

大模型商用必须内置安全策略以避免有害输出，但安全分类器难免误伤部分良性请求，且开发者希望用户请求尽量不被直接拒绝。Claude Fable/Opus 把安全拒绝从模型文本中结构化出来，通过 stop_reason 和 stop_details 暴露给调用方，使应用可以自动降级而不是解析文本猜测拒绝原因。相比早期在回复里查找“我无法协助”的脆弱做法，这是一种可编程、可路由的拒绝信号。

## 痛点

如果不理解 refusal 是 HTTP 200 且 stop_reason=refusal，开发者容易把被拒响应当成正常完成，或误判为 API 错误导致解析失败。没有自动 fallback 时，被误伤的请求会直接拒绝，损害用户体验。自行盲重试还可能重复扣 prompt cache 费用，尤其长上下文场景成本显著。

## 解决办法

核心是把“安全判断”和“模型生成”分离：分类器先判定请求命中的安全策略类别，然后 API 或客户端根据 category 选择更合适的模型重试。服务端 fallback 在单次 API 调用内完成整条链，default 模式使用 Anthropic 维护的按类别推荐路由；SDK 中间件在客户端透明重试；手动重试则检查 stop_details.category，配合 fallback credit 复用 prompt cache，避免二次计费。可以理解为：拒绝不是终点，而是一个携带路由信息的分流信号。

## 关键代码示例

```python
import requests

resp = requests.post(
    'https://api.anthropic.com/v1/messages',
    headers={
        'x-api-key': 'ANTHROPIC_API_KEY',
        'anthropic-beta': 'server-side-fallback-2026-07-01',
    },
    json={
        'model': 'claude-fable-5.1',
        'max_tokens': 1024,
        'messages': [{'role': 'user', 'content': '...'}],
        'fallbacks': 'default',
    },
)
data = resp.json()
print(data.get('model'))
if data.get('stop_reason') == 'refusal':
    print(data['stop_details']['category'])
```

这段代码用原生 HTTP 调用 /v1/messages，传 beta header 和 fallbacks='default'，让服务端在安全拒绝时于单次请求内完成重试。调用后不依赖异常，而是检查 stop_reason，若仍为 refusal 才说明最终被拒。data['model'] 可以显示实际提供回答的模型，方便判断是否发生了回退。

## 关键流程

1. 读取响应时先判断 stop_reason 是否为 'refusal'，而不是只看 HTTP 状态码；refusal 是 HTTP 200。
2. 解析 stop_details.category 与 explanation，记录策略类别；如果已经产生 partial output，应丢弃这些不完整内容。
3. 根据运行环境选择回退方式：Claude API 简易场景用服务端 fallbacks='default'；SDK 场景配置中间件；原生 HTTP 或自定义逻辑采用手动重试。
4. 若使用服务端 fallback，在请求中传 fallbacks='default' 和 beta header，或者传最多三个指定的 fallback 模型。
5. 检查最终响应的 top-level model 字段和 usage.iterations 中的 fallback_message，确认是否发生回退以及由哪个模型实际服务。
6. 手动 retry 时使用 fallback credit 机制复用 prompt cache，避免首次调用和重试重复计费。

## 关键点

- stop_reason='refusal' 是正常的 HTTP 200 响应，不是网络或 API 错误；这种设计让安全拒绝可以被程序稳定识别，而不必解析模型文本。
- stop_details.category 是分类策略类别，explanation 仅供展示不可稳定解析；category 和 explanation 为 null 时也是合法的永久值。
- 服务端 fallback 通过 fallbacks='default' 在单次 API 调用内完成安全拒绝的路由重试，适合最简单的业务接入，但它只对安全分类器触发的拒绝生效，限流或过载不会触发回退。
- 手动重试需要先丢弃 partial output，再根据 recommended_model 或自己的模型链执行重试，并配合 fallback credit 避免 prompt cache 二次计费。
- recommended_model 只是提示而不是保证；当 API 跳过 fallback 尝试，例如 fallback 模型被限流时，它会返回一个可重试模型名，业务侧应把它作为候选而非唯一路由依据。
- 指定 fallback 模型列表最多三个，可以精确控制哪些模型承接被拒请求，例如只使用自己合规验证过的模型，但这需要自行维护 Anthropic 默认推荐的变化。

## 对比与权衡

- 相比服务端 fallback，SDK 中间件在客户端配置一次即可跨平台使用，但可能无法享受单次请求内服务端路由的全部优化，且依赖 SDK 版本支持。
- 相比服务端 fallback 和 SDK 中间件自动应用 fallback credit，手动重试控制力最强，可以自定义重试链、记录审计日志，但需要自己处理 prompt cache 计费，复杂度和出错率更高。
- 相比 default routing 的 Anthropic 推荐模型，指定 fallback 模型列表可以稳定路由到应用已验证的模型，但需要自行维护推荐变化，可能选到不够优的模型。

## 自测问题

**问: refusal 和普通 API 错误怎么区分？**

refusal 是 HTTP 200 响应，只有 stop_reason 为 'refusal'；普通 API 错误通常非 200 或返回 error 字段。出现 partial output 时也要以 stop_reason 为准，一旦为 refusal 就丢弃已输出的不完整内容。

**问: fallbacks='default' 内部做了什么？**

它把被拒请求在同一 API 调用内路由到 Anthropic 按 refusal category 推荐的 fallback 模型。分类器先给出 category，服务端查 per-model/per-category 路由表，可能回退到较弱模型或仍拒绝。最终响应中的 model 字段会变成实际服务的模型，并可能带 fallback_message。

**问: 为什么手动重试要关注 fallback credit？**

如果第一次请求已经写入 prompt cache，第二次重试可能命中相同 prompt 前缀，但不同模型或参数下可能产生重复计费；fallback credit 机制允许在 fallback 复用缓存时减免首轮成本。自己裸重试容易多付一次 prompt cache 写或读成本，长上下文场景尤其明显。

**问: stop_details.recommended_model 一定可靠吗？**

不可靠。它只是在 API 跳过 fallback 尝试时给出的可重试提示，可能因为 fallback 模型限流等原因触发；业务侧应把它作为候选而非唯一路由依据，并且需要校验模型可用性。

**问: 安全分类器误伤良性生命科学或网络安全请求怎么办？**

这些类别明确说明良性工作也可能触发。最好先记录 category 和 explanation，然后自动 fallback 到该类别推荐模型。若默认路由不满足业务要求，可指定自己的模型列表；不要试图绕过安全策略，而是用合规模型继续服务用户。

## 适用场景

- 面向终端用户的产品中，对 Claude Fable/Opus 的拒绝请求自动降级到其他模型，减少直接拒绝带来的体验损失。
- 内容生成平台需要统一审计安全类别和回退模型，便于合规追踪与运营分析。
- 使用原生 HTTP 但没有 SDK 中间件的服务，需要自己实现带 fallback credit 的 retry 逻辑。
- 多模型路由或网关架构中，基于 stop_details.category 做可观测性、策略路由和安全审计。

## 标签

`Claude API` `安全分类器` `fallback` `stop_reason` `LLM 安全`
