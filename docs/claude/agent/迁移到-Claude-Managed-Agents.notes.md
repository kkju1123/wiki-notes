# 迁移到 Claude Managed Agents

*原文: [https://platform.claude.com/docs/en/managed-agents/migration](https://platform.claude.com/docs/en/managed-agents/migration) · 来源: web · 生成时间: 2026-09-20T07:05:22.765058+00:00*

## 背景

过去构建 Claude Agent 通常需要在客户端手写 while 循环：调用 Messages API、维护对话历史、分发工具调用并自行管理沙箱。Claude Agent SDK 抽象了部分复杂度，但仍在用户进程内运行。Claude Managed Agents 将这些能力托管到 Anthropic 服务端，提供持久化 Agent、会话状态、事件流和沙箱执行，降低自建 Agent 基础设施的运维负担。

## 痛点

手写 agent loop 容易在历史拼接、工具分发、终止条件判断上出错，且沙箱安全需要自己保障；Agent SDK 虽然简化了开发，但本地进程难以弹性扩展、状态不易跨实例共享。不懂迁移会造成重复造轮子，无法利用托管基础设施的可靠性。

## 解决办法

核心是把“自己维护循环”改为“事件驱动订阅”。先创建 Agent 定义（system、model、tools），再创建 Session，服务端自动存储历史。内置工具在沙箱内自动执行，自定义工具通过 `agent.custom_tool_use` 事件推给客户端，处理后回传 `user.custom_tool_result`。终态由 `session.status_idle` 事件标记。类比：从自己经营餐厅（采购、烹饪、洗碗全包）切换到中央厨房加定制化主厨，你只处理特殊订单。

## 关键代码示例

```python
# Before: 手写 Messages API loop
history = [{"role": "user", "content": prompt}]
while True:
    resp = client.messages.create(model="claude-sonnet-4-5", messages=history, tools=tools)
    history.append({"role": "assistant", "content": resp.content})
    if resp.stop_reason != "tool_use":
        break
    for block in resp.content:
        if block.type == "tool_use":
            result = execute(block.name, block.input)
            history.append({"role": "user", "content": [{"type": "tool_result", "tool_use_id": block.id, "content": result}]})

# After: Managed Agents 事件流
agent = client.beta.agents.create(model="claude-sonnet-4-5", system=prompt, tools=tools)
session = client.beta.sessions.create(agent_id=agent.id)
client.beta.sessions.messages.create(session_id=session.id, content=prompt)
for event in client.beta.sessions.events.stream(session_id=session.id):
    if event.type == "agent.custom_tool_use":
        result = execute(event.tool_name, event.input)
        client.beta.sessions.messages.create(session_id=session.id, content={"type": "user.custom_tool_result", "tool_use_id": event.tool_use_id, "content": result})
    elif event.type == "session.status_idle":
        break
```

Before 部分手动维护 history 数组，每次请求携带完整历史，循环内判断 `stop_reason` 并逐个执行 tool_use。After 部分只需创建 Agent 和 Session，之后通过事件流接收事件；自定义工具通过 `agent.custom_tool_use` 事件到达，处理完回复 `user.custom_tool_result`，内置工具在沙箱自动执行。`session.status_idle` 替代了手写终止判断，体现从循环控制到事件驱动的迁移。

## 关键流程

1. 创建具备所需网络和运行时环境的 Environment。
2. 将 system prompt 和工具选择迁移到 Agent 定义中。
3. 用 sessions.create 和 sessions.events.stream 替换手写循环。
4. 将 Agent 需要读取的本地文件通过 Files API 上传并挂载为 session resources。
5. 将自定义工具处理器移入事件循环，作为对 agent.custom_tool_use 事件的响应。
6. 先在测试会话中验证，再切换生产流量。

## 关键点

- 服务端会话历史：客户端不再拼接 messages 数组，状态由 Session 管理，可避免多轮交互中的状态漂移，也便于多客户端共享同一会话。
- 工具执行分工：内置工具由沙箱自动运行，自定义工具通过 `agent.custom_tool_use` 事件回调，职责边界清晰且减少样板代码。
- Agent 定义持久化与版本化：Agent 创建一次后服务端保存版本，更新会生成新版本，便于灰度发布、回滚且无需重新部署客户端。
- 事件驱动的终止条件：以 `session.status_idle` 作为循环结束信号，替代原先自行判断 `stop_reason`，更可靠且与流式事件模型统一。
- 部分 SDK 能力需客户端自建：Plan mode、PreToolUse/PostToolUse hooks、output styles、slash commands 等不再由 SDK 自动处理，迁移前需评估这些控制力的损失。
- 模型版本迁移成本低：通常只改 Agent 定义的 `model` 字段，下一会话生效，参数变更由运行时处理，客户端无需修改。

## 对比与权衡

- 相比 Messages API 手写循环，Managed Agents 在历史管理、工具分发、沙箱执行上更简单可靠，但失去了对 prompt 拼接顺序、每步调用和终止逻辑的直接控制。
- 相比 Claude Agent SDK，Managed Agents 将运行时托管到 Anthropic 基础设施，免去本地进程运维和文件系统依赖，但 SDK 的 hooks、输出样式、slash commands、max_turns 等需要迁移到客户端实现。
- 相比完全自建 agent 基础设施，Managed Agents 能更快落地并具备服务端版本管理，但定制深度和可观测性受限于平台接口。

## 自测问题

**问: 为什么把 agent loop 迁移到 Managed Agents？**

从状态管理、工具分发、沙箱安全、终止条件四个维度对比手写循环；强调事件驱动架构如何消除样板代码，并提到持久化 Agent 版本便于迭代和回滚。

**问: Managed Agents 中自定义工具和内置工具有什么区别？**

内置工具在服务端沙箱自动运行，客户端无感知；自定义工具通过 `agent.custom_tool_use` 事件到达客户端，需回复 `user.custom_tool_result`。这意味着自定义工具更灵活但要求客户端保持事件循环在线。

**问: Agent SDK 和 Managed Agents 的核心差异是什么？**

运行位置不同，SDK 在本地进程，Managed Agents 在 Anthropic 基础设施；配置从每次运行构造变为服务端持久化 Agent；工具分发从装饰器自动转为事件回调；本地文件需上传挂载到 session resources。

**问: 如何处理原来 SDK 的 PreToolUse/PostToolUse hooks？**

在自定义工具处理逻辑前后加代码；对内置工具可用 `permission_policy: always_ask` 拦截每次调用审批，或使用 `auto` 让服务端评估，但 `auto` 可能会在安全时直接执行而不经过客户端。

**问: 模型版本迁移要注意什么？**

通常只改 Agent 定义的 `model` 字段，下一会话生效；请求参数变化由运行时隐藏，无需改客户端；事件模型没有 assistant prefill，且 JSON 解析已由运行时处理，不需要额外适配。

## 适用场景

- 需要服务端托管的长时间运行 Agent，如自动化客服、数据流水线处理。
- 多会话并发场景，服务端统一管理历史状态，避免客户端间同步问题。
- 需要沙箱执行不可信生成代码的安全场景，比如代码解释器或自动化脚本运行。
- 希望将 Agent 配置纳入服务端版本治理，便于灰度发布、回滚和审计。

## 标签

`Claude Managed Agents` `Agent 迁移` `事件驱动架构` `Claude Agent SDK` `Messages API`
