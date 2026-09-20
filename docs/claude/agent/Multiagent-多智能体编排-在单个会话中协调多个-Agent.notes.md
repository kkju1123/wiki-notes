# Multiagent 多智能体编排：在单个会话中协调多个 Agent

*原文: [https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration) · 来源: web · 生成时间: 2026-09-20T07:46:24.289064+00:00*

## 背景

单 Agent 常需要把搜索、代码、安全、写作等多种能力塞进同一个 system prompt 和工具集，随任务变复杂会出现上下文过长、工具冲突、串行执行慢等问题。Multiagent orchestration 借鉴多角色协作思想，让一个 coordinator 在会话内动态委派多个专职 agent，每个 agent 拥有独立上下文和配置，从而在保持协作的同时实现隔离与并行。

## 痛点

不懂多智能体编排时，开发者容易把所有能力堆进单个 agent，导致上下文污染、输出质量下降、任务只能串行、定位问题困难；或者自行实现多服务编排，增加了会话、凭据、文件共享和状态同步的复杂度。

## 解决办法

核心做法是在 coordinator 的 agent 定义中声明 multiagent.agents roster，包含三类成员：按 ID/版本引用的外部 agent、self 副本、advisor 模型。coordinator 在 primary thread 报告进展，委派任务时平台为每个子 agent 创建独立 session thread，保留其完整会话历史；所有 agent 共享 sandbox、文件系统和 vault credentials，但工具、MCP server 和上下文不共享。通过版本快照固定被引用 agent 的行为，通过 advisor 在会话中途提供战略建议，并通过事件流把子 agent 活动和 advice 传回主线程。

## 关键代码示例

```json
{
  "name": "research-coordinator",
  "model": "claude-sonnet-4-5",
  "system_prompt": "You coordinate research and synthesis. Delegate independent subtasks to specialists.",
  "tools": ["web_search"],
  "multiagent": {
    "agents": [
      {"type": "agent", "id": "agent_sec_123", "version": "v1"},
      {"type": "self"},
      {"type": "advisor", "model": "claude-opus-4-5"}
    ]
  }
}
```

这段配置定义了一个 coordinator agent，并声明它可以委派给一个已固定的安全 agent、自身副本以及一个 advisor 模型。type=agent 的条目通过 version 固定行为，避免后续被意外更新；type=self 的副本会继承 session 级配置覆盖；type=advisor 使用更强模型在会话中途提供规划或审查建议。对应原理：coordinator 只做调度与汇总，具体隔离上下文由平台根据 roster 创建子线程完成。

## 关键流程

1. 定义 coordinator agent 时，在 multiagent.agents 中声明可委派的 roster，引用已有 agent ID 并固定 version，或使用 self/advisor。
2. 平台在保存或更新 coordinator 时对 roster 做校验：最多 20 个唯一 agent、只能一层委派、inference geography 必须一致。
3. 会话运行时，coordinator 根据任务模式（并行、专业化、升级）决定委派，平台为每个被委派 agent 创建独立 session thread。
4. 子 agent 在自己的上下文中执行并写入共享文件系统，coordinator 从返回结果或共享文件中收集信息，在 primary thread 汇总。
5. 如需中途获得战略建议，coordinator 可调用 advisor；advice 通过 thread events 传回 primary thread，不占用普通 roster 执行工作。

## 关键点

- 多智能体编排的核心是上下文隔离：每个子 agent 有独立 session thread 和会话历史，避免工具、system prompt 与中间结果互相污染。
- 协作依赖共享 sandbox、文件系统和 vault credentials，但 tools、MCP server、context 不共享，这是又隔离又能协作的关键边界。
- 版本快照非常重要：coordinator 创建/更新时会把 roster 中的 agent 引用固定到具体版本，保证行为可复现，不会随被引用 agent 更新而漂移。
- 委派模式主要包括并行化、专业化、升级三类：独立子任务同时跑，专业 agent 各管一摊，复杂子任务交给更强模型或更专职 agent。
- advisor 是特殊 roster 成员，保留名 anthropic.advisor，以平台生成的临时线程提供 mid-turn 咨询，适合规划、解除卡点和完成前审查。
- 多智能体系统不是默认选择：简单任务、强耦合步骤或需要完整共享上下文时，单 agent 或固定工作流更简单、成本更低、更易调试。

## 对比与权衡

- 相比单 Agent 把所有工具和提示词塞进一个上下文，多智能体方案在任务隔离、并行度和专业化上更好，但编排复杂度、延迟和调试成本更高。
- 相比固定顺序的 chain/workflow 编排，coordinator 动态委派更灵活，能根据中间结果改变策略，但执行路径更不确定，成本控制不如固定 pipeline。
- 相比直接使用 Messages API 的 advisor tool，Managed Agents 的 advisor roster 配置更简单、自动通过 thread events 传递 advice，但缺少 max_uses、max_tokens 和 caching 等细粒度控制。

## 自测问题

**问: 为什么每个子 agent 要有独立 session thread，而不是所有 agent 共享同一个上下文？**

上下文隔离可以防止工具定义、系统提示和中间推理互相干扰，提升专注度和输出质量；独立 thread 保留各自完整历史，便于 coordinator 后续追问；同时天然支持并行执行和事件级审计。代价是子 agent 之间不能直接共享内存，只能通过文件系统或最终结果沟通。

**问: roster 里按 ID 引用 agent 和 type=self 有什么区别？**

按 ID 引用是固定到某个 agent 和版本，其配置独立于 coordinator，适合使用专职 agent 且保证行为稳定；type=self 是复制 coordinator 自身配置产生的副本，session 级 agent configuration overrides 也会应用到这些副本，适合同一配置水平扩展并发处理。注意按 ID 引用不受 session override 影响。

**问: coordinator 为什么只允许一层委派？**

一层委派可以避免代理树过深导致的不可控传播、循环依赖和状态追踪爆炸，同时简化权限、成本和调试模型。若被引用 agent 自己也有 multiagent.agents，创建或更新会直接报验证错误；需要多层时应拆分顶层 coordinator 或在工作流层显式编排。

**问: advisor 和普通委派 agent 的差异是什么？**

advisor 是平台在 primary thread 中提供的临时顾问模型，保留名 anthropic.advisor，不执行工具、不占用普通 roster 名额，主要用于中途规划、解决卡点或完成前审查；普通委派 agent 是实际干活的上下文隔离执行单元。其 advice 以 thread events 返回，而不是 advisor_tool_result；也没有 max_uses、max_tokens、caching 等参数。

**问: 什么时候不该用 multiagent orchestration？**

当任务简单、步骤强耦合、需要频繁共享全部中间状态，或对延迟和成本敏感时，单 agent 更合适；当流程固定且可预先确定时，workflow/chain 比动态协调更可靠。判断标准通常看是否存在可并行、可边界清晰的子任务，以及是否需要不同系统提示/工具隔离。

## 适用场景

- 研究报告生成：coordinator 并行派多个 agent 搜索不同来源、分析不同文档，再汇总成一致结论。
- 代码审查与修复：一个 agent 负责安全检查，一个负责文档，一个负责生成测试，coordinator 综合形成完整改动。
- 高风险输出审查：在最终回复用户前，让 advisor 模型 review 工作产物或规划，减少错误。
- 多领域协作：法律、合规、财务等专业 agent 各自维护独立提示词和工具，由 coordinator 统一调度。

## 标签

`Multiagent` `Agent Orchestration` `Context Isolation` `Claude Managed Agents` `Advisor`
