# Define your Agent：可复用、版本化的 Claude 智能体配置

*原文: [https://platform.claude.com/docs/en/managed-agents/agent-setup](https://platform.claude.com/docs/en/managed-agents/agent-setup) · 来源: web · 生成时间: 2026-09-20T07:08:18.226459+00:00*

## 背景

Claude 的 Managed Agents 要解决大型语言模型应用里“配置散落、行为难对齐”的问题。传统调用大模型时，system prompt、工具、MCP 连接和技能往往分散在每次请求或业务代码里，难以复用和审计。把 agent 抽象成可版本化资源配置后，团队可以像管理基础设施一样管理智能体，保证多会话一致并支持回滚。

## 痛点

如果每次会话都手动传 system、工具和 MCP 配置，容易出现配置漂移、复制粘贴错误，也无法做版本审计。更新配置时若缺少并发控制，还可能出现旧版本覆盖新版本的竞态，导致线上 agent 行为不符合预期。

## 解决办法

核心是把 agent 作为一等资源：创建时一次性声明 name、model、system、tools、mcp_servers、skills 等，平台返回 id 和 version。会话只引用 agent id，需要临时调整时使用 session override。更新时通过 version 做乐观并发，带 version 版本不匹配会返回 409，省略则 last-write-wins；字段更新遵循省略保留、scalar 替换等语义。类比 Docker image 或 Kubernetes Deployment：配置声明一次，运行时多处复用，每次变更形成新版本。

## 关键代码示例

```yaml
name: coding-agent
model:
  id: claude-opus-5
  effort: high
  inference_geo: us
system: You are a senior backend engineer. Prefer tests and clear code.
tools:
  - code_execution
  - web_search
mcp_servers:
  - github
skills:
  - python-best-practices
description: Coding assistant for the platform team
```

这段 YAML 是 agent 配置的最小示例，供 ant apply 声明式创建或更新。name 和 model 为必填；model 使用对象形式同时固定 id、effort 和 inference_geo，避免每次会话单独指定。system 定义 persona，tools、mcp_servers、skills 分别叠加预置工具、MCP 能力和领域技能。创建成功后平台会返回 id 和 version，后续 session 只引用 id。

## 关键流程

1. 确定 agent 的职责边界，选好 model、system prompt、工具、MCP 和 skills。
2. 通过 API、CLI 或 SDK 创建 agent，保存返回的 id 和 version。
3. 启动 session 时引用 agent id；需要临时调整时使用 session override，不影响 agent 本身。
4. 更新 agent 时先读取当前 version，交互式更新建议带 version 做乐观并发控制；CI 声明式同步可以省略 version。
5. 如需数据驻留合规，固定 inference_geo 并确认模型支持；multiagent 中所有成员的 pin 必须一致。

## 关键点

- Agent 把 model、system、tools、mcp_servers、skills 打包成一个带 id 和 version 的可复用资源，避免重复配置并支持审计。
- model 字段的字符串形式只是模型 ID，对象形式才能设置 speed、effort、inference_geo；session 覆盖时整个 model 对象会被替换。
- version 字段用于乐观并发控制：带 version 时版本不匹配返回 409，省略 version 则是 last-write-wins，适合声明式 CI。
- 更新语义是省略字段保留、scalar 字段替换，system 和 description 可传 null 清除，model 和 name 必填不能清空。
- inference_geo 固定用于合规和数据驻留，在保存、创建 session 和每轮推理时都会被校验，workspace 收紧后运行中的 session 也可能被拒绝。
- multiagent 字段声明可委派的下属 agent，适合把复杂任务拆给多个专业 agent 协同。

## 对比与权衡

- 相比每次 session 直接传 system/tools/mcp，agent 配置在多会话复用、版本管理和配置审计上更好，但临时灵活性略差，需要 session override 补充。
- 相比 OpenAI Assistants API 的 assistant/thread 抽象，Claude Managed Agents 更强调 MCP 工具生态和 skills 的渐进披露，但生态成熟度和第三方兼容性可能仍在演进。
- 相比用 LangChain/LlamaIndex 等框架自管 prompt 和工具链，平台托管 agent 在权限控制、数据驻留和基础设施代码上更好，但可移植性和自定义灵活性上不如框架方案。

## 自测问题

**问: agent 配置里 model 字段的字符串和对象两种形式有什么区别？**

字符串只是模型 ID；对象可以额外指定 speed、effort、inference_geo。重点说明 per-session model override 是整体替换，因此 session 内单独设置的 effort 不会生效；要固定 effort 应设置在 agent 的 model 对象上。

**问: 创建 agent 后返回的 version 有什么用？**

实现乐观并发控制，避免并发更新互相覆盖。带 version 更新时如果版本不匹配会返回 409，调用方重新读取再合并；省略 version 则是最后写入胜出，适合 CI 等声明式同步场景。

**问: 更新 agent 时，未出现的字段会被清空吗？**

不会。省略字段会保留。scalar 字段如 system、description 可以被 null 清除，model 和 name 必填不能清空；model 对象内如果 id 不变，省略 effort 会保留原 effort，id 改变则会重置为新模型默认。

**问: inference_geo 固定后有什么风险或限制？**

它主要用于合规和数据驻留。固定后会在 agent 保存、创建 session、每个 turn 校验 workspace 的 allowed_inference_geos；如果 workspace 策略收紧，新 session 无法创建，运行中会话也会拒绝继续。multiagent 的 coordinator 和成员必须保持一致。

**问: session override 和更新 agent 的本质区别是什么？**

session override 只影响单次会话，不改变 agent 版本，适合临时实验；更新 agent 会产生新版本，影响所有后续引用该 agent 的会话，适合规则性变更。面试时可以举例说明什么时候用哪一个。

## 适用场景

- 团队需要在多个 session 中统一客服、代码评审或数据查询助手的行为。
- 需要固定推理地理区域以满足数据驻留和合规要求。
- 通过 CI 声明式管理 agent 配置，自动同步并保留版本历史。
- 使用 multiagent 让协调者 agent 把任务委派给不同专业 agent。

## 标签

`Claude Managed Agents` `Agent 配置` `版本控制` `MCP` `inference_geo`
