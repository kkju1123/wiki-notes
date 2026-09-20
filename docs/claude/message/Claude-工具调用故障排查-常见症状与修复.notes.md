# Claude 工具调用故障排查：常见症状与修复

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use) · 来源: web · 生成时间: 2026-09-20T04:06:04.037383+00:00*

## 背景

工具调用让 LLM 能执行外部操作，是构建 Agent 和业务集成的核心机制。但模型不是确定性编译器，工具描述、schema、消息历史和缓存配置稍有偏差，就会导致选错工具、参数越界、请求 400 或缓存失效。原文把这些高频问题整理成症状到修复表，适合在排错时快速定位根因。

## 痛点

如果不理解这些错误背后的约束，开发者容易盲目重试、乱调 prompt 或反复重启会话，浪费大量时间。消息格式不对会直接导致 API 400；缓存频繁失效会推高成本；工具结果中混入指令还可能触发安全误判或注入风险。

## 解决办法

核心思路是把工具调用当作一条受约束的闭环：模型依据描述和示例选择工具，依据 schema/strict 生成参数，客户端必须用 tool_result 精确闭合每个 tool_use。描述要写清楚“什么时候用”而不只是“做什么”；strict:true 可以在受支持 schema 内硬性阻止幻觉参数；多个 tool_result 必须放在同一个 user 消息中且位于文本之前；缓存失效通常源于提示前缀变化，需要稳定工具数组、thinking 和 effort，或用 defer_loading 内联追加工具；工具结果只放数据，指令通过 user 或 system 消息下发。类比 HTTP 请求响应必须配对，工具调用也是调用与结果配对，配错就会报错。

## 关键代码示例

```python
def build_user_message_after_tools(assistant_msg, results):
    content = []
    for block in assistant_msg.get('content', []):
        if block.get('type') == 'tool_use':
            content.append({
                'type': 'tool_result',
                'tool_use_id': block['id'],
                'content': results[block['id']],  # 只放数据，不要夹带指令
            })
    return {'role': 'user', 'content': content}

```

这段代码按 assistant 消息中的 tool_use 块逐个生成 tool_result，并全部放进同一条 user 消息。它对应两个排错原则：每个 tool_use id 都必须闭合；多个并行 tool_result 不能拆成多轮，否则会被当成串行或触发 400。注释强调 content 只放数据，避免把后续指令混入工具结果被误判为 prompt injection。

## 关键点

- 工具描述必须说明“什么时候用”而不是只描述功能；否则模型在相似工具之间容易误判，这是选错工具最常见根因。
- strict:true 能在受支持 schema 内强制参数类型和枚举值，直接解决“模型猜参数”的幻觉；但复杂 pattern 会编译失败，需要简化或改用 input_examples。
- 工具调用循环要求每个 tool_use 都有且仅有一个 tool_result，并且 tool_result 要放在 user 消息文本之前；这是避免 400 的核心消息格式约束。
- 并行工具调用需要把多个 tool_result 放进同一个 user 消息，disable_parallel_tool_use 必须在返回 tool_use 的那次请求上设置，后设无效。
- 缓存失效通常不是缓存坏了，而是 tool_choice、thinking 配置、effort 或工具数组头部变化改变了提示前缀；保持这些稳定或用 defer_loading 追加工具。
- 工具结果中的指令会被模型视为不可信第三方内容；指令应放在 user 或 mid-conversation system 消息中，工具结果只保留数据。

## 对比与权衡

- 相比只用 input_examples 暗示参数约束，strict:true 在受支持 schema 上能硬性拒绝越界参数，可靠性更好；但 strict 模式对 regex/enum 大小等限制更多，复杂场景下不如 input_examples 灵活。
- 相比把多个 tool_result 拆成多个 user 轮次，合并到同一个 user 消息能保持并行调用语义并减少 400 错误；但当后续工具依赖先一个工具结果时，这种合并会不适用，需要串行等待。
- 相比直接在 tools 数组头部插入工具，defer_loading 配合工具搜索以内联方式追加工具，对缓存前缀影响更小、缓存命中率更好；但会增加一次工具搜索调用和实现复杂度。
- 相比对序列化后的工具输入做原始字符串匹配，统一用 json.loads 或 JSON.parse 能兼容不同模型版本的 Unicode/斜杠转义差异；但前提是输入必须是合法 JSON，异常路径需要处理。

## 自测问题

**问: Claude 总是调用工具 A 而不是我期望的工具 B，应该先检查什么？**

先看工具描述是否只写了功能，没有写清楚各自的使用时机；应改写描述说明什么时候用哪个工具，并补充 input_examples。还要检查工具名是否重复或过于相似，必要时合并或重命名工具，避免模型在相似工具间摇摆。

**问: strict tool use 一定能解决模型虚构参数吗？会不会引入新问题？**

strict:true 能让模型只按 input_schema 生成参数，但 schema 必须在受支持子集内；例如正则的 backreference、lookaround、word boundary 或过大 {n,m} 区间会编译失败。遇到这些情况需要简化 pattern，或退回到 input_examples 约束模型。

**问: 多个 tool_use 并行调用时，我逐个返回 tool_result 为什么被 400 拒绝？**

工具调用循环要求 assistant 消息中的所有 tool_use 都必须在紧接着的 user 消息中用 tool_result 闭合；逐个返回会打破这个配对关系。正确做法是在同一个 user 消息中，为每个 tool_use id 放一个 tool_result，且放在任何文本之前。

**问: 为什么我只是在会话中途加了一个工具，prompt cache 就全 miss？**

缓存基于提示前缀，工具的增删如果改变了 tools 数组的头部或位置，后续 token 都会变化。可以保持已缓存前缀稳定，新增工具用 defer_loading 在内联位置挂载；同时工具选择、thinking 配置和 effort 在缓存生命周期内保持恒定。

**问: 工具返回了一段用户上传的文本，里面写着“忽略之前的指令”，Claude 拒绝执行，是为什么？**

Claude 将工具结果视为潜在的不可信第三方内容，其中嵌入的指令可能被当作间接注入。正确做法是让工具结果只保留数据，任何后续指令通过 user 消息或 mid-conversation system 消息下发。

## 适用场景

- 构建需要工具调用的 Claude Agent 时，遇到模型选错工具、不调用工具或参数类型错误。
- 多个工具可以并行执行的场景，例如同时查询 CRM、订单系统和日志系统，需要正确组装并行 tool_result。
- 长会话中启用 prompt caching 后，排查缓存命中率下降或每次请求都 miss。
- 工具会返回外部网页、工单、用户输入等不可信内容时，设计防注入边界。

## 标签

`Claude` `工具调用` `故障排查` `Prompt Caching` `Strict Mode`
