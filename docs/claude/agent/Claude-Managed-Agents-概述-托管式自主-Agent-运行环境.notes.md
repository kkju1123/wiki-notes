# Claude Managed Agents 概述：托管式自主 Agent 运行环境

*原文: [https://platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview) · 来源: web · 生成时间: 2026-09-20T06:56:58.200543+00:00*

## 背景

自主 Agent 通常需要实现“模型调用→工具执行→观察结果→再次调用”的闭环，还要处理沙箱安全、上下文管理、状态持久化、异步队列与错误恢复。Claude Managed Agents 把这些通用工程能力打包成托管服务，让开发者从直接操作模型接口升级到使用生产级 Agent 运行时。

## 痛点

直接用 Messages API 构建长时 Agent 时，开发者要自己写 agent loop、工具执行层和沙箱，容易在上下文超长、任务中断、并发限制等问题上反复踩坑。无状态 API 也无法自然保存会话文件与历史，导致长任务恢复和多人协作非常麻烦。

## 解决办法

Managed Agents 把 Agent 定义为模型+系统提示+工具+MCP+技能的配置单元，用 Environment 选择云沙箱或自托管沙箱，再通过 Session 启动一个可持久化的运行实例。服务端内置 agent loop：Claude 自主调用 Bash、文件操作、Web 搜索/抓取和 MCP 工具，通过 SSE 流式返回事件，并在服务端保存事件历史和沙箱文件。内置 prompt caching 与 compaction 控制上下文成本，会话可暂停、恢复、中途引导或中断。类比：它像 Agent 领域的 PaaS，你不需要自己拼装 worker、队列和沙箱。

## 关键代码示例

```python
import httpx

BASE = 'https://api.anthropic.com/v1/managed-agents'
HEADERS = {
    'x-api-key': 'sk-ant-...',
    'anthropic-version': '2023-06-01',
    'anthropic-beta': 'managed-agents-2026-04-01',
    'content-type': 'application/json',
}

agent = httpx.post(BASE + '/agents', headers=HEADERS, json={
    'model': 'claude-sonnet-4-5',
    'system': 'You are a research assistant.',
    'tools': ['bash', 'file', 'web_search', 'web_fetch'],
}).json()

env = httpx.post(BASE + '/environments', headers=HEADERS, json={
    'type': 'cloud-sandbox',
    'packages': ['pandas', 'requests'],
}).json()

session = httpx.post(BASE + '/sessions', headers=HEADERS, json={
    'agent_id': agent['id'],
    'environment_id': env['id'],
}).json()

url = BASE + '/sessions/' + session['id'] + '/events'
with httpx.stream('POST', url, headers=HEADERS, json={'type': 'user', 'content': 'Analyze the CSV'}) as r:
    for line in r.iter_lines():
        if line.startswith('data:'):
            print(line)
```

这段示例展示四个核心步骤：创建 Agent 定义模型/工具/系统提示，创建 Environment 指定运行沙箱，启动 Session 获得可持久化实例，最后用流式 POST 发送用户事件并通过 SSE 接收结果。以上端点和字段为示意，实际需以官方 API 文档为准。与 Messages API 的关键区别在于：你不需要自己维护 agent loop 和工具调用状态，服务端会持久化 session 并在事件流中返回工具调用、文件变更等。

## 关键流程

1. 创建 Agent：定义模型、系统提示、工具、MCP servers、skills
2. 创建 Environment：配置云沙箱或自托管沙箱
3. 启动 Session：引用 agent 和 environment 配置
4. 发送事件并流式接收响应：用户消息作为事件，Claude 自主运行工具，SSE 返回结果，历史持久化
5. 引导或中断：发送额外用户事件或中断执行改变方向

## 关键点

- 核心抽象是 Agent/Environment/Session/Events，分别对应“定义能力”“运行位置”“执行实例”“通信协议”，理解这四层是使用和排查问题的基础。
- 内置 agent loop 与工具执行沙箱，开发者不需要自己实现“模型-工具-观察”循环，因此特别适合长时异步任务。
- 状态化设计既有优势（持久文件系统、会话历史、可恢复）也有代价：不满足 ZDR 和 HIPAA BAA，合规敏感场景需评估。
- 支持 Bash、文件操作、Web search/fetch 以及 MCP servers，其中 MCP 让外部工具生态可插拔，是扩展能力的关键。
- 适用于需要云基础设施、定时执行、最小基础设施投入和自托管合规的场景。

## 对比与权衡

- 相比直接使用 Messages API，Managed Agents 在开发效率、长时任务、状态管理和工具执行上更省心，但灵活性和底层控制不如 Messages API，且有状态存储导致不满足 ZDR/HIPAA。
- 相比自建 agent 框架（如 LangGraph、自研 loop），Managed Agents 免去维护沙箱、队列、上下文压缩等基础设施，但可定制性和运行环境控制可能受限，深度定制仍可回退到 Messages API。
- 相比云沙箱，自托管沙箱更满足数据驻留和合规要求，但需要自己承担基础设施运维。

## 自测问题

**问: Claude Managed Agents 和 Messages API 的本质区别是什么？**

Messages API 是无状态的模型调用接口，开发者需要自己实现 agent loop、工具执行和上下文管理；Managed Agents 是托管的有状态 Agent 运行时，提供 Agent/Environment/Session/Events 抽象，内置沙箱、SSE 事件流和持久化。类比：前者像买服务器自己装系统，后者像用托管容器服务。

**问: 四个核心概念分别解决什么问题？**

Agent 把“模型+提示+工具+技能”打包成可复用配置；Environment 解耦运行位置，支持云托管或私有化部署；Session 是一次具体任务的有状态实例，保存文件与历史；Events 是客户端与运行中 agent 的通信协议，支持流式反馈和干预。

**问: 为什么状态化设计导致不满足 ZDR 和 HIPAA BAA？**

ZDR 要求 Anthropic 不保留请求/响应数据，而 Managed Agents 需要在服务端持久化 session 历史和沙箱文件系统才能支持长时运行、暂停恢复。HIPAA BAA 对受保护健康信息的处理有严格约束，托管沙箱和持久化存储可能超出 BAA 覆盖范围。敏感场景应使用 Messages API 配 ZDR，或自托管沙箱加自己的合规控制。

**问: 如果要中断一个正在运行的 session 并改变方向，应该怎么做？**

可以发送额外的用户事件来引导 agent，或者调用中断接口终止当前执行。由于事件历史是服务端持久化的，中断后可以基于已保存状态恢复或重新定向，而不需要从头重跑。

**问: MCP servers 在 Managed Agents 中的作用？**

MCP 是标准化外部工具协议，允许 agent 连接第三方工具提供方，而不局限于内置的 Bash、文件和 Web 工具。通过配置 MCP servers，可以把企业私有 API、数据库等能力接入 agent 的工具集，扩展其能力边界。

## 适用场景

- 长时自动化任务：如代码库批量重构、数据分析、报告生成，运行几分钟到几小时。
- 需要网络访问和预装包的云端沙箱环境：如抓取网页、执行浏览器自动化、安装依赖跑脚本。
- 定时任务/流水线：用 scheduled deployments 定期运行 agent，如定时巡检、日报生成。
- 合规要求需要私有化运行：使用自托管沙箱在自有基础设施中执行。

## 标签

`Claude API` `Managed Agents` `Agent 运行时` `长时任务` `Anthropic`
