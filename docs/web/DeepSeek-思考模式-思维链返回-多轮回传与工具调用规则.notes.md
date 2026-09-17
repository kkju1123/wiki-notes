# DeepSeek 思考模式：思维链返回、多轮回传与工具调用规则

*原文: [https://api-docs.deepseek.com/zh-cn/guides/thinking_mode](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) · 来源: web · 生成时间: 2026-09-17T02:54:24.444036+00:00*

## 背景

随着大模型在数学证明、代码生成和复杂决策等任务上要求更高准确率，直接一次性生成答案容易在中间步骤出错。DeepSeek 等推理模型在最终回答前先进行思维链推理，再输出精简答案，显著提升复杂任务表现。API 因此需要重新定义思维链的可见性、上下文拼接和采样参数规则。

## 痛点

不懂思考模式规则时，开发者容易在多轮对话中丢失推理上下文，尤其在工具调用场景少回传 reasoning_content 会导致 400 错误。误用 temperature、top_p 等参数也会造成调参不生效，难以优化和复现模型行为。

## 解决办法

请求侧通过 thinking.type 开关和 reasoning_effort 控制推理强度，响应侧返回 reasoning_content 与 content 两个同级字段。无 tools 的普通多轮中，服务端不拼接历史思维链以节省 token，客户端无需回传；携带 tools 时，思维链是工具调用推理状态的一部分，必须随 assistant 消息完整回传，否则会报 400。采样上采用更确定策略：temperature 等惩罚项被忽略，top_p 下限为 0.95，从而保持思维链稳定。

## 关键代码示例

```python
from openai import OpenAI
import json

client = OpenAI(api_key='<key>', base_url='https://api.deepseek.com')
messages = [{'role': 'user', 'content': '杭州明天天气怎么样？'}]
tools = [{'type': 'function', 'function': {'name': 'get_weather', 'parameters': {'type': 'object', 'properties': {'location': {'type': 'string'}}}}}]

while True:
    resp = client.chat.completions.create(
        model='deepseek-flash', messages=messages, tools=tools,
        reasoning_effort='high', extra_body={'thinking': {'type': 'enabled'}},
    )
    msg = resp.choices[0].message
    messages.append(msg)  # 关键：携带 tools 时必须回传 reasoning_content

    if not msg.tool_calls:
        print(msg.content)
        break
    for tc in msg.tool_calls:
        messages.append({'role': 'tool', 'tool_call_id': tc.id, 'content': '{}'})
```

这段代码演示工具调用场景下最容易出错的部分：因为请求携带 tools，历史轮次的 reasoning_content 必须随 assistant 消息回传。直接 append 整个 message 对象等价于同时回传 content、reasoning_content 和 tool_calls；如果只回传 content 或 tool_calls 会触发 400。循环中每次工具执行后以 tool 角色加入结果，直到模型不再发起 tool_calls 后输出最终答案。

## 关键流程

1. 开启思考模式：OpenAI 格式可通过 extra_body 传 thinking.type=enabled，并用 reasoning_effort 设置强度；默认已开启且 effort=high。
2. 从响应中分别读取 message.reasoning_content 和 message.content，思维链与最终回答同级返回。
3. 无 tools 的普通多轮对话：不要回传 reasoning_content，服务端也不把它拼进上下文，只正常追加 assistant/content 消息。
4. 携带 tools 的请求：每轮必须完整回传上一轮 assistant 消息中的 reasoning_content，可直接 append 整个 message 对象。
5. 根据产品策略决定是否向最终用户展示 reasoning_content，但 API 调用链路必须按规则保留该字段。

## 关键点

- 思考模式默认开启且 effort 默认为 high；用户传入 minimal/low 会映射为 low，medium/high/xhigh 映射为 high，max/ultra 映射为 max。
- 在思考模式下 temperature、presence_penalty、frequency_penalty 不生效；top_p 虽生效但会被抬升到至少 0.95，非思考模式 top_p 恒为 1.0。
- reasoning_content 与 content 同级返回，是否回传取决于请求是否携带 tools，而不是客户端偏好。
- 工具调用模式下，reasoning_content 是推理状态的一部分，缺少它会导致 400 错误，因此应直接 append 整个 assistant message 对象。
- 无工具调用时，API 不拼接历史思维链，这能节省上下文 token，客户端也不应把 reasoning_content 塞回上下文，因为它会被忽略。
- 思考模式适合需要多步推理或工具决策的任务，开发者需要额外关注思维链的 token 成本、延迟和泄露风险。

## 对比与权衡

- 相比 OpenAI 一些推理模型默认只返回思维链摘要或不暴露完整推理 token，DeepSeek 将 reasoning_content 作为普通字段返回，开发者调试和多轮衔接更直接，但需要开发者自己管理回传规则和 token 成本。
- 相比普通聊天模式，思考模式牺牲了 temperature、频率惩罚等采样灵活性，并增加思维链 token 成本，但在数学推理、代码生成等复杂任务上准确性更高。

## 面试可能会问

**问: 思考模式下为什么 temperature、presence_penalty、frequency_penalty 不生效？**

这些参数会引入随机性或改变 token 分布，容易打断长思维链的一致性；API 为了兼容旧客户端不报错但静默忽略。可以扩展：推理模型通常采用更确定的采样策略，top_p 也会被抬升到 0.95 以上。

**问: 多轮对话中 reasoning_content 到底要不要回传？**

关键看是否携带 tools。无 tools 时回传也会被忽略，不拼上下文；有 tools 时必须回传，因为工具调用是多步推理的一部分，缺少会导致 400。实践中可直接 append 整个 assistant message。

**问: DeepSeek 思考模式的 effort 参数是如何映射的？**

请求传入 minimal/low 映射为 low，medium/high/xhigh 映射为 high，max/ultra 映射为 max，默认是 high。不同强度主要影响思维链长度、延迟和 token 消耗。

**问: 思维链内容是否应该展示给终端用户？**

通常只展示 content，不展示 reasoning_content。但 API 返回它可以用于调试、审计和工具调用衔接；需要注意思维链可能包含敏感中间推导，产品层应做权限控制或脱敏。

**问: 为什么非思考模式 top_p 恒为 1.0？**

这是 DeepSeek 服务端的采样策略，非思考模式下为了解码稳定和兼容性忽略传入的 top_p；思考模式下仅限制下限 0.95，避免过度随机破坏推理链。

## 适用场景

- 数学证明、数值比较、代码生成等需要多步推理的任务，优先开启思考模式。
- 需要模型先推理再调用工具/函数的多轮 Agent 场景，如天气查询、数据库查询、RAG 检索决策。
- 开发者需要调试或审计模型中间推理过程，可利用 reasoning_content 分析模型决策。
- 需要高准确率且可接受更高延迟和 token 成本的生产问答或自动化流程。

## 标签

`DeepSeek` `推理模型` `思维链` `工具调用` `API设计`
