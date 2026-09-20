# 停止原因与回退处理（Stop reasons and fallback）

*原文: [https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) · 来源: web · 生成时间: 2026-09-20T02:19:32.464079+00:00*

## 背景

LLM API 在生成时可能因自然结束、达到 token 上限、触发工具调用、遇到停止序列或安全拒绝而停止。Claude 在 Messages API 响应中通过 stop_reason 字段显式返回停止原因，并引入 fallback 机制帮助开发者在主模型不可用或输出不完整时降级处理。这个设计尤其适合 Agent 和长文本生成等需要多轮控制的应用。

## 痛点

如果不检查 stop_reason，容易把 max_tokens 截断当作完整回答展示给用户；忽略 tool_use 会导致 Agent 不执行工具就结束；模型不可用或 refusal 时没有回退路径，会造成生成中断、体验差甚至业务失败。

## 解决办法

把 stop_reason 当作生成状态机的控制信号，而不是只看响应文本。收到响应后先判断 stop_reason：end_turn 表示正常结束；tool_use 表示模型请求调用工具，需要执行工具并把 tool_result 追加到消息后继续对话；max_tokens 表示输出被截断，可提高 max_tokens 或追加“Continue”让模型续写；stop_sequence 和 refusal 则需要按业务或安全策略处理。对于主模型异常或结果不可接受的情况，可切换到备用模型作为 fallback，类似微服务中的降级与重试。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

def run_agent(messages, model='claude-sonnet-4-5', fallback='claude-haiku-4-5', max_loop=5):
    for _ in range(max_loop):
        try:
            r = client.messages.create(model=model, max_tokens=1024, messages=messages)
        except anthropic.APIError:
            r = client.messages.create(model=fallback, max_tokens=1024, messages=messages)

        messages.append({'role': 'assistant', 'content': r.content})

        if r.stop_reason == 'end_turn':
            return r
        if r.stop_reason == 'tool_use':
            for b in r.content:
                if b.type == 'tool_use':
                    messages.append({'role': 'user', 'content': [
                        {'type': 'tool_result', 'tool_use_id': b.id, 'content': execute_tool(b.name, b.input)}
                    ]})
            continue
        if r.stop_reason == 'max_tokens':
            messages.append({'role': 'user', 'content': 'Continue exactly where you left off.'})
            continue
        return r
    raise RuntimeError('agent loop limit reached')
```

这段代码先尝试主模型，遇到 API 错误时切到 fallback 模型，体现可用性兜底；随后把 assistant 响应写入 messages。核心是 stop_reason 分支：end_turn 正常返回；tool_use 执行工具并回填 tool_result 后继续循环；max_tokens 追加“Continue”续写，避免截断被当作完整结果。

## 关键流程

1. 检查响应中的 stop_reason 字段，而不是仅判断返回文本是否为空。
2. end_turn：作为正常结束返回给用户。
3. tool_use：执行对应工具，生成 tool_result 并追加到 messages 后继续调用。
4. max_tokens：提高 max_tokens 或追加续写提示，必要时拆分长任务。
5. refusal 或 stop_sequence：按安全或业务规则处理，例如转人工或停止生成。
6. 为整个流程设置最大循环次数，并在主模型异常或不可用时切换备用模型作为 fallback。

## 关键点

- stop_reason 是生成状态的一等公民字段：同一 HTTP 200 响应可能代表完整回答、截断、工具请求或拒绝，业务必须根据它分流。
- tool_use 是 Agent 的核心信号：模型停下来是为了把控制权交还给开发者执行工具，必须回填 tool_result 后继续，否则任务会中断。
- max_tokens 不代表模型已经说完：它是资源上限导致的截断，应设置合理的 max_tokens 预算，并通过续写或拆分控制成本。
- fallback 是生产系统的兜底策略：当主模型不可用、返回 refusal 或结果不满足要求时切换备用模型，可以提高可用性，但要注意能力与成本差异。
- 多轮 Agent 或续写流程必须设置循环上限、幂等和可观测性，否则容易陷入重复调用工具或无限续写的死循环。
- 不同模型或厂商的 finish reason 命名不同，迁移或对接时要做映射，不能假定 stop_reason 语义完全一致。

## 对比与权衡

- 相比只判断 HTTP 200 或文本非空的简单逻辑，基于 stop_reason 的分支处理更可靠，能正确区分正常结束与截断或工具调用，但也要求开发者维护状态机。
- 相比仅通过调大 max_tokens 解决截断，续写加 fallback 方案更灵活，能应对上下文或成本限制，但会引入额外推理成本和循环控制复杂度。
- 相比平台自动 fallback，客户端自建回退逻辑可控性和可观测性更强，但需要自己处理模型能力差异和工具 schema 兼容问题。

## 自测问题

**问: Claude Messages API 中 stop_reason 常见取值有哪些？分别代表什么？**

常见有 end_turn（自然结束）、max_tokens（输出达到 token 上限被截断）、stop_sequence（命中自定义停止序列）、tool_use（模型请求工具调用），新版本还可能看到 refusal、pause_turn。回答时不要只报名字，要说明每种停止后开发者应做什么。

**问: 如果应用收到 stop_reason=max_tokens，但回答不完整，你会怎么处理？**

先确认是输出预算不足还是任务本身过长。可以调大 max_tokens；如果上下文有限，采用“Continue exactly where you left off”续写；更优方案是问题拆分、分段生成或摘要压缩前文。避免无限续写，设置循环上限，记录续写次数。

**问: 如何实现一个基本的工具调用 Agent 循环？**

把 assistant 消息和 tool_use block 保留在 messages 中，执行工具后追加 user 角色的 tool_result，再继续调用模型；直到 end_turn 或达到循环上限。注意工具失败也要回传错误信息，方便模型调整。

**问: fallback 模型切换时要注意哪些问题？**

能力差异可能导致输出质量和格式变化，工具定义和指令需兼容；要控制成本与延迟，避免所有请求都降级；切换原因要可观测；最好可配置回退链，并在主模型恢复后切回。

**问: 为什么不能只靠 HTTP 状态码判断一次生成是否成功？**

API 可能返回 200 但业务未完成，例如 stop_reason=max_tokens 或 refusal。需要把 stop_reason 纳入判定；HTTP 错误通常重试，而业务未完成需要分支处理或降级。

## 适用场景

- 客服或问答应用中需要保证答案完整，不能把截断当结束。
- 工具调用型 Agent 需要根据 tool_use 执行函数并继续多轮推理。
- 长文生成或代码补全中出现 max_tokens 后需要续写。
- 生产环境高可用架构中主模型不可用或拒绝时降级到备用模型。

## 标签

`Claude API` `stop_reason` `fallback` `LLM工程` `Agent`
