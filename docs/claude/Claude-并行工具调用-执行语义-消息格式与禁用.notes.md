# Claude 并行工具调用：执行语义、消息格式与禁用

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use) · 来源: web · 生成时间: 2026-09-20T03:40:11.844377+00:00*

## 背景

在大模型 Agent 中，一次对话往往需要调用多个工具，例如同时查询天气、股价和新闻。若每次只允许一个工具调用，会导致多轮往返、延迟明显增加。Claude 允许在一个 assistant turn 中返回多个 tool_use 块，让开发者可以并行执行独立操作，从而提升响应速度和任务吞吐。

## 痛点

如果开发者不按格式返回 tool_result，例如把每个结果单独放在不同 user message 中，或在 tool_result 前插入文本，模型会学到避免并行调用，导致后续延迟变高。此外，不了解如何禁用并行会在需要严格顺序执行或限制并发的场景中误触发并行，可能引发副作用、竞态或限流问题。

## 解决办法

Claude 返回 stop_reason 为 tool_use 的 assistant 消息，里面可以包含多个 tool_use 块。API 不规定执行顺序，开发者可根据工具特性选择并发、顺序或混合执行。但所有工具结果必须作为同一个 user message 返回，每个结果用 tool_use_id 与调用一一匹配，并且必须位于任何 text 内容之前。若要禁用并行，可在 tool_choice 对象内设置 disable_parallel_tool_use: true；auto 模式下最多调用一次工具，any 或 tool 模式下强制恰好调用一次。类比：模型一次开出多张任务单，执行顺序由你决定，但所有回单必须一起交回。

## 关键代码示例

```python
import asyncio
from anthropic import Anthropic

client = Anthropic()

async def execute_tool(tool_use):
    # 实际调用你的工具；这里用名字模拟返回
    return {'type': 'tool_result', 'tool_use_id': tool_use.id,
            'content': f'ok:{tool_use.name}'}

async def run_agent(messages, tools):
    resp = client.messages.create(
        model='claude-sonnet-4-5',
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    if resp.stop_reason != 'tool_use':
        return resp

    tool_uses = [b for b in resp.content if b.type == 'tool_use']

    # 并行执行；如果有依赖或副作用，请改成顺序执行
    results = await asyncio.gather(*(execute_tool(tu) for tu in tool_uses))

    # 关键：追加 assistant 原始消息，并把所有 tool_result 放在同一个 user message
    # 且该 message 的 content 中没有任何 text 内容
    messages.append({'role': 'assistant', 'content': resp.content})
    messages.append({'role': 'user', 'content': results})
    return await run_agent(messages, tools)
```

代码发送请求后检查 stop_reason 是否为 tool_use，如果是则提取所有 tool_use 块。用 asyncio.gather 并发执行无依赖的工具调用，体现开发者可以自由决定执行策略。最后将 assistant 原始内容和所有 tool_result 作为连续两条消息追加，其中 user 消息只包含 tool_result，且所有结果合并在一起，严格遵守并行工具调用的消息格式要求。

## 关键流程

1. 发送一个可能需要多个工具调用的请求，检查响应 stop_reason 是否为 tool_use 且 content 中有多个 tool_use 块。
2. 根据工具是否有副作用、共享状态或依赖关系，选择并发执行（如 asyncio.gather）或顺序执行。
3. 为每个 tool_use 生成一个 tool_result，用 tool_use_id 精确匹配；跳过或失败的调用也返回 is_error: true。
4. 把所有 tool_result 放在同一个 user message 中，并确保它们位于任何文本内容之前，然后继续下一轮对话。
5. 如需禁用并行，在 tool_choice 对象内设置 disable_parallel_tool_use: true；auto 时最多一次，any 或 tool 时恰好一次。

## 关键点

