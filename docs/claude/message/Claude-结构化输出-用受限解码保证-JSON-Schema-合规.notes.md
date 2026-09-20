# Claude 结构化输出：用受限解码保证 JSON Schema 合规

*原文: [https://platform.claude.com/docs/en/build-with-claude/structured-outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) · 来源: web · 生成时间: 2026-09-20T02:30:05.649421+00:00*

## 背景

大语言模型即使经过 prompt 强调，生成 JSON 时仍可能出现非法语法、缺失字段或类型错误。随着 agent 工作流和函数调用普及，下游系统越来越依赖机器可解析的结构化数据，只能靠重试解析已经不够。行业因此引入了 constrained decoding 技术，把 JSON Schema 编译成语法约束，从采样阶段就限制模型只能生成合规 token。

## 痛点

没有结构化输出时，开发者常遇到 JSON.parse 失败、字段缺失、类型不一致等问题，需要额外编写错误处理和重试逻辑。这些重试不仅增加延迟和 token 成本，还可能把坏数据传给下游数据库或 API，导致更严重的故障。

## 解决办法

核心做法是把 JSON Schema 编译为语法约束或有限状态机，在每一步解码时屏蔽不符合 schema 的 token，从而硬性保证生成结果符合结构。SDK 将 Pydantic、Zod 等原生模型自动转换为 JSON Schema，并移除底层暂不支持的约束（如 minimum、maximum），把这些约束写入字段描述，返回后再按原始 schema 做本地验证。JSON outputs 控制模型最终回复的格式，strict tool use 控制工具调用参数的合法性，两者解决不同问题，可以在同一个请求中组合使用。

## 关键代码示例

```python
from anthropic import Anthropic
from pydantic import BaseModel, Field

class UserInfo(BaseModel):
    name: str
    email: str
    age: int = Field(ge=0, le=120, description='Age in years')

client = Anthropic()
response = client.messages.parse(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    messages=[
        {'role': 'user', 'content': 'Extract: Alice, alice@example.com, 30'}
    ],
    output_format=UserInfo,
)
user = response.parsed
print(user.name, user.email, user.age)
```

这里用 Pydantic 模型定义期望结构，Field 约束会被 SDK 自动转换为 JSON Schema。client.messages.parse 负责把模型类传给 output_format，底层通过 constrained decoding 保证返回合法 JSON；response.parsed 则是 SDK 在本地按原始约束验证后的对象。即使平台移除了 age 的 min/max 约束，SDK 也会在返回后重新校验，确保最终拿到的是类型安全且满足业务规则的实例。

## 关键流程

1. 定义目标 JSON Schema，或使用 Pydantic/Zod/Java 类等原生模型描述期望结构。
2. 在 API 请求中设置 output_config.format 为 json_schema，并传入 schema 定义；SDK 中通常直接传模型类。
3. 调用模型，从响应的 text content block 或 SDK 的 parsed 属性中获取结构化结果。
4. 检查 stop_reason：如果是 refusal 或 max_tokens，输出可能不符合 schema，需要做相应降级或重试处理。

## 关键点

- Structured outputs 不是靠 prompt 请求，而是通过 grammar-constrained sampling 在解码阶段硬性限制 token，因此能保证 JSON 语法和 schema 合规。
- JSON outputs 与 strict tool use 必须区分：前者约束模型说什么，后者约束工具调用的参数，两者可独立使用，也可组合用于 agentic 工作流。
- SDK 的 schema 转换非常重要：移除底层不支持的约束（如 minimum、maximum），将约束写入字段描述，并在返回后按原始 schema 验证，从而保留强类型约束。
- 首次使用某个 schema 会有语法编译延迟，编译结果缓存 24 小时；改变 schema 结构或工具集会失效，但只改 name 或 description 不会失效。
- 属性顺序有一个非直观行为：required 字段会排到 optional 字段之前，因此解析时不要依赖顺序，如果顺序敏感则应把所有字段标为 required。
- refusal 和 max_tokens 是结构化输出仍可能不匹配 schema 的两种情况，开发者必须根据 stop_reason 做防御性处理。

## 对比与权衡

- 相比单纯在 prompt 中写“返回 JSON”，结构化输出在合规率与可靠性上更好，但会引入首次编译延迟和额外的系统提示 token 成本。
- 相比 JSON outputs，strict tool use 更关注工具调用参数的 schema 校验，而不是最终回复内容；两者需要组合使用才能同时保证可靠工具调用和结构化回复。
- 相比手动解析并重试的流程，SDK 的 Pydantic/Zod 集成在开发效率和本地验证上更好，但灵活性上仍受限于平台支持的 JSON Schema 子集。

## 自测问题

**问: 结构化输出的底层实现是什么？**

使用 constrained decoding 或 grammar-constrained sampling，先把 schema 编译成语法约束，在解码每一步屏蔽不符合 schema 的 token，从生成机制上保证合法性，而不是靠后处理校验。

**问: 为什么 SDK 还要对 schema 做转换？**

底层对 JSON Schema 支持有限，比如不支持 minimum、maxLength 等；SDK 会移除这些约束，把信息写入字段描述，并在返回后按原始 schema 验证，从而在平台限制下保留严格约束。

**问: 如果返回 JSON 的字段顺序很重要，应该怎么处理？**

文档明确 required 字段会先出现、optional 字段后出现；要么把顺序敏感字段都标为 required，要么解析时按字段名而不是数组位置取值，不要依赖固定顺序。

**问: 结构化输出能 100% 保证返回合法 schema 吗？**

不能。Claude 的安全拒绝和 max_tokens 截断可能返回不符合 schema 的内容；需要检查 stop_reason，分别处理 refusal 和 truncation 场景。

**问: schema 缓存失效的条件有哪些？**

改变 JSON schema 结构或请求中的工具集会失效；只改 name 或 description 不会失效。此外改变 output_config.format 会失效 prompt cache，所以不要把动态字段放进 schema 以免频繁失效。

## 适用场景

- 从图片或非结构化文本中提取实体信息，如发票、简历、用户描述，并直接写入数据库。
- 生成结构化报告或 API 响应，供前端渲染或下游服务消费，避免二次解析错误。
- Agent 工作流中同时需要可靠工具调用和结构化最终输出，例如智能助手既调函数查询又返回格式化结果。
- 批处理或数据管道中需要强类型模型输出，保证每个样本都符合固定结构，便于后续分析。

## 标签

`结构化输出` `JSON Schema` `constrained decoding` `Claude API` `工具调用`
