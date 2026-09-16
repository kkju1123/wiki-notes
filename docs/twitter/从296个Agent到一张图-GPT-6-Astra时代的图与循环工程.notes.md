# 从296个Agent到一张图：GPT-6 Astra时代的图与循环工程

*原文: [https://x.com/N01ennn/status/2096962591125905888](https://x.com/N01ennn/status/2096962591125905888) · 来源: twitter · 生成时间: 2026-09-16T11:39:29.279136+00:00*

## 背景

随着Claude Code、Codex等工具让编码Agent大规模落地，社区开始讨论如何让Agent自主循环工作而不再靠人一句句提示。但很多团队把Harness（执行环境）、Loop（循环验证）、Graph（流程图拓扑）三个概念混为一谈，导致大规模Agent部署既昂贵又难以可靠性验证。这篇文章基于arXiv论文与两项大规模实证扫描，试图澄清这三层的分工与边界。

## 痛点

如果不区分这三层，就会画出无法解释行为的流程图、让同一个模型给自己打分、写出没有终止条件的重试循环、把Harness塞满工具和过宽权限，然后把本属于编排层的问题甩锅给模型。实证扫描发现，36,710个仓库中只有0.59%真正在跑自主Agent循环，而6,549个Agent仓库里有68个确认存在无限循环失败，95.6%造成API成本耗尽。

## 解决办法

把系统拆成三层：Harness负责模型能访问什么、执行如何被控制；Loop负责用显式触发器和基于证据的停止规则做observe/act/verify循环；Graph用节点和边建模工作流拓扑，控制下一步分支、并发、状态转换与恢复。核心思想是用证据而非模型自评来判定完成，用图结构把296个worker变成一张可检查的控制流图，而不是296套独立策略。

## 关键点

- Harness、Loop、Graph不是同义词：Harness让模型能跑，Loop让执行可迭代可恢复，Graph让控制流可检查，混用会导致昂贵且难以定位的失败。
- Ralph Wiggum循环的核心技巧是每次迭代用全新上下文重启Agent，并把进度持久化到文件系统——Agent会忘，仓库不会忘。
- 实证扫描发现社区共识里推荐的state file、verifier subagent、budget、stop condition几乎无人提交，因为大多数循环跑的是PR审查和issue triage这类无状态任务。
- IA L-Scan发现68个确认的无限Agent循环，LangGraph和AutoGen贡献了45个，因为它们用API编码反馈路径，代码里看不到while True，只看到add_conditional_edges。
- 所有主流框架都已内置max_iterations/max_turns/recursion_limit等限制，但IA L失败的本质是边界没有覆盖真正重复的路径——限制设在内层调用而外层评估循环自由运行。
- 验证成本决定委派边界：自主性只能提升到你能以可接受成本检查结果的程度，这是把296收敛为一张图时最实用的约束。

## 适用场景

- 正在设计多Agent编码工作流，纠结该用图还是用循环、该给Agent多大自主权时。
- 发现Agent跑得又贵又慢、无法解释上周二系统到底做了什么，需要定位是Harness、Loop还是Graph层的锅时。
- 要用LangGraph或AutoGen搭自动化流程，担心出现看不见的无限循环和API成本失控时。
- 想评估现有仓库的Agent循环是否真的在生产运行，而不是只在README里存在时。

## 标签

`Agent工程` `Graph Engineering` `Loop Engineering` `多Agent编排` `LLM可靠性`
