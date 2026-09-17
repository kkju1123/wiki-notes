# Graph Engineering：从串行 Agent 到并行 Agent 舰队的 14 步路线图

*原文: [https://x.com/angeldot_/status/2081061068516798931](https://x.com/angeldot_/status/2081061068516798931) · 来源: twitter · 生成时间: 2026-09-17T02:40:33.618686+00:00*

## 背景

多步 LLM Agent 通常被写成线性脚本：步骤 1 → 2 → 3，但其中很多步骤并不依赖前一步结果。这种串行执行不仅慢，还会让主上下文窗口不断累积中间结果，导致上下文膨胀和成本上升。Graph Engineering 把工作流显式建模为有向图，借助 Claude Code 的动态工作流与子代理，把调度层从模型对话转移到确定性代码中。

## 痛点

线性 agent 会把所有步骤排成一条队列，无依赖的步骤也必须等待；一个节点失败会阻断后续所有工作，且中间结果全塞进主上下文，窗口很容易被撑爆并产生不必要的模型 token 消耗。

## 解决办法

把工作流画成图：节点是接受显式输入、产出结构化输出的工作单元，边是“A 的输出喂给 B 的输入”的数据契约，而不是“A 然后 B”。给每个 agent() 调用加 JSON schema，让 tool call 层校验输出并自动重试，保证下游可编程消费。对互不读取对方的 N 个节点，用 parallel() 扇出并发执行；parallel() 本身是 barrier，等所有子代理结束后才返回，单个失败解析为 null，再用 filter(Boolean) 丢弃。归并、去重、扁平化等边操作直接用 JS/TS 代码完成，不再调用模型，因此编排层几乎不消耗 token。整体类似 CI 流水线的 DAG：LLM 只负责有判断价值的节点，代码负责管道与并发控制。

## 关键代码示例

```javascript
// 每个 thunk 启动一个子代理，按 schema 返回结构化结果
const sources = ["docs/api.md", "docs/auth.md", "docs/billing.md"];

const results = await parallel(
  sources.map((file) => async () => {
    const out = await agent({
      prompt: `Summarize ${file}`,
      schema: { summary: "string", topics: ["string"] },
    });
    return { file, ...out };
  })
);

const clean = results.filter(Boolean); // 失败 thunk 变 null，不拖垮整批

const merged = clean.flatMap((r) =>
  r.topics.map((topic) => ({ topic, file: r.file }))
);
```

sources.map 为每个文件生成一个 thunk，并行执行子代理；schema 约束每个子代理返回 {summary, topics}，否则 tool call 层会重试。parallel() 等待所有 thunk 完成后返回；失败的 thunk 会解析为 null，filter(Boolean) 丢弃异常结果。最后的 flatMap 只做数据归并，是边操作，不消耗模型 token。

## 关键流程

1. 01：把每个工作单元画成节点，把数据流动画成边；只保留真正传递数据的边。
2. 02：重画线性链式流程，删除不传输数据的“然后”伪依赖，让无依赖节点变宽。
3. 03：给每个节点定义输入/输出 schema，让输出可被下游消费和验证。
4. 04：用 JS/TS 代码实现边的归并、过滤和去重，而不是用 agent 做管道。
5. 05：对 N 个独立节点调用 parallel() 扇出，并发执行子代理。
6. 06：在 barrier 处等待全部结果，过滤失败项后进入 fan-in 汇总节点。

## 关键点

- 图的节点是工作单元，边是数据依赖而不是时间顺序；只有后一步读取前一步输出时，才需要串行等待。
- 很多线性链中隐藏着无数据流动的伪依赖，删除它们通常比购买任何工具都更能降低等待时间。
- 用 JSON schema 给每个子代理输出定型，能让 tool call 层自动校验和重试，避免下游解析不可靠文本。
- parallel() 的 barrier 语义保证 fan-out 后的下一阶段看到完整结果集；失败变为 null 的容错设计避免单点异常拖垮整批。
- 调度和边操作是确定性代码，不占用模型上下文，LLM 只应负责需要判断的节点。
- 把 agent 用于管道/合并是常见的浪费；flatMap、Set、filter 等代码级边操作比 agent 更快、更便宜、更确定。

## 对比与权衡

- 相比线性链式编排，图编排能让无依赖节点并行执行、减少整体延迟和上下文压力，但需要额外设计依赖关系和节点合同，小任务会显得过度设计。
- 相比让一个主 agent 顺序处理多个来源并在同一上下文中合并，parallel() 扇出使每个子代理只加载自己的上下文，主上下文只接收最终结果，可扩展性更好，但需要把合并逻辑拆成代码或独立节点。
- 相比 LangGraph 这类专用图编排框架，Claude Code 动态工作流直接用 JavaScript 表达图更轻量、更贴近代码库，但在可视化、持久化状态、失败恢复和复杂图模式上通常不如专用框架成熟。

## 面试可能会问

**问: 怎么判断两个步骤能不能并行？**

看后一步是否消费前一步的输出。如果只是脚本里的先后顺序但没有数据流动，就是伪依赖，可以拆成独立节点并行；真正的边必须是一个变量从 A 的返回进入 B 的 prompt。

**问: 为什么要求每个节点返回 schema，而不是自然语言？**

schema 让验证发生在 tool call 层，不匹配时自动重试，并且下游节点可以像调用函数一样消费结构化数据；自然语言需要额外解析，失败不可控。

**问: parallel() 中某个子代理失败了会怎样？**

在常见封装里，抛异常的 thunk 会解析为 null 而不是让整批失败，因此通常会在 parallel() 之后 filter(Boolean) 丢弃失败项，允许部分失败继续收敛，适合容错场景。

**问: 为什么编排层不消耗 token？**

因为 fan-out、fan-in、过滤、去重等逻辑是 JavaScript 控制流，不是在和大模型对话；只有节点内部需要模型判断才计费，图的连接和归并是确定性代码。

**问: 图工程什么时候不必要？**

如果步骤之间本来就有强数据依赖、任务规模很小或只有一两个节点，强行并行只会增加复杂度；先画依赖图并删除伪边，通常是投入产出比最高的一步。

## 适用场景

- 需要从多个数据源或文件并行收集信息，再汇总成单一结论或报告。
- 批量审查、测试、翻译或摘要等互不依赖的任务，需要并发执行以降低延迟。
- 多步 Agent 项目中主上下文频繁膨胀、成本过高时，用来识别和删除无意义的串行等待。
- 需要让输出可验证、可重试、可被下游程序稳定消费的生产级 Agent 工作流。

## 标签

`Graph Engineering` `Agent Orchestration` `Parallelism` `Claude Code` `LLM Workflow`
