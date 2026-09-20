# Claude Thinking 机制：配置、摘要读取与多轮工具调用

*原文: [https://platform.claude.com/docs/en/build-with-claude/thinking](https://platform.claude.com/docs/en/build-with-claude/thinking) · 来源: web · 生成时间: 2026-09-20T02:54:27.696725+00:00*

## 背景

传统大模型如果只允许单次直接输出答案，等于没有草稿纸：遇到证明、复杂 bug、长链路 agent 任务时，第一次想到的路径往往不是最优路径。Claude Thinking 让模型在正式回答前产生独立推理块，先提出假设、尝试、校验并放弃无效路线，从而提升复杂任务的可靠性。

## 痛点

没有 thinking 时，模型被迫一次性写对答案，中间校验和试错无处发生，复杂计算、代码调试和长任务容易中途跑偏或输出伪推理。如果不理解 thinking 的 signature 和 token 计费机制，则会在多轮对话、工具调用中丢失上下文，或错误设置 max_tokens 导致响应被截断。

## 解决办法

通过 thinking 配置项控制是否思考以及返回多少思考内容：type 可设为 adaptive、enabled 或 disabled，display 可设为 summarized、omitted 或 updates。服务端会在 text 块之前生成 thinking content blocks；其中 thinking 字段是可读推理摘要，不是原始思维链，完整推理被加密在 signature 字段中。模型可以结合 effort 级别或 budget_tokens 控制思考深度；thinking token 计入输出 token 和 max_tokens，因此需要预留预算。多轮对话和工具调用时，必须把上一轮响应中的 thinking 块及 signature 原样回传，不能删除或修改。

## 关键代码示例

```python
from anthropic import Anthropic

client = Anthropic()

def ask_with_thinking(prompt: str):
    response = client.messages.create(
        model='claude-sonnet-5',          # 替换为目标模型
        max_tokens=8000,                  # 要同时容纳 thinking + 回答
        thinking={'type': 'adaptive', 'display': 'summarized'},
        messages=[{'role': 'user', 'content': prompt}],
    )

    reasoning, answer = [], []
    for block in response.content:
        if block.type == 'thinking':
            reasoning.append(block.thinking or '(omitted)')
        elif block.type == 'text':
            answer.append(block.text)

    # 多轮/工具调用：必须把 response.content 原样作为 assistant content 带回，
    # 不能删除 thinking 块，也不要修改 signature。
    return reasoning, ''.join(answer), response.content

```

代码设置 type='adaptive'，让模型根据请求复杂度自行决定是否思考及思考深度；display='summarized' 返回可读推理摘要。max_tokens 必须同时覆盖 thinking token 和最终回答 token，否则容易截断。遍历响应时先处理 thinking block 再处理 text block；多轮或工具调用时必须原样回传 response.content，尤其不能修改 signature 字段。

## 关键流程

1. 确认目标模型支持的 thinking 类型和默认值，选择 adaptive、enabled 或 disabled。
2. 按产品需要设置 display：用户可见或调试用 summarized，隐藏思考且降低传输量用 omitted，工具过程可见性可用 updates(beta)。
3. 增大 max_tokens 以覆盖 thinking token 和最终回答；必要时用 effort 或 budget_tokens 控制推理深度与成本。
4. 在多轮对话和工具调用中，将前次响应中的 thinking 块、tool_use 块及 signature 原样回传，不要篡改或删除。

## 关键点

- thinking blocks 是独立于正式回答的生成内容，出现在 text 块之前；模型会在其中试错、检查和放弃错误路线，因此复杂任务性能更好。
- API 返回的 thinking 字段是可读摘要，不是原始思维链；完整推理被加密保存在 signature 字段中，这个设计兼顾了可解释性、安全性和多轮连续性。
- display='omitted' 时 thinking 字段为空，但 token 照常计费，signature 也仍然存在；不能因为看不到思考文本就认为模型没有思考或没有产生成本。
- thinking token 计入输出 token 和 max_tokens，预算太小会截断最终回答；关闭展示省的是传输和渲染成本，不是模型推理成本。
- 多轮对话和工具调用必须原样回传 thinking content blocks，尤其是 signature；否则会破坏模型的前序推理链，通常触发 400 错误。
- 不同模型默认值不同：部分新模型默认开启 thinking 但 display 为 omitted，部分旧模型需要显式开启 adaptive，个别模型不允许关闭 thinking。

## 对比与权衡

- 相比普通单次直出模式，Thinking 在复杂推理、调试和规划上更可靠，但 token 消耗更高、首字输出更慢，简单任务可能不需要开启。
- 相比 extended thinking 的 enabled + budget_tokens 显式预算模式，adaptive 模式接入更省心，由模型自行决定何时思考和思考多深，但硬成本上限控制不如 budget_tokens 直接。
- 相比 display='summarized'，display='omitted' 在流式场景下能更快吐出文本 token，并避免暴露中间推理细节，但牺牲了可读的调试信息；两者计费相同。

## 自测问题

**问: thinking block 和普通 text block 有什么本质区别？**

两者都是模型生成内容，但 thinking 是独立于正式答案的中间推理块，不直接面向用户；其 thinking 字段只是摘要，完整推理通过 signature 加密保存。多轮场景中 thinking 块必须原样回传。

**问: 为什么多轮对话或工具调用必须原样回传 thinking 块和 signature？**

后续请求依赖前序完整推理上下文，signature 是该上下文的加密表达。删除或改动会导致模型无法恢复之前的推理链路，通常返回 400 错误。即使 display='omitted'，也要把空 thinking 字段和 signature 原样带回。

**问: display='omitted' 能否省钱或减少 token 消耗？**

不能。thinking token 仍按输出 token 计费，并计入 max_tokens。omitted 只是不在响应中展示推理摘要，可降低传输量、减少隐私暴露，并可能改善流式首字延迟。成本控制应通过 effort、budget_tokens、提示词简化或直接关闭 thinking 来实现。

**问: 工具调用中 thinking 通常出现在哪里？开发时要注意什么？**

thinking 可能出现在首个工具调用之前，也可能出现在多个 tool_use 之间。典型循环是：模型先 thinking，再发起 tool_use；服务端执行后用 tool_result 继续，下一轮请求必须包含之前的 thinking/tool_use 块并保留 signature。

**问: 如何选择 display 的 summarized、omitted 和 updates？**

需要给用户展示推理摘要或做提示词调试时用 summarized；生产环境不需要展示、希望减少首字延迟或避免泄露中间逻辑时用 omitted；长 agent 任务中希望展示简短进度信息可用 updates(beta)。

## 适用场景

- 复杂代码调试、算法推导、数学证明等需要逐步推理并验证中间结果的场景。
- 长链路 agent 或工具调用任务，模型需要在多次工具执行之间规划、检查和调整方向。
- 需要审计或调试模型推理路径的产品，可以通过 display='summarized' 查看推理摘要。
- 流式输出、缓存和长时间会话中需要综合管理 token 预算、上下文窗口和首字延迟的生产系统。

## 标签

`Claude Thinking` `Anthropic API` `链式推理` `Token 成本` `工具调用`
