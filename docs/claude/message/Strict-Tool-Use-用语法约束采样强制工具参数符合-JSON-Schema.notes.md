# Strict Tool Use：用语法约束采样强制工具参数符合 JSON Schema

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use) · 来源: web · 生成时间: 2026-09-20T03:44:17.552307+00:00*

## 背景

LLM 本质是概率文本生成器，自由输出无法保证结构化；在工具调用和函数调用场景中，下游代码依赖确定类型和必填字段。传统做法靠提示词约束与事后校验、重试，但模型仍可能返回字符串 `'2'` 而非整数 `2`，或遗漏 required 字段，造成运行时错误。Grammar-constrained sampling 因此被引入，把 JSON Schema 编译成语法，在生成阶段直接限定输出空间。

## 痛点

没有 strict 模式时，Claude 可能把 `passengers` 返回成 `'2'` 或 `'two'`，或者省略必填字段，函数调用会立即抛异常，打断代理工作流。开发者被迫在每次工具调用后编写校验、修复和重试逻辑，增加延迟、成本和维护复杂度。

## 解决办法

通过在工具定义中设置 `strict: true`，服务端会把 `input_schema` 编译成等价语法，通常可理解为有限状态自动机或上下文无关文法；解码时每一步只保留最终能生成合法 JSON 的 token，非法路径被屏蔽。这不是生成后校验，而是在采样阶段直接约束，因此每次工具输入都必然符合 schema，工具名也保证有效。可以类比导航系统：只允许走能到达目的地的道路，而不是走错后再倒车。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.create(
    model='claude-sonnet-4-20250514',
    max_tokens=1024,
    tools=[{
        'name': 'book_trip',
        'description': 'Book a trip with passenger count and destination',
        'strict': True,
        'input_schema': {
            'type': 'object',
            'properties': {
                'passengers': {'type': 'integer', 'minimum': 1},
                'destination': {'type': 'string', 'enum': ['LAX', 'JFK', 'SFO']}
            },
            'required': ['passengers', 'destination'],
            'additionalProperties': False
        }
    }],
    messages=[{'role': 'user', 'content': 'Book a trip for two passengers to LAX'}]
)
print(resp.content[0].input)  # {'passengers': 2, 'destination': 'LAX'}
```

这段代码定义了一个工具 `book_trip`，并把 `strict` 设为 `True`，触发语法约束采样。`input_schema` 限定 `passengers` 必须是 integer、`destination` 必须是枚举值，且两个字段必填。调用后 `resp.content[0].input` 会得到 `{'passengers': 2, 'destination': 'LAX'}`，其中乘客数是整数 2，而不是字符串 `'2'`；这体现了从采样层保证类型安全。

## 关键流程

1. 定义工具的 `input_schema`，使用标准 JSON Schema 描述类型、属性、`required` 和 `additionalProperties` 等约束。
2. 在工具定义顶层添加 `strict: true`，与 `name`、`description`、`input_schema` 并列。
3. 调用 API 后从 `response.content[x].input` 读取参数，这些参数保证符合 schema。
4. 注意 `computer use` 和 `browser use` 工具集不支持 `strict: true`，设置会被拒绝。

## 关键点

- `strict: true` 通过在解码阶段屏蔽非法 token 来保证 schema 合规，而不是依赖生成后的校验和重试，因此可靠性更高。
- 开启后工具 `input` 严格符合 `input_schema`，工具 `name` 也总是有效，这对多步代理工作流至关重要。
- 该机制与 structured outputs 共用 grammar-constrained sampling 管道，编译后的 schema 会被临时缓存最多 24 小时。
- PHI 不能出现在 `input_schema` 的属性名、枚举值、常量或正则中，因为缓存 schema 不具备与消息内容相同的 PHI 保护。
- strict tool use 适用于复杂嵌套对象、枚举、必填字段和类型安全函数调用等场景，可显著减少额外校验代码。

## 对比与权衡

- 相比非严格工具调用（依赖提示词让模型遵守 schema），strict tool use 在类型正确性、必填字段保障上更好，但受 JSON Schema 支持子集限制，某些高级关键字可能不被支持。
- 相比 structured outputs（直接返回 JSON），strict tool use 专注于工具参数校验，并额外保证工具名有效；两者同源但应用场景不同。
- 相比生成后用 JSON Schema 校验并重试的方案，strict tool use 不会浪费无效 token，在延迟和成本上更优，但依赖服务端 grammar-constrained sampling 能力。

## 自测问题

**问: grammar-constrained sampling 具体怎么保证输出一定符合 JSON Schema？**

服务端将 JSON Schema 编译成语法或有限状态自动机，解码时在每个生成步骤屏蔽掉所有无法到达合法结束状态的 token，使整条序列始终在合法空间内。它不是生成完再校验，而是从概率采样上直接限制，因此合规是确定性的。代价是可能牺牲少量生成多样性和增加编译/首包延迟。

**问: strict tool use 与 structured outputs 有什么区别？**

两者共用同一条 grammar-constrained sampling 管道，但 strict tool use 作用于工具调用的 `input` 字段，同时工具名也被验证；structured outputs 则是让模型直接返回一个符合 JSON Schema 的 JSON 响应，适用于非工具场景，如信息抽取、结构化报告。

**问: 为什么文档强调 PHI 不能放在 tool schema 定义中？**

编译后的 schema 会被缓存最多 24 小时，且缓存与 prompts/responses 分离，不享受同等 PHI 保护。如果 PHI 出现在属性名、enum、const 或 pattern 中，可能残留在缓存里造成合规风险。PHI 应只放在 message content 中。

**问: 如果某个工具需要计算机或浏览器操作，能使用 strict:true 吗？**

不能。computer use 和 browser use 的工具集条目会拒绝 strict:true，因为它们的输入结构由系统管理。普通自定义工具可以开启 strict。

**问: 在工程中引入 strict tool use 有哪些潜在成本？**

复杂 schema 编译成语法可能增加首包延迟，且受支持 JSON Schema 子集限制；某些高度开放或需要模型自由发挥的字段可能因约束过严而降低生成质量。需要评估延迟、schema 复杂度和模型输出质量之间的权衡，必要时拆分成多个工具。

## 适用场景

- 构建需要多步工具调用的代理系统，如旅行预订、订单支付、库存查询，确保每次函数调用参数类型正确。
- 处理复杂工具参数，例如嵌套对象、枚举、必填字段和 `additionalProperties: false` 的严格契约。
- 需要减少重试和运行时错误、提高生产一致性的 API 集成和后台任务。
- 合规场景下使用 strict tool use 提升可靠性，但要遵守 PHI 不进入 schema 的规则。

## 标签

`Strict Tool Use` `grammar-constrained sampling` `JSON Schema` `Claude Tools` `agent reliability`
