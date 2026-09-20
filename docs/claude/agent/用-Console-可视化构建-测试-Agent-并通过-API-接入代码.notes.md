# 用 Console 可视化构建、测试 Agent，并通过 API 接入代码

*原文: [https://platform.claude.com/docs/en/managed-agents/onboarding](https://platform.claude.com/docs/en/managed-agents/onboarding) · 来源: web · 生成时间: 2026-09-20T07:02:37.044593+00:00*

## 背景

当 Agent 从单一 prompt 发展为模型、系统提示、工具、MCP 与 Skills 的组合时，纯代码配置和调试变得繁琐。Console 这类可视化工作台因此出现，它把同一份配置暴露为 GUI 表单与等价 API 请求，让开发者先交互式调通，再落到代码。

## 痛点

没有这种可视化构建/测试入口，开发者要在代码里反复修改 JSON 或 SDK 参数；工具/MCP 鉴权、事件流调试都要另起脚本，反馈慢且容易产生配置漂移。

## 解决办法

Console 把 Agent 定义拆成模型与系统提示、MCP servers、tools、skills 等字段；每个字段实际上都映射到 API 请求参数，因此配置时会同步显示等价 API 请求。内置 session runner 让开发者直接发消息并观察事件流，快速验证提示词和工具选择。调通后无需把整套配置搬到业务代码，只需复制 agent_id 和 environment_id 并在创建 session 时引用，相当于用标识符绑定一套已保存的托管 Agent 配置，配置与运行解耦。

## 关键代码示例

```python
import os
import requests

api_key = os.environ["ANTHROPIC_API_KEY"]
agent_id = "agent_01H..."       # 从 Console 复制
environment_id = "env_01H..."   # 从 Console 复制

# 创建托管 Agent 会话：只传 ID，不重复传 prompt/tools/MCP 配置
resp = requests.post(
    f"https://api.anthropic.com/v1/agents/{agent_id}/sessions",
    headers={
        "x-api-key": api_key,
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
    },
    json={"environment_id": environment_id},
)
session = resp.json()
print(session["id"])
```

这段代码先通过 agent_id 和 environment_id 创建会话，这是从 Console 到代码的关键桥接：业务代码不保存 prompt、tools、MCP 认证等细节，只引用 Console 中已测试好的配置。之后在该会话中发消息时，托管 Agent 会按 Console 里配置好的模型、工具和 Skills 自动执行。示例重点是 ID 引用模式，具体 API 路径或 SDK 名称以 Anthropic 官方当前文档为准。

## 关键流程

1. 打开 Console 的 agent quickstart 页面，选择模型并编写 system prompt。
2. 通过 URL 添加远程 MCP servers 并完成认证，让 Agent 能访问外部系统。
3. 按需添加预构建 Tools 和 Anthropic/自定义 Skills，扩展 Agent 能力。
4. 查看 Console 同步生成的等价 API request，确认配置能直接落到代码。
5. 使用 inline session runner 发送测试消息，观察事件流和工具调用是否符合预期。
6. 调通后复制 agent_id 与 environment_id，在业务代码中创建 session 引用该 Agent。

## 关键点

- Console 的每个配置字段都对应最终 API 请求中的参数，这让 GUI 调试结果可以直接翻译成生产调用，降低配置漂移风险。
- MCP server 是连接外部数据/工具的标准化协议，在 Console 中通过 URL 添加并授权，使 Agent 能代表用户跨系统执行动作，而不是只会生成文本。
- Tools 和 Skills 是两种互补扩展：Tools 偏向单个可调用能力，Skills 偏向封装好的领域流程或组织知识；组合使用能避免把所有逻辑塞进 system prompt。
- Inline session runner 提供消息发送和事件流观察，能在进入代码前快速发现提示词、工具选择或 MCP 连接问题。
- 代码侧只引用 agent_id 和 environment_id，而不是复制整套配置，保持配置与业务代码解耦，便于非工程师协作和后续版本管理。

## 对比与权衡

- 相比在代码中手写 Messages API 的 system/tools 配置，Console 模式在快速试验和协同配置上更友好，但配置可版本化、CI 集成和复杂分支控制不如 as-code 方案。
- 相比直接用 Claude Code/Agent SDK 以脚本方式构建 Agent，Console 更偏可视化原型和团队演示，但不容易做代码级单测与自动化回归。
- 相比普通模型 Playground，Console 增加了 MCP、Skills、环境 ID 等托管能力，更接近生产 Agent 生命周期管理。

## 自测问题

**问: Console 里的 Agent 配置和代码里的 API 请求是什么关系？**

两者本质是同一配置的不同表示。Console 的字段会映射成 API 参数；测试完成后用 agent_id/environment_id 引用，而不是复制配置，避免配置漂移。

**问: MCP server 为什么需要认证？在 Console 中如何工作？**

MCP 允许模型访问外部工具/数据源，认证是为了让 Agent 代表用户安全执行动作；Console 通过 URL 添加并保存凭据，常见做法是使用最小权限 token，生产环境用密钥管理服务。

**问: 如何判断一个 Agent 是否值得从 Console 迁到代码？**

先在 Console 用 inline session runner 验证提示词、工具与 MCP 连接；当行为稳定后，把 agent_id/environment_id 接入代码；若需要复杂流程控制、版本化、自动化测试，再考虑把配置迁移为 as-code。

**问: Tools 和 Skills 有什么区别？**

Tools 更偏模型可调用的离散函数/API，Skills 更偏打包好的领域流程或组织知识；两者都能扩展 Agent，但 Skills 复用性更强，Tools 更原子。

**问: environment_id 的作用是什么？**

它标识一套已保存/已部署的 Agent 配置版本，用来区分草稿、测试和生产环境；代码引用 environment_id 可以避免改配置影响线上。

## 适用场景

- 快速验证客服/工单 Agent 的提示词与工具组合，无需先写工程代码。
- 产品、运营等非工程角色在 Console 配置 MCP/Skills，工程师只需复制 ID 接入业务系统。
- 演示或培训 Agent 能力时，用可视化界面实时展示事件流和工具调用。
- 需要复用企业 Skills 库，快速组装一个领域 Agent。

## 标签

`Claude Console` `Agent` `MCP` `可视化配置` `LLM 工程`
