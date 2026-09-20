# Claude 工具定义与调用控制：Schema、描述与 tool_choice

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools) · 来源: web · 生成时间: 2026-09-20T03:36:40.345422+00:00*

## 背景

传统 LLM 只能输出自然语言，无法可靠触发外部动作或生成机器可读的调用参数。Claude 工具调用让模型输出结构化 tool_use 块，由客户端执行后把结果回传，实现真实业务操作。工具定义质量直接决定模型是否选对工具、填对参数，因此需要专门的 tools 参数和最佳实践。

## 痛点

描述太简略或 schema 不清晰时，模型容易错误选择工具、遗漏必填参数或生成不合法入参。工具数量多且命名随意会造成选择歧义；响应的冗余字段会消耗大量 context，使模型忽略真正关键的信息。

## 解决办法

在 API 请求顶层 tools 数组传入工具定义，每个工具包含 name、description、input_schema，可选 input_examples。API 会把工具定义和用户 system prompt 编译为专用 system prompt，指导模型生成合法 tool_use 调用。模型不执行工具，而是由开发者在客户端执行并把结果以 tool_result 返回。描述是模型判断“何时用、参数什么含义”的核心依据；input_schema 用 JSON Schema 约束参数结构和类型；input_examples 提供 few-shot 样例；tool_choice 控制是否强制或禁止调用。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=[{
        "name": "get_stock_price",
        "description": "获取指定股票在指定日期的收盘价。当用户询问股价、涨跌或历史价格时使用；不适用于查询公司基本面。",
        "input_schema": {
            "type": "object",
            "properties": {
                "ticker": {
                    "type": "string",
                    "description": "股票代码，如 AAPL、MSFT"
                },
                "date": {
                    "type": "string",
                    "format": "date",
                    "description": "查询日期，YYYY-MM-DD；缺省时返回最近交易日"
                }
            },
            "required": ["ticker"]
        },
        "input_examples": [
            {"ticker": "AAPL", "date": "2024-01-15"}
        ]
    }],
    tool_choice={"type": "auto"},
    messages=[{"role": "user", "content": "苹果昨天收盘价多少？"}]
)

```

代码在 messages.create 请求的 tools 数组中定义一个用户工具。name 是模型用来匹配工具的标识符；description 说明何时用、何时不用；input_schema 用 JSON Schema 约束 ticker 和 date，其中 required 强制 ticker。input_examples 给出一个合法调用示例，tool_choice=auto 表示由模型决定是否调用；真实执行时客户端需处理返回的 tool_use 并把结果作为 tool_result 回传。

## 关键流程

1. 在请求顶层传入 tools 数组，为每个工具填写 name、description、input_schema。
2. 按“做什么、何时用/不用、参数含义、返回值与限制”写至少 3-4 句描述。
3. 用 JSON Schema 定义参数类型、必选字段、枚举和格式。
4. 对复杂入参添加 input_examples，并确保每个示例通过 schema 校验。
5. 按服务或资源加命名空间前缀，合并相关操作为带 action 参数的少量工具。
6. 用 tool_choice 选择 auto、any、tool 或 none 控制调用策略。

## 关键点

- name 必须匹配 ^[a-zA-Z0-9_-]{1,128}$，并建议使用服务前缀命名，如 github_list_prs，以避免工具增多后的选择歧义。
- description 是工具性能最重要的因素，至少写 3-4 句，覆盖做什么、何时用/不用、每个参数的影响、返回内容与限制。
- input_schema 是 JSON Schema 对象，能用 required、type、enum、format 等约束参数，是模型生成合法调用参数的基础。
- input_examples 对复杂嵌套和格式敏感参数很有价值，但必须严格通过 input_schema 校验，否则会返回 400 并增加 token 成本。
- 工具响应应返回高信号、语义稳定的信息，如 slug、UUID 和必要字段，避免上下文膨胀和内部引用泄漏。
- tool_choice 支持 auto、any、tool、none，但 any/tool 在某些模型和手动 extended thinking 下不支持，需用 auto + strict tool use 或 structured outputs 替代。

## 对比与权衡

- 相比在普通 system prompt 里描述工具，tools 参数使用 JSON Schema 和专用 system prompt 约束调用，在参数合法性和调用一致性上更好，但会增加 prompt token 与定义维护成本。
- 相比为每个动作单独建工具，合并为带 action 参数的少量工具在降低选择歧义和维护复杂度上更好，但单个工具的 description 和 input_schema 会更复杂。
- 相比 structured outputs 直接返回固定 JSON，tool use 在需要执行外部副作用和多轮交互上更好，但需要客户端执行函数并正确回传 tool_result，链路更长。
- 相比详细描述，input_examples 在复杂嵌套和格式敏感场景下更直观，但它是辅助手段，不能替代清晰描述，且额外消耗 token。

## 自测问题

**问: 为什么说描述质量是工具性能最重要的因素？**

模型选择工具和填充参数时依赖 description 理解语义，schema 只能限制类型而不能解释业务规则。描述应至少 3-4 句，说明触发条件、非触发条件、参数影响、返回值和限制，这能显著降低误用和漏用。

**问: 如何为复杂工具提供输入示例？**

在 tool 定义中添加 input_examples 数组，每个对象必须严格符合 input_schema，否则 API 返回 400。示例会被注入 prompt，帮助模型学习可选参数和格式；但 server tools 和 computer/browser toolsets 不支持，并需评估 token 成本。

**问: forced tool use 为什么有时会报错？**

手动 extended thinking 以及部分模型不支持 tool_choice 的 any/tool。遇到 400 时改用 auto 配合 strict tool use 保证 schema 合法，或使用 structured outputs 返回固定 JSON；none 仍然可用。

**问: 工具响应应该返回多少数据？**

只返回模型下一步推理所需的高信号字段，使用语义稳定的 slug 或 UUID，避免 opaque 内部引用和整页原始数据。否则会浪费 context、干扰模型，甚至导致幻觉。

**问: 如何设计一组工具以降低选择混淆？**

优先合并相关操作为一个工具，用 action 枚举区分；工具名称加服务前缀；描述中明确边界。工具面越小越清晰，模型越容易导航和选择。

## 适用场景

- 构建需要查询实时数据或调用内部 API 的客服、助手类应用。
- 管理多个业务系统的 Agent，例如同时集成 GitHub、Slack、工单和搜索工具。
- 创建类操作需要严格参数校验，如创建订单、PR、部署任务。
- 合规要求必须强制经过某工具或禁止工具直接回答的流程。

## 标签

`Claude API` `工具调用` `函数调用` `JSON Schema` `提示工程`
