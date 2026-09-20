# 面向递归自改进的 Harness 工程

*原文: [https://lilianweng.github.io/posts/2026-07-04-harness/](https://lilianweng.github.io/posts/2026-07-04-harness/) · 来源: web · 生成时间: 2026-09-20T08:41:14.293484+00:00*

## 背景

递归自我改进（RSI）可追溯到 I. J. Good 对“超智能机器”的设想，Yudkowsky 进一步把它描述为“用当前智能改进产生智能的认知机器”的反馈回路。现代 AI 中，RSI 未必表现为模型直接改权重，而更多表现为改进训练管线与部署系统；编码 agent（Claude Code、Codex）等产品证明，模型与真实世界之间的 harness 层对最终能力贡献巨大。

## 痛点

如果只关注裸模型能力而忽略 harness，长程任务会很快遇到上下文爆炸、中间状态丢失、失败后无法恢复、多个实验无法并行管理等问题。没有持久状态和评测闭环，模型只能在单次问答中“聪明”，无法沉淀轨迹和自动改进，更无法支撑近中期的 RSI。

## 解决办法

把 harness 视为模型之上的“操作系统”：封装工具、权限、上下文、持久状态和并发执行，向模型暴露简单稳定的接口。核心循环是 goal-oriented 的 plan → execute → observe/test → improve，每次行动和结果都作为文件、diff、日志落盘，而不是全部塞进上下文；文件系统是 LLM 最熟悉的持久记忆形态。需要并行时，harness 以子代理/后台作业方式运行，父代理像进程管理器一样检查、取消、合并结果。这样优化的不是单次答案，而是“获取答案的机制”，使自改进在可审计、可恢复、可工程化的系统层发生。

## 关键代码示例

```python
from pathlib import Path
import subprocess

class Harness:
    def __init__(self, workdir):
        self.workdir = Path(workdir)
        (self.workdir / 'logs').mkdir(parents=True, exist_ok=True)

    def run(self, goal, max_steps=50):
        state = {'summary': '', 'artifacts': []}
        for _ in range(max_steps):
            action = model.choose_action(goal, state['summary'])
            result = self.execute(action)
            self.append_log(action, result)
            state['summary'] = model.summarize(self.tail('logs', 20))
            self.save_state(state)
            if self.goal_met(goal, result):
                return state
        raise RuntimeError('goal not reached')

    def execute(self, action):
        p = subprocess.run(action.cmd, shell=True, capture_output=True, text=True)
        log = self.workdir / 'logs' / f'{action.id}.log'
        log.write_text(p.stdout + p.stderr)
        return {'code': p.returncode, 'log': str(log.relative_to(self.workdir))}
```

这段代码演示了一个最小可用的 harness 主循环：run 按 goal 循环选择动作并执行，执行结果通过 append_log 落盘、state_summary 只保留近期日志摘要，避免上下文膨胀；execute 用 bash 运行命令并把 stdout/stderr 写到 logs/<id>.log，返回日志路径而不是全文。这个结构对应正文三个模式：目标循环自动化、文件系统持久记忆、可扩展并行子代理的落盘基础。

## 关键流程

1. 明确目标与验收标准，必要时向用户澄清任务规范和执行偏好。
2. 模型按目标生成计划并选择工具执行，形成 plan → execute → observe/test 循环。
3. 把动作、日志、结果、diff 和状态写入文件系统，使进度可恢复、可审计。
4. 分析自身轨迹和失败用例，改进下一步行动，直到目标达成。
5. 遇到多假设或独立子任务时，启动子代理/后台作业并行执行，并将结果文件化后合并回主线程。

## 关键点

- Harness 是连接裸模型与真实世界任务的运行时层，它组织思考、工具、记忆和评估，因此对长程任务的影响不亚于模型原始智能。
- 近中期 RSI 更可能发生在训练管线和部署系统层面，而不是直接修改模型权重；这让自改进更可审计、更安全，也更贴近工程现实。
- 文件系统是适合 LLM 的持久记忆：模型预训练已大量掌握 bash/文件编辑，日志与状态落盘能避免上下文爆炸并支持中断恢复。
- 并行子代理的结果必须显式写为文件、日志或状态记录，否则很快变为不可见的瞬时上下文；父代理需要像进程管理器一样检查、取消和合并。
- 通用且简单的工具接口（bash、git、patch、MCP）优于复杂专用 DSL，因为模型能复用软件工程预训练知识，也更容易行业标准化。
- harness 设计的长期方向是优化“获取答案的机制”而不仅是单次答案，成熟后部分能力可能内化回核心模型。

## 对比与权衡

- 相比早期 agent 框架（LLM + memory + tools + planning + action），harness 工程在持久状态管理、评测、权限和工作流设计上更完整，适合长程任务，但系统复杂度和运维成本更高。
- 相比将所有日志留在上下文窗口，文件系统持久记忆在可扩展性、可恢复性和多任务合并上更好，但要求模型具备可靠的工具调用和文件编辑能力。
- 相比模型直接改写自身权重的强 RSI，基于 harness 的自改进在可审计性、可回滚和工程可行性上更好，但自动化程度和理论上限更低。
- 相比高层领域专用 API，bash/git/apply_patch 等低层通用工具在泛化能力和模型兼容性上更好，但需要沙箱、权限控制和更谨慎的安全治理。

## 自测问题

**问: harness 和常见的 agent framework 到底有什么区别？**

早期 agent framework 更像把 LLM、memory、tools、planning、action 组合起来的提示工程；harness 则偏向 runtime/OS，重点在 workflow loop、持久状态、评测、权限、进程管理和恢复能力。面试中可以强调一句话：从“如何组织提示词”转向“如何构造一个可运行、可检查、可恢复的系统”。

**问: 递归自我改进是不是就是模型自己改自己的权重？**

不一定。直接改权重风险高、不可审计且不稳定；近中期更可行的是用当前模型智能去改进训练管线、数据筛选、工具/harness 和部署系统，再训练更强后继模型。原文也把这条路径视为更实际的 RSI。

**问: 为什么长程 agent 往往用文件系统而不是向量数据库做记忆？**

文件系统提供精确、可 grep、可 diff、可编辑的持久状态，且 bash/文件操作是 LLM 预训练强项；向量数据库适合语义检索但不适合保存和恢复代码 diff、实验日志和状态。实际系统常以文件为持久层，再在需要时检索摘要或片段进上下文。

**问: 子代理的输出为什么必须落盘，而不是直接放在聊天上下文？**

只在 chat context 中的输出不可持久、不可并发等待、难以失败重试和中断恢复；落盘为 logs/status/artifact 后，父代理才能像进程管理器一样检查日志、取消任务、合并结果，也便于审计与后续复用。

**问: coding agent 的工具集为什么围绕 bash、git、patch 这类低层接口？**

因为它们贴近真实开发者工作流，模型在预训练中见过大量 shell、git 和代码编辑模式，组合能力强；低层通用接口更容易标准化和泛化到不同仓库。代价是需要沙箱、权限管理和安全检查来防止破坏性操作。

## 适用场景

- 长时间自动研究/实验：自动提出假设、运行实验、分析日志、修改代码，直到验收目标满足。
- 编码 agent 产品：在真实仓库中搜索、修改、运行测试、提交变更，如 Claude Code、Codex、Cursor。
- 多假设并行搜索与消融实验：用子代理同时跑多组实验，父代理合并文件化结果并做决策。
- 自改进流水线：收集 agent 轨迹和失败样本，优化 harness 的工具、工作流与评估，部署更强的后继模型。

## 标签

`Harness Engineering` `Recursive Self-Improvement` `Agent Architecture` `LLM Deployment` `Coding Agents`
