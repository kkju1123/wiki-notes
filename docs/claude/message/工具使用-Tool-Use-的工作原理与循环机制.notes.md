# 工具使用（Tool Use）的工作原理与循环机制

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works) · 来源: web · 生成时间: 2026-09-20T03:33:22.793978+00:00*

## 背景

大语言模型本质上是文本生成器，无法直接执行代码、查询数据库或获取训练数据之外的信息。为了让模型在真实应用中完成有副作用的操作和实时数据获取，工具使用（Tool Use）机制被引入。它将模型从只能说话扩展为可以调用函数，形成了应用与模型之间的执行契约，使工程师能以类似调用类型化接口的方式集成 LLM。

## 痛点

没有工具使用，模型无法完成发邮件、写文件、查数据库等任务，因为模型只能描述而不能执行。开发者在集成时若不清楚工具的执行位置和循环机制，容易误以为模型会自动运行代码，导致工具未被调用、结果丢失或陷入死循环。

## 解决办法

工具使用通过契约实现：应用定义工具的 schema（名称、描述、输入输出格式），模型根据对话上下文决定是否调用以及调用哪个工具，并输出结构化的 tool_use 请求。应用代码负责执行实际操作，并把结果以 tool_result 返回，模型再继续推理。执行位置分三类：用户定义工具和 Anthropic-schema 工具在客户端执行，需要应用驱动一个 while 循环；服务器执行工具由 Anthropic 托管运行，应用只需启用并读取结果。可以把模型比作经理，工具比作员工——经理下达指令，员工执行并汇报，经理根据结果继续决策。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
messages = [{'role': 'user', 'content': '查询北京天气'}]
tools = [{'name': 'get_weather', 'description': '获取城市天气', 'input_schema': {'type': 'object', 'properties': {'city': {'type': 'string'}}, 'required': ['city']}}]

def execute_tool(name, args):
    return args['city'] + '：晴，25°C' if name == 'get_weather' else None

while True:
    resp = client.messages.create(model='claude-sonnet-4-5', max_tokens=1024, tools=tools, messages=messages)
    messages.append({'role': 'assistant', 'content': resp.content})
    if resp.stop_reason == 'tool_use':
        for block in resp.content:
            if block.type == 'tool_use':
                result = execute_tool(block.name, block.input)
                messages.append({'role': 'user', 'content': [{'type': 'tool_result', 'tool_use_id': block.id, 'content': result}]})
    else:
        break
```

这段代码展示了客户端工具循环（agentic loop）的核心：先定义工具 schema，然后进入 while 循环，根据 stop_reason 判断是否继续。模型返回 tool_use 时，应用提取块并执行本地函数 execute_tool，将结果包装成 tool_result 后追加到消息列表，再次请求模型。循环直到 stop_reason 不再是 tool_use（如 end_turn），此时模型给出最终答复。

## 关键流程

1. 发送带 tools 数组和用户消息的请求。
2. 检查响应 stop_reason 是否为 tool_use。
3. 提取所有 tool_use 块，执行对应工具函数。
4. 将每个执行结果格式化为 tool_result 块，包含 tool_use_id 和输出内容。
5. 把 assistant 响应和 tool_result 消息追加到对话，重新发送请求。
6. 重复直到 stop_reason 为 end_turn、max_tokens 等，循环退出。

## 关键点

- 工具使用是应用与模型之间的契约：模型只产出结构化的 tool_use 请求，实际执行由应用代码或 Anthropic 服务器完成。
- 工具按执行位置分为用户定义、Anthropic-schema 和服务器执行三类，不同的位置决定了应用需要参与循环的程度。
- 客户端工具需要应用驱动 while 循环，以 stop_reason 作为循环条件，这是集成时的关键控制点。
- 服务器端工具在 Anthropic 内部循环执行，应用通常只需启用并读取最终结果，但可能遇到 pause_turn 需要继续。
- 工具适用于有副作用或需要外部数据、专业计算的任务，不应替代模型擅长的纯文本推理和常识问答。

## 对比与权衡

- 相比纯文本生成，工具使用能执行动作和获取实时数据，但会增加延迟、成本和错误处理复杂度，因此在简单问答场景下不应使用。
- 相比用户自定义工具，Anthropic-schema 工具（如 bash、text_editor）因为经过大量训练轨迹优化，调用更可靠、错误恢复更优雅，但灵活性受限，只能用于预定义操作。
- 相比客户端执行工具，服务器执行工具（如 web_search、code_execution）减少了应用开发和运维负担，但应用对执行过程的控制和透明度较低，且服务器内部迭代有上限可能返回 pause_turn。

## 自测问题

**问: 工具调用过程中 stop_reason 可能有哪些值？分别应该怎么处理？**

常见值包括 tool_use、end_turn、max_tokens、stop_sequence 和 refusal。tool_use 表示需要执行工具并继续循环；end_turn 表示正常结束；max_tokens 需要增加最大 token 数或截断后继续；stop_sequence 表示命中自定义停止序列，应检查业务逻辑；refusal 表示模型出于安全拒绝，需要处理或反馈用户。

**问: 客户端工具和服务器端工具的循环机制有何本质区别？**

客户端工具需要应用显式驱动 while 循环，每轮工具调用都是一次往返；服务器端工具在 Anthropic 基础设施内自动运行多次工具调用，直到得出最终答案或达到迭代上限。服务器端循环可能返回 pause_turn 表示未完成，需要应用重新发送对话继续。

**问: 为什么 Anthropic-schema 工具比自定义工具更可靠？**

这些工具的 schema 是模型训练数据中大量成功轨迹的一部分，模型对这些工具的参数和错误恢复有更强的先验知识。自定义工具则需要开发者提供清晰描述、参数说明和示例，否则模型可能误用或无法从错误中恢复。

**问: 在什么情况下不应该使用工具？**

当任务仅依赖模型内部知识、不需要外部数据或副作用时，应直接使用文本生成。例如通用知识问答、文本摘要、开放式推理等，使用工具会增加延迟、成本和潜在错误，且可能降低回答质量。

**问: 如何处理并行工具调用？**

模型可以在一个响应中返回多个 tool_use 块。应用应并发执行这些工具（如果它们之间没有依赖），然后将所有 tool_result 一次性返回。每个 tool_result 必须携带对应的 tool_use_id，以便模型正确关联结果。

## 适用场景

- 需要实时数据：如查询天气、股价、新闻、数据库内容。
- 需要执行有副作用的操作：如发送邮件、创建工单、写入文件或更新 CRM 记录。
- 构建自主代理：如编程助手、浏览器自动化、桌面控制等需要多步工具调用的场景。
- 与企业内部 API 集成：将 LLM 接入现有系统，通过工具调用完成业务流程操作。

## 标签

`工具使用` `Agent` `Claude API` `函数调用` `LLM 集成`
