# Claude 工具调用（Tool Use / Function Calling）机制与实战

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) · 来源: web · 生成时间: 2026-09-20T03:31:10.382869+00:00*

## 背景

大语言模型本身只能根据训练数据生成文本，无法访问实时数据、执行代码或操作系统，很多业务问题需要模型调用外部能力。Tool use/function calling 就是让模型输出结构化的调用指令（工具名和参数），由应用或平台实际执行，再回传结果，从而把模型的推理能力转化为可执行动作。这是构建 agent、检索增强生成和自动化工作流的基础。

## 痛点

如果没有工具调用，模型遇到需要实时信息或外部操作的问题只能编造或拒绝，回复不可靠。要实现自动化，开发者过去只能让模型输出自然语言，再由正则或解析器抽取参数，不仅脆弱，而且无法支持多步骤、可回溯的调用。工具调用把“该执行什么”标准化为可验证的结构化协议。

## 解决办法

核心是把工具定义（name、description、input_schema）注入请求上下文，模型在推理时决定是否调用并生成 tool_use 块；应用检测 stop_reason='tool_use'，按 tool_use_id 执行对应函数，再以 tool_result 回传，模型基于结果生成最终答案。工具按执行位置分为客户端工具（应用自己执行，如数据库、bash）和服务端工具（Anthropic 基础设施执行，如 web_search、code_execution）。可以理解为模型是大脑，只负责决策和参数生成，执行由外部运行时完成，模型和工具通过消息往返形成一个闭环。

## 关键代码示例

```python
import anthropic
client = anthropic.Anthropic()

def get_weather(city):
    return f'{city}: 24C, sunny'

tools = [{
    'name': 'get_weather',
    'description': 'Get current weather for a city',
    'input_schema': {
        'type': 'object',
        'properties': {'city': {'type': 'string'}},
        'required': ['city'],
    },
}]
msg = [{'role': 'user', 'content': 'What is the weather in Paris?'}]
resp = client.messages.create(model='claude-sonnet-4-5', max_tokens=1024, tools=tools, messages=msg)
if resp.stop_reason == 'tool_use':
    tool = next(b for b in resp.content if b.type == 'tool_use')
    result = get_weather(**tool.input)
    resp2 = client.messages.create(
        model='claude-sonnet-4-5', max_tokens=1024, tools=tools,
        messages=msg + [
            {'role': 'assistant', 'content': resp.content},
            {'role': 'user', 'content': [{'type': 'tool_result', 'tool_use_id': tool.id, 'content': result}]},
        ],
    )
    print(resp2.content[0].text)
```

这段代码先定义 get_weather 工具及其 JSON Schema，Claude 在首轮请求中返回 stop_reason='tool_use' 和一个 tool_use 块；应用取出参数执行本地函数，再把结果作为 tool_result 与包含 tool_use 的 assistant 消息一起回传。第二轮请求后模型拿到工具结果生成自然语言答案。这样体现了“模型只下指令、应用执行、结果回填”的核心闭环。

## 关键流程

1. 定义工具：提供 name、description 和 input_schema，描述越准确，模型调用边界越清楚。
2. 发送首轮 Messages API 请求，tools 参数中带上工具定义。
3. 检查响应 stop_reason 是否为 'tool_use'，并解析 content 中的 tool_use 块，获取 tool_use_id 和 input 参数。
4. 应用执行对应客户端工具，或对服务端工具直接读取返回结果；若是并行调用，先收集同一轮所有 tool_use 块。
5. 将执行结果封装为 tool_result，并携带原始 assistant 消息和 tool_use_id 回传，形成第二轮请求。
6. 模型基于 tool_result 生成最终回复；可通过 tool_choice 或系统提示控制调用行为。

## 关键点

- 工具调用本质是模型输出结构化调用指令，而不是模型直接执行代码或访问外部 API；因此执行环境、权限和错误处理都必须由应用或平台负责。
- 工具的 name/description/input_schema 是影响模型何时调用、如何传参的关键，模糊的描述会导致该调不调或参数错误。
- 客户端工具在应用内执行，服务端工具在 Anthropic 基础设施执行；选择时要考虑数据隐私、执行环境、额外费用和实现复杂度。
- 每次工具往返都要把 assistant 消息与对应 tool_use_id 回传，否则模型无法把结果和调用关联，可能产生幻觉或错误答案。
- tool_choice 可以从 auto 调整为 any/tool 强制调用，也可通过系统提示微调触发边界，解决“模型不调用工具”的问题。
- 工具调用的 token 成本包括 tools 参数、tool_use 和 tool_result 内容块，以及模型专属的工具系统提示，预算评估时要把这些计入。

## 对比与权衡

- 相比让模型直接生成自然语言再由正则解析 API 参数，Tool use 通过 JSON Schema 和 tool_use/stop_reason 提供结构化协议，可靠性更高、更易调试，但需要额外的 tokens 和应用执行逻辑。
- 相比客户端工具，服务端工具无需自己实现执行环境与 handler，开发更省事，但数据会离开应用、且可能产生按次或按资源计费，可控性和成本透明度不如客户端工具。
- 相比通过 MCP connector 访问远程 MCP 服务器，自定义工具更轻量、直接，适合少量固定函数；但工具数量多、需要跨进程复用或标准化生态时，MCP 提供更好的发现与治理能力。

## 自测问题

**问: Claude 的工具调用完整流程是怎样的？**

先定义工具 schema 随请求传入；模型在适用时返回 stop_reason='tool_use' 和 tool_use 块，不执行任何副作用；应用解析参数并执行，再把 tool_result 按 tool_use_id 回传；最后模型生成面向用户的答案。可以强调这是 orchestration loop 的基本单位。

**问: auto、any、tool 和 none 几种 tool_choice 有什么区别？**

auto 是默认值，模型根据请求和工具描述自行判断是否调用；any 和 tool 强制至少调用一个工具，适合必须依赖外部能力的任务；none 禁止工具调用。还要说明强制调用会增加 token 成本，且可能因不合适的工具导致低效。

**问: 如何让模型更稳定地调用工具，或不要乱调用工具？**

一方面优化工具 name/description/input_schema，使其与任务边界一致；另一方面在 system prompt 中给出轻量或强约束，例如 'Always call a tool first' 或 'Use your judgment'；需要确定性时用 tool_choice 显式强制或禁止。同时通过日志分析 stop_reason，定位是描述问题还是提示问题。

**问: 多个工具并行调用时要注意什么？**

同一个响应里可能有多个 tool_use 块，应用应识别并执行所有调用，再把每个结果分别以对应 tool_use_id 的 tool_result 返回。若其中某个执行失败，应回传 is_error 或错误内容，而不是中断整个轮次，这样模型可以决定重试或降级。

**问: 工具调用的成本为什么比普通对话高？**

因为输入包含 tools 参数（名称/描述/schema）、对话历史中的 tool_use 和 tool_result 块，且 Anthropic 会额外注入工具专用 system prompt；服务端工具还会按使用量收费。面试中可以量化为 tokens 和额外费用两部分，体现成本意识。

## 适用场景

- 实时信息查询：天气、股价、航班、汇率等训练数据之外的事实性信息。
- 外部系统操作：订单查询、创建工单、CRM/数据库读写等业务 API 集成。
- 代码执行与数据分析：沙箱运行 Python/SQL 生成报表、图表或验证代码。
- 多步 Agent 工作流：搜索网页、抓取页面、执行代码、再汇总答案的自动化链路。

## 标签

`Claude` `Tool Use` `Function Calling` `Agent` `MCP`
