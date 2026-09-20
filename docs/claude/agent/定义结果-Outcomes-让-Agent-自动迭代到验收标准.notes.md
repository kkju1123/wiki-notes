# 定义结果（Outcomes）：让 Agent 自动迭代到验收标准

*原文: [https://platform.claude.com/docs/en/managed-agents/define-outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes) · 来源: web · 生成时间: 2026-09-20T07:39:18.794672+00:00*

## 背景

LLM 代理处理复杂任务时，一次生成很难保证交付物满足真实需求；传统做法是用户反复看输出、提修改意见，既慢又不可复用。Anthropic 的托管代理因此引入 outcome 机制：用户先声明“完成长什么样”和评分标准，系统自动配置评估器驱动迭代。这借鉴了软件开发中的验收测试驱动（ATDD）和 CI 评审闭环，把“完成定义”从人脑搬到可执行规则。

## 痛点

没有明确 outcome 时，代理容易生成仅有表面完整性但不符合需求的产物，用户必须逐轮检查并反馈，沟通成本高。更麻烦的是，让主代理自己评估自己会产生自我偏袒，难以区分“真的做完”和“看起来做完了”。

## 解决办法

定义 outcome 时传入必需的 rubric，系统会在独立上下文中自动启动 grader。grader 按 rubric 对交付物逐项打分，返回哪些标准通过/失败以及解释，而不是只给总分。这份反馈会返回给主代理作为下一轮修订依据，形成自动迭代闭环。整个过程通过事件流暴露评估开始、进行中和结束事件，result 字段决定下一步：satisfied 就停，needs_revision 就继续，max_iterations_reached 则最后确认后停。

## 关键代码示例

```python
# 伪代码：展示 outcome 驱动自动迭代的核心流程
session = create_session(agent_id, environment_id)

rubric = '''
# 评分标准
- 正确实现所有 API 方法 (40%)
- 包含错误处理和重试 (30%)
- 类型标注完整 (20%)
- 文档清晰 (10%)
'''

# 告诉 agent 什么是 done，并让系统自动配置独立 grader
define_outcome(
    session_id=session.id,
    description='实现一个 REST API 客户端',
    rubric=rubric,
    max_iterations=5,
)

# 监听评估事件：grader 独立评估，agent 自动修订
for event in stream_events(session.id):
    if event.type == 'span.outcome_evaluation_end':
        print(event.result, event.feedback)
        if event.result in ('satisfied', 'failed', 'max_iterations_reached'):
            break
```

这段伪代码先定义 rubric，明确各标准的权重和通过条件；define_outcome 将“验收标准”绑定到 session，系统据此配置独立 grader。随后监听 span.outcome_evaluation_end 事件：每次该事件出现代表一次迭代评估完成，agent 已拿到反馈并可能修订；当 result 变成 satisfied/failed/max_iterations_reached 时退出。这比普通 user.message 多了一个自动评估-修订闭环。

## 关键流程

1. 设计有价值的 rubric：按可验证、可打分方式描述交付物需要满足的标准，并确保与任务描述一致。
2. 创建 outcome session：已有 agent 和 environment 后，发送 define_outcome 事件，内联传入 rubric 或通过 Files API 复用。
3. 代理自动执行与迭代：主 agent 朝 outcome 目标工作，产出到沙箱 /mnt/session/outputs/。
4. 独立 grader 评估：grader 在单独上下文按 rubric 评估当前交付物，并返回通过/失败项和解释。
5. 根据 result 处理循环：needs_revision 继续迭代；satisfied 进入 idle；failed 或 max_iterations_reached 进入收尾；interrupted 可开始新 outcome。
6. 检索交付物：轮询或监听事件确认结束后，通过 Files API 按 session_id scope 列出并下载输出文件。

## 关键点

- Outcome 的核心是“可验收目标 + 可执行评分标准”，rubric 是必需的；没有它，grader 无法客观判断 done，代理也无法自我纠偏。
- grader 使用独立 context window，避免主代理的推理过程和实现选择污染评估，这是保证评估可信的关键设计。
- 迭代次数和结果状态应该被监控：iteration 字段从 0 开始计数，result 的 satisfied/needs_revision/max_iterations_reached/failed/interrupted 决定后续是否还有自动回合。
- 用户仍可发送 user.message 对执行中的 outcome 进行干预，也可以 user.interrupt 暂停当前 outcome 并开启新目标；会话保留历史，便于连续治理。
- 交付物检索依赖沙箱输出目录和 Files API 的 scope_id 过滤，且需要 beta header；文件列表有短暂延迟，因此要有轮询或重试策略。

## 对比与权衡

- 相比传统多轮人工反馈，outcome 机制把验收标准程序化并自动迭代，能大幅减少人工往返，但前期设计 rubric 需要额外成本。
- 相比让主代理自己做 self-evaluation，独立 grader 避免了自我偏袒和上下文污染，但增加了每次迭代的额外模型调用开销。
- 相比固定流程或工作流 DAG，outcome 更适合实现路径不唯一、结果可由标准评分的开放式任务；但它的可控性弱于显式 DAG，可能出现迭代到 max_iterations 仍未达标的情况。
- 相比离线通用评估框架（如 RAGAS/OpenAI Evals），这里的 grader 是在线、按 session 自动嵌入的，适合驱动运行时迭代，而不是只做批量回归评测。

## 自测问题

**问: 为什么 grader 要放在独立 context window，而不是直接让主 agent 自评？**

独立上下文就像考试中的密封阅卷，grader 只看 rubric 和最终交付物，不会受到主代理中间推理、自我解释或实现偏好的影响，避免自我确认偏差。同时它也避免挤占主代理本来就很紧张的上下文空间。可以补充：独立评估也可以换用不同模型或更严格的评分策略，而不影响主代理。

**问: rubric 应该怎么写？如果 rubric 和任务描述冲突会怎样？**

rubric 最好把标准写成可验证、可观察的指标，比如‘包含重试逻辑且测试通过’而不是‘代码写得好’。如果 rubric 不适用于交付物，grader 会返回 failed，会话直接 idle，不会继续迭代；所以 description 和 rubric 必须对齐，否则任务无法自动收敛。

**问: max_iterations 起什么作用，应该怎么设置？**

它防止代理无限迭代烧 token 和时间。设置时要看任务复杂度和预算：简单任务可以 2-3 次，复杂生成/重构可能需要 5-10 次。达到上限后系统不会再评估，只给一个确认回合，通常需要人工介入决定接受还是重开 outcome。

**问: 如何判断一个 outcome 会话最终是成功还是失败？**

监听 span.outcome_evaluation_end 事件或轮询 session 里的 outcome_evaluations[].result。satisfied 表示达标；failed 表示 rubric 本身不可用；max_iterations_reached 表示做了多轮但未达标；interrupted 表示被用户暂停。成功与否不能只看会话 idle，还要看具体 result。

**问: 交付物为什么不在事件流里直接返回，而要经过 Files API？**

代理运行在沙箱中，产出文件可能较大或多种格式，事件流适合传控制信号和状态，不适合塞大文件。宿主通过 session_id 作 scope 列出 /mnt/session/outputs/ 下的文件再下载，既解耦又便于管理版本。需要注意该过滤能力在 beta header 下，文件列表可能有短暂延迟。

## 适用场景

- 代码生成/重构：给定仓库约束和 rubric，代理自动完成实现并按要求迭代到满足测试与风格标准。
- 技术文档或报告生成：rubric 检查结构、覆盖点、引用和可读性，代理自动修订直到达标。
- 数据分析与清洗：定义缺失值、异常值、输出格式等质量标准，让代理自行迭代直到数据产品合格。
- 需要多轮修改的设计方案：rubric 检查需求覆盖、风险分析和 trade-off 质量，减少人工反复评审。

## 标签

`Agents` `Outcomes` `Grader` `Rubric` `Claude API`