- 并行工具调用是默认行为，Claude 可以在一个 assistant turn 中返回多个 tool_use 块；开发者可以自行选择并发、顺序或混合执行，但必须理解工具之间的依赖和副作用。
- 返回工具结果时，所有 tool_result 必须放在同一个 user message 中，并且位于任何 text 内容之前，用 tool_use_id 精确匹配，否则会被视为错误格式，降低模型后续并行调用的意愿。
- 即使某个工具调用没有实际执行，例如因为前序调用失败而跳过，也必须返回一个带 is_error: true 的 tool_result，保持每个 tool_use 都有对应结果。
- 禁用并行不是顶层参数，而是放在 tool_choice 对象内设置 disable_parallel_tool_use: true；当 tool_choice 为 auto 时最多调用一次工具，为 any 或 tool 时强制只调用一次。
- computer use 和 browser use 的批量动作更严格，必须按出现顺序顺序执行，并在首个失败时停止；跳过时需返回工具定义的特定文本。
- 排查并行工具调用问题时，最常见原因是消息历史中 tool_result 格式错误，尤其是把多个 tool_result 拆成多条 user message；应始终合并到一条消息。

## 对比与权衡

- 相比顺序执行所有工具调用，并行执行能显著降低独立只读操作的总延迟，但若工具之间存在副作用或共享状态，并行执行可能引发竞态，因此需要根据工具语义选择策略。
- 相比把多个 tool_result 分散在多个 user message 中，合并到同一个 user message 更符合 Claude 的消息协议，也能保持模型对并行调用的偏好；分散发送会导致格式错误和模型并行意愿下降。
- 相比普通工具调用的自由执行策略，computer use 和 browser use 的批量动作更严格，要求按出现顺序顺序执行并在首个失败时停止，这是为了保证 UI 自动化场景中的状态一致性。

## 自测问题

**问: Claude 返回多个 tool_use 时，我应该并发还是顺序执行？**

API 不规定执行顺序。判断标准是工具之间是否存在依赖、副作用或共享状态。独立只读操作适合并发以降低延迟；有依赖或副作用时需要顺序执行。还可以在 system prompt 中要求模型只批量调用相互独立的工具，减少依赖调用同时出现。

**问: 为什么所有 tool_result 必须放在同一个 user message，而且不能在前面放 text？**

这是 Anthropic 消息格式的硬性要求。每个 tool_use 必须有对应的 tool_result，用 tool_use_id 匹配。所有 tool_result 应在一个 user turn 中一起返回，且不能在其前面插入文本，否则 API 认为格式不合法，模型也会学到避免并行调用。

**问: disable_parallel_tool_use 为什么放在 tool_choice 里而不是顶层参数？**

因为它影响的是工具选择策略，与 tool_choice 的语义紧密耦合。auto 下禁用后表示最多一次工具调用，any 或 tool 下禁用后表示强制恰好一次。如果放在顶层，会破坏不同 tool_choice 模式下的语义解释。另外并非所有模型都支持 any 或 tool。

**问: 如果批量工具调用中某个失败了，还要返回 tool_result 吗？**

要。每个 tool_use 都必须有对应 tool_result。如果因为执行失败或跳过，需要返回 is_error: true 和简短说明，这样模型才知道哪个调用失败并决定下一步。对于 computer use 和 browser use，跳过时还要返回工具规定的特定文本。

**问: 如何测试和验证并行工具调用是否正常工作？**

发送一个需要多个工具的信息，检查响应 stop_reason 是否为 tool_use 且 content 中有多个 tool_use 块。然后按格式把所有结果合并在同一个 user message 返回，观察后续响应是否继续保持并行。如果模型不并行，优先检查历史消息格式是否错误，或加强 system prompt 引导。

## 适用场景

- 需要同时从多个独立数据源获取信息，例如同时查询天气、股价和新闻，以减少用户等待时间。
- 多步骤 Agent 工作流中，对无依赖的检索、计算或外部 API 调用进行并行执行，提高吞吐量和响应速度。
- 需要严格顺序执行或避免并发的场景，如写入数据库、扣减库存等有副作用的操作，可通过 disable_parallel_tool_use 禁用并行。
- 调试工具调用格式或排查模型为什么不并行的问题时，参考本文的故障排查规则和消息历史格式要求。

## 标签

`Claude` `工具调用` `并行执行` `Agent` `Anthropic API`
