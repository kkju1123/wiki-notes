# Advisor Tool：执行器与顾问模型分离的成本/智能平衡方案

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) · 来源: web · 生成时间: 2026-09-20T03:53:28.350779+00:00*

## 背景

长程 agentic 工作负载（如编码代理、计算机使用、多步研究）中大部分 token 是机械执行，但关键决策（架构、纠偏、下一步计划）需要高智能。传统单一模型方案要么全用大模型导致成本过高，要么只用小模型导致关键决策不可靠。Advisor tool 将“规划”与“执行”拆成两个模型，让低成本执行器在生成中途获得高智能顾问的指导。

## 痛点

没有 advisor tool 时，开发者必须在成本与智能之间二选一：全用大模型成本高、延迟大；只用小模型则在长程复杂任务中容易迷航，关键决策错误会拖垮整个任务。同时，要在单个请求内获得中途指导需要额外编排或多轮往返，工程复杂度高。

## 解决办法

将 advisor 作为特殊服务端工具加入 executor 的 tools 数组，executor 在生成过程中自主决定何时调用。调用时 input 为空，服务器自动将完整对话 transcript（system prompt、工具定义、历史、当前生成文本）作为上下文，运行 advisor 模型子推理；advisor 返回建议文本作为 advisor_tool_result，executor 读取后继续生成。整个过程在同一个 /v1/messages 请求内完成，无客户端额外 round trip。类比：执行者遇到关键决策就敲旁边专家顾问的门，专家看完整记录给建议，执行者继续干活。费用按 advisor 模型的子推理计费，但大部分 token 由 executor 生成，因此总成本可控。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        'type': 'advisor_20260301',
        'name': 'advisor',
        'model': 'claude-opus-4-8',   # 返回明文 advisor_result
        'max_uses': 5,
        'max_tokens': 2048,
        'caching': {'type': 'ephemeral', 'ttl': '5m'}
    }
]

response = client.messages.create(
    model='claude-sonnet-4-5',  # 执行器模型
    max_tokens=4096,
    system='You are a coding agent...',
    messages=[{'role': 'user', 'content': 'Refactor the authentication module...'}],
    tools=tools,
)

for block in response.content:
    if block.type == 'server_tool_use' and block.name == 'advisor':
        print('executor called advisor')
    elif block.type == 'advisor_tool_result':
        print('advisor guidance received')
```

这段代码定义了一个 advisor 工具，指定顾问模型为 claude-opus-4-8（返回明文建议），并设置 max_uses=5、max_tokens=2048、启用 ephemeral 缓存。executor 模型 claude-sonnet-4-5 在生成过程中可自行决定何时调用 advisor；响应中会出现 server_tool_use 和 advisor_tool_result 块，执行器读取建议后继续完成回复。整个咨询过程在单个 messages.create 请求内完成。

## 关键流程

1. 在 tools 数组中定义 advisor 工具，指定 advisor 模型及可选的 max_uses、max_tokens、caching。
2. executor 在生成过程中自行判断并发出 server_tool_use 调用 advisor，input 为空。
3. 服务器后台运行 advisor 模型子推理，自动传入完整对话 transcript 作为上下文。
4. advisor 返回 advisor_tool_result 块给 executor（明文或加密）。
5. executor 读取建议后继续生成剩余内容，整个流程在单个 /v1/messages 请求内闭环。

## 关键点

- advisor tool 让 executor 在生成中途按需获取高层战略指导，大部分 token 仍由低成本模型生成，实现成本与智能的分离优化。
- 调用由 executor 自主决定，input 固定为空，服务器自动构建完整 transcript，客户端无需手动传递上下文。
- max_uses 和 max_tokens 是关键成本控制参数：超过 max_uses 后返回错误块，executor 继续执行；max_tokens 限制 advisor 子推理总输出。
- 不同 advisor 模型返回不同结果变体：claude-opus-4-8 返回明文 advisor_result，claude-opus-5 返回加密 advisor_redacted_result；加密结果客户端不可读但 executor 可读。
- advisor 本身无工具、无上下文管理，thinking blocks 被丢弃，仅建议文本返回给 executor，降低子推理开销。

## 对比与权衡

- 相比所有 turn 都用 advisor 模型（advisor solo），本方案在大部分机械执行上使用 executor 模型，成本显著降低、延迟可能更低，但在关键决策质量上略低于纯大模型。
- 相比完全不用 advisor 只用 executor，本方案在复杂长程任务中的规划和纠偏能力更强，但会增加 advisor 子推理的额外成本和可能的调用次数上限。
- 相比普通客户端 tool use（需要客户端处理工具结果并多轮往返），advisor 是服务端工具，整个咨询过程在单次请求内完成，无需额外 round trip。
- 相比静态 prompt caching/context engineering，advisor 提供动态、按需的规划能力，但需要配合 max_uses/max_tokens 控制成本。

## 自测问题

**问: executor 模型如何决定何时调用 advisor？**

executor 将 advisor 视为普通工具，根据当前任务复杂度、自身不确定性或长程规划需求自行决定调用；调用时 input 为空，服务器自动提供 transcript。训练/对齐使模型学会在合适的时机触发，类似 React 模式中的工具选择。

**问: advisor tool 的成本如何计算和控制？**

advisor 子推理按 advisor 模型的 token 计费，包含 thinking 和 text；通过 max_uses 限制单请求调用次数，max_tokens 限制每次输出，caching 可缓存 advisor 的 transcript 以降本；达到 max_uses 后错误块返回，executor 继续执行。

**问: 为什么 claude-opus-5 返回加密的 advisor_redacted_result，而 claude-opus-4-8 返回明文？**

加密变体防止客户端直接读取建议文本，但仍可供服务端 executor 使用；明文变体便于调试和审计。选择时需根据模型兼容性和业务合规要求。

**问: 哪些工作负载不适合用 advisor tool？**

单轮问答无需规划；用户已经自行选择成本/质量权衡的纯透传模型选择器；每一轮都需要 advisor 模型完整能力的场景，这些情况下 advisor 的额外子推理不会带来收益或反而增加成本。

**问: 在单个请求内 advisor 如何被调度？如果执行中途暂停怎么办？**

advisor 由 executor 在生成过程中通过 server_tool_use 触发，服务端同步运行子推理并返回结果，整个流程在一个 /v1/messages 请求内。若 turn 中途暂停，需要后续请求 resume，遵循 API 的暂停恢复机制。

## 适用场景

- 编程代理：长时间重构或多文件编辑中，执行器处理机械编辑，顾问在架构决策或错误恢复时提供指导。
- 计算机使用代理：多步 UI 操作中，executor 执行普通点击/输入，顾问在遇到异常或需要调整路径时给出纠偏建议。
- 多步研究流水线：低成本执行器负责搜索、提取、汇总，顾问制定研究计划和判断信息充分性。
- 成本敏感的大规模 agent 部署：用 Haiku 做执行器、Opus/Fable 做顾问，获得比纯 Haiku 高的智能，成本低于纯大模型。

## 标签

`Advisor Tool` `Claude API` `Agentic AI` `Cost Optimization` `Tool Use`
