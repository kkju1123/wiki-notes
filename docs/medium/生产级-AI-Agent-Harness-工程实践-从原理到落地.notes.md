# 生产级 AI Agent Harness 工程实践：从原理到落地

*原文: [https://medium.com/@tort_mario/ai-agent-best-practices-production-ready-harness-engineering-2026-guide-c1236d713fac](https://medium.com/@tort_mario/ai-agent-best-practices-production-ready-harness-engineering-2026-guide-c1236d713fac) · 来源: medium · 生成时间: 2026-09-16T14:01:03.806167+00:00*

## 背景

LLM 能力越来越强，但把它们真正丢到生产环境做客服、财务分析、DevOps 自动化时，绝大多数 Agent 会失败——原因往往不在模型本身，而在包裹模型的『运行外壳』（harness）写得又脆又不安全。Harness 就是模型与真实世界之间的确定性中间层：模型只负责『提议』要调用什么工具，Harness 负责校验、授权、执行、日志、预算控制。这个方向在 2025-2026 年随着 Claude Code、Codex、Agent Skills 等工具链的成熟，逐渐形成了一套可复用的工程范式。

## 痛点

没有规范的 Harness，Agent 就是黑盒：可能陷入死循环疯狂烧 token、被 prompt injection 诱导执行任意命令、调用越权工具造成不可逆破坏、任务超时却没有任何结构化失败信号。更麻烦的是，出了问题是模型『不听话』还是工程层没兜住，往往分不清楚，无法定位。

## 解决办法

核心思想是『模型提议、Harness 执行』的职责分离，类似操作系统内核与用户态程序的关系——模型是不被信任的应用，Harness 是带权限检查、系统调用校验、资源配额的内核。具体做法包括：模型只输出结构化 tool call，Harness 校验 schema、检查权限矩阵、执行并把结构化观察结果注入回上下文（即使失败也一定返回结果）；按只读/草稿/外部写三档风险分级，外部写必须先 draft 后 commit；上下文分层装配而非全量堆叠；每轮循环都设步数、时间、token、成本四类预算；反复出现的失败模式不要靠改 prompt，而要固化为 Harness 里的校验器或工具。

## 关键流程

1. Map：先明确领域（客服/财务/DevOps）、自主等级（0-4 级）、风险级别（只读/财务/破坏性）、要接入的外部系统。
2. Identify：根据上一步选择 MVP 级别，首次做 Agent 通常选 Level 1（外部写均需人工审批）或 Level 2（计划审批后低风险步骤可自主执行）。
3. Blueprint：生成 Harness 设计蓝图，包含目标与边界、agentic loop（停止条件、预算）、工具注册表（带类型 schema 与风险类）、权限矩阵、上下文与记忆分层、Skills 与连接器。
4. Implement：按蓝图实现 15 个组件模块（指令管理、上下文构建、模型适配、工具注册、权限解析、预算追踪等），用伪代码把 agentic loop 的预算、压缩触发点、停止条件固化下来。
5. Launch：上线前跑安全评估（注入、超时、过度调用工具），验证的是 Harness 本身而不只是模型。

## 关键点

- 模型永远不直接调用工具，必须由 Harness 校验 schema、检查权限后再执行——这是防 prompt injection 升级为任意代码执行的关键闸门。
- 每一次工具调用无论成功、拒绝还是超时，都必须返回结构化 observation，绝对不能出现悬空的调用，否则 Agent 状态机会卡死或产生幻觉。
- 风险分级采用 draft-commit 模式：只读可自主、草稿只做内部模拟、外部写必须显式批准，让危险动作先预演再落地。
- 上下文是分层『装配』不是全量『灌入』：策略层、任务级指令层、运行时 JIT 提示层分离，并对不可信数据打 trust 标签区别对待。
- 四类预算（步数、墙钟时间、每轮与累计 token、USD 成本）是生产 Agent 的保险丝，超限必须优雅终止并返回结构化失败而不是崩溃。
- 反复出现的失败应该在 Harness 里固化为校验器或工具，而不是靠调整 prompt 打补丁——工程韧性优于提示工程。

## 对比与权衡

- 相比把逻辑全塞进 prompt 的『prompt-only 方案』，Harness 工程在安全性、可观测性和成本可控性上明显更好，但开发和维护成本更高，团队需要像做后端服务一样对待 Agent。
- 相比直接绑定某家厂商的 Agent SDK（如 OpenAI Assistants、Anthropic 的原生工具调用封装），provider-neutral 的 Harness 在可迁移性和长期可控性上更强，但享受不到厂商最新特性的即时集成。
- 相比 LangChain/LlamaIndex 这类高层框架，自建 Harness 在权限矩阵、预算、审计这些生产级关切上可控性更高，但需要自己造轮子，前期上手慢。

## 面试可能会问

**问: 什么是 Agent Harness？它和普通 Agent 框架有什么本质区别？**

Harness 是模型与外界之间的确定性运行时层，强调职责分离——模型只提议、Harness 执行。区别于框架在于它把『校验、授权、执行、日志』作为第一等公民，而不是框架那样主要提供编排抽象；可以类比内核 syscall 层。

**问: 为什么不能让 LLM 直接调用工具？**

LLM 输出不可信，prompt injection 可以诱导它生成任意参数。中间必须有 schema 校验、权限检查、参数白名单，才能阻止注入升级为任意代码执行或数据外泄。这就是 Confused Deputy 问题在 Agent 场景的体现。

**问: 如何防止 Agent 在生产环境里陷入死循环烧钱？**

四类预算——步数、时间、token、成本，任一超限由 Harness 结构化终止。此外要有重复动作检测（同一 tool call 连续多次）和渐进式 compaction，避免上下文膨胀。

**问: draft-commit 模式解决了什么问题？**

把『产生副作用之前先可回滚』这件事变成一等流程。Agent 先产出 draft（内部模拟、无外部影响），人工或策略批准后才 commit 到外部系统，极大降低不可逆破坏的风险，也便于审计。

**问: 上下文压缩（context compaction）应该什么时候触发？**

按 token 预算水位触发（如使用超过 70%），而不是固定轮数；压缩要保留策略层和最近的关键 observation，摘要中间历史。压缩策略本身要作为 harness feature 做测试，避免摘要丢关键约束。

## 适用场景

- 正在从 demo 走向生产的 Agent 团队，需要一套检查清单来审计现有 Harness 的安全性与预算控制。
- 平台/架构师设计公司内部的 Agent 运行时标准，需要 provider-neutral 的组件模型与权限方案参考。
- 面试准备：回答『如何设计一个生产级 Agent 系统』这类系统设计题时，把 Harness 分层、预算、draft-commit 作为答题骨架。
- 做安全评估（red team）时，参照该 skill 的 security evals 清单测试注入、超时、过度调用工具等失效模式。

## 标签

`AI Agent` `Harness Engineering` `LLM 生产实践` `Agent 安全` `系统设计`
