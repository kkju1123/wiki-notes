# Agent Harness 完全指南：LLM Agent 的控制平面与基础设施

*原文: [https://medium.com/gitconnected/agent-harness-a-no-bs-guide-8f69e3c0a3da](https://medium.com/gitconnected/agent-harness-a-no-bs-guide-8f69e3c0a3da) · 来源: medium · 生成时间: 2026-09-17T08:49:28.193472+00:00*

## 背景

2023—2024 年模型竞赛趋于饱和，模型从简单问答转向可调用工具、记忆和规划；用户也从提问升级为指派长任务。要让这些能力可靠协作，仅靠模型本身不够，需要统一的运行与控制层，于是 Agent Harness 作为 LLM 外围的“操作系统”出现。Anthropic 数据也显示，agent 执行时长正在快速上升，长任务治理变得必要。

## 痛点

没有 harness 或 harness 不完善时，长任务 agent 可能无限循环、超 token/compute 预算、静默失败、误用危险工具、丢失目标与上下文；这不仅造成成本失控，还会带来安全与信任问题，使 agent 难以生产化。

## 解决办法

Harness 把 Agent 拆成 state、tools、planning、memory、orchestration、runtime、security/governance、observability/evaluation 等组件，并实现 reasoning→planning→choose tool→execute→observe→repeat/stop 的控制循环。通过步骤/时间/token/compute 预算、工具 allowlist 与最小权限、检查点和轨迹评估，给模型加上安全边界与闭环控制。类比：LLM 是大脑，harness 是包围大脑的神经系统和操作系统。

## 关键代码示例

```python
class AgentHarness:
    def __init__(self, llm, tools, memory, budget):
        self.llm, self.tools = llm, {t.name: t for t in tools}
        self.memory, self.budget, self.state = memory, budget, {}

    def run(self, task):
        self.state.update(task=task, steps=[], observations=[])
        plan = self.llm.plan(task, self.memory.retrieve(task))
        while not self.state.get('done'):
            if self.budget.is_exceeded(self.state):
                raise BudgetExceeded('budget exceeded')
            action = self.llm.choose_action(plan, self.state)
            if action.name not in self.tools:
                raise UnknownTool(action.name)
            result = self.tools[action.name](**action.args)
            self.state['steps'].append(action.name)
            self.state['observations'].append(result)
            self.memory.save(action, result)
            plan = self.llm.revise_plan(plan, self.state)
            self.state['done'] = self.llm.should_stop(self.state)
        return self.state
```

这段伪代码体现了 Harness 的主循环：先初始化当前任务状态，再从 memory 检索并生成 plan；每一步先做预算检查，再做工具允许列表校验，避免幻觉/越权工具；执行后写回 state 与 memory，最后修订计划并判断是否停止。BudgetExceeded 和 UnknownTool 就是生产 harness 中的 guardrails。

## 关键流程

1. 解析任务并初始化 AgentState
2. 从 Memory 检索相关信息并生成或修订计划
3. 循环选择工具/动作，先做预算与权限校验
4. 执行工具，把观察结果写回状态与记忆
5. 评估目标是否达成，或触发 stop/budget/eval 条件后退出

## 关键点

- Agent Harness 不是模型本身，而是围绕 LLM 的控制平面/运行时，统一管理状态、工具、规划、记忆、安全、评测和可观测；面试时要把它和模型能力区分开。
- 长任务必须设置 time/token/compute/cost 等多维预算，否则一个死循环可能在几小时内烧掉大量资源；预算要进入主循环而不是只在外层兜底。
- 工具 allowlist 和最小权限是安全底线：即使模型幻觉出工具名也无法执行未注册工具，危险操作应引入 human-in-loop。
- State 管当前任务进度，Memory 管跨步骤/会话记忆；两者不能混用，否则会出现上下文丢失或状态污染。
- Observability/evaluation 要内建于 harness，而不只是看最终结果；需要记录 trajectory、tool call、中间观察和评估信号，便于回归和排障。

## 对比与权衡

- 相比裸调 LLM，Agent Harness 在状态管理、工具控制、预算和可观测性上更好，但会显著增加系统设计、延迟和运维复杂度。
- 相比 LangChain/CrewAI 这类编排框架，专职 harness 在运行时治理、安全策略与生产观测上更强，但开发成本更高、组件整合更费劲。
- 相比单一 agent 的简单 loop，sub-agent harness 能通过分层规划/执行处理更复杂任务，但在调试分布式 agent、状态同步和错误隔离上更难。

## 自测问题

**问: Agent Harness 和 LangChain、CrewAI 这些框架到底有什么区别？**

框架主要解决构建和编排，harness 更接近运行时/控制平面，覆盖 agent loop、状态、工具 allowlist、预算、安全、评测、可观测等生命周期。框架可以成为 harness 的某个组件，但生产级 harness 一般要自己在框架外补治理和监控。

**问: 一个 agent 任务跑了 12 小时还没结束，你会怎么排查和治理？**

优先看 harness 的 budget/step limit 是否生效；检查 state 和 trajectory 是否有重复 action、计划未收敛、tool 返回空转；用 trace 定位 loop；治理上设置 time/token/compute/cost 预算、最大步数、heartbeat、checkpoint、circuit breaker，以及危险工具 human-in-loop。

**问: 如果 LLM 幻觉出一个工具，或者调用删除文件的工具，harness 怎么防？**

工具调用必须经过注册工具 allowlist 校验，未注册工具直接拒绝并返回可观测错误；高危工具用最小权限、sandbox、参数校验、审批卡点；记录每次 tool call 和权限决策用于审计。

**问: State 和 Memory 的边界是什么？Memory 里放向量检索结果就够了？**

State 是当前任务执行上下文：完成到第几步、当前 plan、最近 observations；Memory 是跨步骤/会话长期信息，通常包括工作记忆、情节记忆、语义/程序记忆。向量检索只是召回，还需要摘要、压缩、遗忘/淘汰机制，避免把原始长上下文全部塞进 prompt。

**问: 如何评估一个 Agent 系统，而不只看它“像不像做完了”？**

要做 trajectory 级评估：任务成功率、子目标达成率、工具调用正确率、重试次数、预算/成本、latency、安全违规率；回放关键 trace，构建 eval set，在 harness 内嵌评分器，避免只对最终答案评分。

## 适用场景

- 长时间 coding agent：修 issue、跨文件改代码，需要 repo 工具、沙箱、token budget、状态管理和检查点/回滚。
- 深度研究/浏览器自动化 agent：搜索、浏览、总结，容易跑飞，需要步骤预算和轨迹记录。
- 企业多工具 agent（客服/RPA/数据查询）：访问 API/DB/内部系统，需要细粒度权限、审计和 human-in-loop。
- 多 sub-agent 编排：planner、executor、critic 等分层结构，需要统一运行时和可观测。

## 标签

`AI Agent` `Agent Harness` `LLM` `Orchestration` `Observability`
