# 程序化工具调用：让 Claude 在代码执行沙箱中直接驱动你的工具

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) · 来源: web · 生成时间: 2026-09-20T04:14:41.780324+00:00*

## 背景

在传统 tool calling 中，Claude 每调用一次工具都需要一次模型采样，开发者要把工具结果送回上下文，模型再决定下一步。这在多步检索、批量查询、数据分析等场景中会带来大量往返和 token 消耗。程序化工具调用把“调用工具”的能力下沉到代码执行沙箱中，让 Claude 写一段代码来编排多个工具调用，中间结果只在沙箱内处理，只有最终结果进入模型上下文。它解决的是 agentic workflow 中“多步工具调用又慢又贵、中间数据淹没上下文”的问题。

## 痛点

没有这个能力时，检查 20 个员工的预算合规可能需要 20 次模型往返，并把几千行费用明细拖进上下文。多步搜索、批量查询或需要条件判断的工具编排也容易因为反复采样而变慢、变贵，甚至超过上下文窗口。此外，大量无关工具结果进入模型后，还会增加推理噪声，降低最终答案质量。

## 解决办法

核心机制是在工具定义中增加 `allowed_callers` 字段，允许工具被 `code_execution_20260120` 这个代码执行环境调用。Claude 判断需要这些工具时，不再逐个发起传统 tool call，而是在沙箱里写 Python 代码，把工具当作函数来调用，可以包含循环、条件、过滤和聚合。代码执行到工具调用处会暂停，API 返回带 `caller` 字段的 `tool_use` 块；开发者完成真实工具调用后，把结果作为 `tool_result` 续传，代码会在原容器中继续运行。中间结果不会进入 Claude 的上下文窗口，只有代码执行的最终输出会返回给模型。可以把它类比为：以前是“你给助手 20 个电话，一个一个打，每个结果都回来汇报”；现在是“你给助手一段脚本，让它在小房间里自己跑完，最后只把不符合预算的名单交给你”。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
tools = [{
    "name": "query_database",
    "description": "Run SQL against analytics DB",
    "input_schema": {
        "type": "object",
        "properties": {"sql": {"type": "string"}},
        "required": ["sql"],
    },
    # 关键：允许该工具从代码执行容器内被调用
    "allowed_callers": ["code_execution_20260120"],
}]

resp = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=[{"type": "code_execution_20260120", "name": "code_execution"}, *tools],
    messages=[{"role": "user", "content": "Query last quarter purchase history and identify top 5 customers by revenue."}],
)
# 若 resp.stop_reason == "tool_use" 且 tool_use.block.caller == "code_execution_20260120"
# 则把该 tool_use 对应的真实工具结果作为 tool_result 发回，并携带 resp.container.id
```

示例先定义了一个数据库查询工具，并通过 `allowed_callers` 将其标记为只能从代码执行容器中调用，这是启用程序化调用的关键开关。请求中同时启用了代码执行工具并传入自定义工具，Claude 可以在沙箱里写代码调用 `query_database`。响应阶段通过 `stop_reason` 和 `caller` 判断是否为沙箱发起的工具调用，开发者只需返回真实工具结果并携带容器 ID 即可让代码继续执行。

## 关键流程

1. 在工具定义中添加 `allowed_callers: ["code_execution_20260120"]`，并在请求工具列表中启用代码执行工具。
2. Claude 判断需要该工具时，会在沙箱中编写 Python 代码调用它，API 暂停并返回带 `container` ID 和 `caller` 字段的 `tool_use` 块。
3. 开发者在真实环境中执行对应工具，构造只包含 `tool_result` 的用户消息，并携带上一步返回的 `container` ID 和原始 `tools` 数组。
4. 代码执行容器恢复运行，可能继续产生更多程序化 `tool_use`，也可能完成代码执行。若有更多调用，重复上一步。
5. 代码执行完成后，Claude 收到代码最终输出，继续生成最终回答。

## 关键点

- `allowed_callers` 是启用程序化调用的开关：默认值是 `["direct"]`，只有显式加入 `code_execution_20260120` 才能让沙箱代码调用该工具。它同时承担了能力开启和调用路径控制两个职责。
- 中间结果不会进入模型上下文，只有代码执行完成后的最终输出会返回给 Claude。这是 token 消耗大幅下降的根本原因。
- 响应中的 `caller` 字段用于区分调用来源：传统直接调用和代码执行沙箱调用。`tool_id` 还能关联到具体是哪个代码执行块发起的调用。
- 在代码暂停等待 `tool_result` 时，续请求必须携带 `container` ID，否则 API 会拒绝。容器有约 5 分钟空闲回收时间和 30 天最大复用期限。
- 该能力在 BrowseComp 和 DeepSearchQA 等多步检索基准上平均提升 11%，同时减少 24% 输入 token，说明它特别适合需要大量工具调用但中间结果不需要逐次推理的场景。
- 程序化工具调用不是万能方案：单步简单工具调用、需要模型对每个中间结果做复杂语义判断、或强副作用工具需要逐步确认的场景，传统 direct 调用反而更简单可靠。

## 对比与权衡

- 相比传统 direct tool calling，程序化工具调用在减少模型往返、降低 token 消耗和过滤无关数据上更好，但它在模型对中间结果的实时推理控制上更弱，且需要代码执行容器，增加系统复杂度。
- 相比在客户端自己实现 agent loop 来编排工具，程序化调用把循环、条件等编排逻辑放进沙箱代码中，减少了客户端状态机和采样次数；但客户端仍要处理容器生命周期和暂停/恢复机制，调试链路也更长。
- 相比纯代码执行而不接入外部工具，程序化工具调用让沙箱代码能访问真实业务工具、数据库或 API，能力更强；但这也要求更严格的工具权限控制，避免代码执行环境滥用敏感工具。

## 自测问题

**问: 程序化工具调用和传统 tool calling 的核心区别是什么？**

传统方式是模型每次决定调用一个工具，API 返回 tool_use 后必须把结果送回模型再推理，多步工作流会反复采样。程序化方式是模型写一段代码，在代码执行容器内把工具当函数调用，中间结果由代码处理，只有最终输出进入上下文。API 的暂停/恢复由 stop_reason、container ID 和 caller 字段配合完成。

**问: allowed_callers 字段有什么作用？取值有哪些？**

它控制工具可以被哪些上下文调用。默认是 ["direct"]，表示由模型直接调用；["code_execution_20260120"] 表示只允许从代码执行容器调用；也可以两者都允许。两个代码执行版本字符串可以互换。它本质上是在工具定义层做调用路径控制和权限边界。

**问: 为什么在暂停等待 tool_result 时，续请求必须携带 container ID？**

因为代码执行容器是有状态的沙箱，运行中的代码暂停在工具调用处，继续执行必须找到原来的容器环境。如果不回传 container ID，API 无法恢复对应运行状态，会直接拒绝请求。类比进程暂停后需要句柄才能恢复。

**问: 中间结果不进入模型上下文是如何做到的？对 token 有什么影响？**

工具返回值由代码执行容器拦截，继续在容器内计算、过滤、聚合。模型只在代码执行完全结束后获得 print 或返回的最终结果。比如预算检查中，20 个员工的几千行明细被代码过滤成几个超标员工，模型上下文只需处理几行，因此输入 token 大幅下降。

**问: 什么场景不适合使用程序化工具调用？**

单步简单工具调用会增加容器开销；需要模型对每个中间结果做复杂语义判断或逐步确认的流程不适合；强副作用工具需要人在环或严格审计时，传统 direct 调用更直观可控。另外，如果团队没有代码执行容器管理经验，引入它会增加运维复杂度。

## 适用场景

- 批量查询与合规检查：例如一次性检查 20 个员工的预算使用情况，只返回超标人员。
- 多步检索与深度研究：例如 BrowseComp、DeepSearchQA 中的网页搜索、过滤、排序和聚合。
- 数据库分析：多次 SQL 查询后在沙箱内完成 join、聚合、Top N 计算。
- 需要循环或条件逻辑的工具编排：根据上一步工具结果决定下一步调用，但中间结果不需要模型逐次推理。

## 标签

`tool-calling` `code-execution` `agent` `Claude API` `token-optimization`
