---
title: 评测 Agent
url: wikibar://summary/questions/评测-Agent
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-22T03:09:53.246702+00:00'
---

# 评测 Agent

这篇文章讲的是一个你做 **Agent 项目/面试非常容易被问到的问题：**

> **“你做了一个 Agent，怎么证明它做得好？”**

文章的核心其实只有一句话：

**评测 Agent ≠ 看最终答案对不对，而是看它能不能正确、稳定、安全、低成本地把整个任务完成。** ([Aliyun Developer Community][1])

[你发的原文：面试官问：Agent 怎么评测？](https://developer.aliyun.com/article/1739858)

我用一个非常具体的例子把整篇文章串起来。

---

## 1. 为什么 Agent 不能只看 Accuracy？

假设你做了一个**天气 Agent**。

用户：

> 今天杭州天气怎么样？

普通 LLM 的流程很简单：

```text
用户问题
   ↓
LLM
   ↓
答案
```

所以你可能只需要判断：

```text
回答正确了吗？
```

但 Agent 不一样：

```text
用户：杭州今天天气怎么样？
        ↓
     Agent
        ↓
判断：需要实时天气
        ↓
选择 weather_tool
        ↓
生成参数
city="杭州"
date="today"
        ↓
调用天气 API
        ↓
得到 28°C，小雨
        ↓
LLM 根据工具结果组织答案
        ↓
“杭州今天 28°C，有小雨……”
```

这里任何一步都可能出问题。文章特别强调，Agent 是**多步骤执行系统**，因此最终答案只是评测的一部分。([Aliyun Developer Community][1])

例如最终回答碰巧是：

```text
杭州今天 28°C，小雨。
```

看起来完全正确。

但实际上 Agent **根本没调用天气 API，是模型猜的。**

那这个 Agent 算好吗？

当然不能算。

因为明天它可能继续猜：

```text
杭州今天 26°C，晴。
```

结果实际上暴雨。

所以 Agent evaluation 的思想是：

```text
不能只看 WHAT
还要看 HOW
```

也就是：

```text
最终结果 + 执行过程
```

---

# 2. Agent 到底评什么？

文章列了很多指标，你面试的时候不用死背。我建议你记成 **5 大类**：

```text
Agent Evaluation

1. 任务完成了吗？
2. 结果对吗？
3. 过程对吗？
4. 稳定 / 快 / 便宜吗？
5. 安全吗？
```

文章进一步拆成任务完成率、结果质量、过程质量、工具调用、检索质量、性能成本、稳定性和安全权限等维度。([Aliyun Developer Community][1])

我们继续拿天气 Agent 来理解。

### ① Task Success：任务完成了吗？

用户：

```text
帮我查杭州今天的天气
```

Agent 最终真的拿到了天气：

```text
success
```

如果 API 报错然后 Agent 直接挂掉：

```text
failed
```

那么可以统计：

```text
Task Success Rate
= 成功任务数 / 总任务数
```

例如：

```text
测试 1000 次

成功 930
失败 70

Task Success Rate = 93%
```

这个指标非常重要，因为 Agent 的目标通常不是“说句话”，而是**完成任务**。([Aliyun Developer Community][1])

---

# 3. Result Quality：最后答案好吗？

例如工具返回：

```json
{
  "temperature": 28,
  "weather": "rain",
  "humidity": 80
}
```

Agent 回答：

```text
杭州今天 28°C，有雨，湿度 80%。
```

很好。

但如果回答：

```text
杭州今天 38°C，大晴天。
```

明显有问题。

这里可以评：

```text
Accuracy
Completeness
Relevance
Hallucination
```

也就是：

```text
正确吗？
完整吗？
回答用户真正的问题了吗？
有没有瞎编？
```

所以 **accuracy 没有消失，只是它不再是全部。**

---

# 4. Tool Calling：工具调用对不对？

这个是 **Agent 面试重点中的重点**。

Agent 最大的特点之一就是：

```text
LLM
 +
Tools
```

所以必须评估：

```text
该不该调用工具？
        ↓
调用了正确工具吗？
        ↓
参数对吗？
        ↓
工具返回结果使用对了吗？
```

例如用户：

```text
杭州今天天气怎么样？
```

Agent 有两个工具：

```python
weather_tool(city)

route_tool(start, end)
```

正确：

```text
weather_tool("杭州")
```

错误可能有很多种：

```text
❌ 没调用工具，直接猜

❌ 调用 route_tool

❌ weather_tool("上海")

❌ weather_tool 参数格式错误

❌ API 返回 28°C，Agent 却回答 38°C
```

所以 Tool Evaluation 可以拆成：

```text
Tool Selection Accuracy

Parameter Accuracy

Tool Execution Success Rate

Tool Result Utilization
```

这部分是你做 Agent 项目时非常值得强调的。文章也指出，很多 Agent 错误其实不是“语言能力差”，而是工具链路出了问题。([Aliyun Developer Community][1])

---

# 5. Process Quality：Agent 有没有瞎折腾？

假设查询天气其实只需要：

```text
weather_tool
```

一次就够。

Agent A：

```text
LLM
 ↓
weather_tool
 ↓
answer
```

Agent B：

```text
LLM
 ↓
Google Search
 ↓
LLM
 ↓
weather_tool
 ↓
LLM
 ↓
weather_tool
 ↓
Google Search
 ↓
LLM
 ↓
answer
```

两个人最后都回答正确。

但是显然：

```text
Agent A > Agent B
```

不是因为答案更正确，而是 A：

```text
步骤少
延迟低
token 少
API 调用少
成本低
```

文章因此强调，要检查重复调用、多余步骤、循环推理、不必要重试，以及失败后的降级是否合理。([Aliyun Developer Community][1])

你可以把它理解成：

> **Agent 不仅要“做对”，还要“聪明地做对”。**

---

# 6. Trace / Span 是什么？

这是文章里一个很重要的工程概念。

假设一次 Agent 请求：

```text
用户
 ↓
LLM
 ↓
Search
 ↓
LLM
 ↓
Weather API
 ↓
LLM
 ↓
答案
```

我们把**整个请求生命周期**叫：

```text
Trace
```

里面每一步叫：

```text
Span
```

所以：

```text
Trace #10086

├── Span 1: LLM
│      800 ms
│
├── Span 2: Search
│      1200 ms
│
├── Span 3: LLM
│      600 ms
│
├── Span 4: Weather API
│      300 ms
│
└── Span 5: LLM
       900 ms
```

这样如果用户说：

> 怎么这么慢？

你就能查：

```text
到底哪里慢？
```

可能发现：

```text
LLM       0.8s
Search    15s   ← 问题
Weather   0.3s
LLM       0.9s
```

那就知道不是 LLM 慢，是 Search 慢。

所以：

```text
Trace = 一次完整 Agent 任务

Span = 任务中的一个步骤
```

文章把这一点概括得很好：

**没有可观测性，就很难真正评测 Agent。** ([Aliyun Developer Community][1])

---

# 7. RAG Agent 又怎么评？

这个跟你自己的 Agentic RAG 项目就很相关。

假设：

```text
用户问题
 ↓
Vector DB
 ↓
Top-K Documents
 ↓
LLM
 ↓
Answer
```

最终答案错了。

你不能马上说：

```text
LLM 不行。
```

因为可能：

```text
Retriever
   ↓
找错文档
   ↓
LLM拿到错误context
   ↓
当然答错
```

所以 RAG 至少拆成：

```text
Retrieval Evaluation
        +
Generation Evaluation
```

前者看：

```text
正确文档召回来了吗？
文档相关吗？
证据可靠吗？
```

后者看：

```text
LLM有没有根据context回答？
引用对吗？
有没有脱离证据瞎编？
```

也就是：

```text
Question
   ↓
Retriever
   ↓
正确 evidence？   ← 评一次
   ↓
LLM
   ↓
正确 answer？     ← 再评一次
```

这比：

```text
Answer 对不对？
```

高级很多。文章也明确建议 RAG Agent 单独评检索召回、证据相关性、引用正确性和无答案时的拒答能力。([Aliyun Developer Community][1])

---

# 8. Cost / Latency：答对了也可能是垃圾 Agent

假设：

```text
Agent A
成功率：95%
平均耗时：3 秒
成本：$0.01 / task
```

另一个：

```text
Agent B
成功率：96%
平均耗时：40 秒
成本：$0.30 / task
```

B 虽然成功率高了 1%，但是：

```text
慢 13 倍
贵 30 倍
```

生产环境里你肯定需要认真权衡。

所以要记录：

```text
Latency
Token Usage
LLM Calls
Tool Calls
API Cost
Cost / Task
```

这就是为什么 Agent evaluation 本质上已经不是单纯的“模型评测”，而更接近：

```text
AI System Evaluation
```

文章也强调了单次任务耗时、首 token 延迟、模型/工具调用次数、token 消耗、接口费用和平均任务成本。([Aliyun Developer Community][1])

---

# 9. Stability：正常情况会做还不够

比如：

```text
weather API 正常
        ↓
Agent 正常
```

没什么了不起。

真正的问题是：

```text
Weather API timeout
```

Agent 怎么办？

差的 Agent：

```text
Exception
程序崩了
```

或者更糟：

```text
API失败
 ↓
LLM自己编天气
```

好的系统可能：

```text
API失败
 ↓
retry
 ↓
仍失败
 ↓
fallback
 ↓
明确告诉用户暂时无法获取实时天气
```

因此还要测：

```text
Timeout Rate
Error Rate
Retry Success Rate
Fallback Success Rate
```

文章强调生产 Agent 不只是要在正常情况下跑通，还要看异常情况下能不能“兜住”。([Aliyun Developer Community][1])

---

# 10. Safety：Agent 比普通 Chatbot 更危险

为什么？

因为 Chatbot 通常只是：

```text
说话
```

Agent 可以：

```text
查数据库
发邮件
删文件
执行代码
调用支付 API
修改数据
```

所以例如：

```text
用户：
删除生产数据库所有数据。
```

Agent 不能：

```text
好的！

DELETE FROM users;
```

而应该有：

```text
Permission Check
Risk Check
Human Approval
```

因此测试：

```text
有没有越权？
有没有泄露数据？
危险操作有没有确认？
Prompt Injection 能不能突破限制？
```

文章把权限控制、敏感信息、高风险操作确认和危险指令识别都放进了 Agent 的核心评测范围。([Aliyun Developer Community][1])

---

# 11. Offline Evaluation 和 Online Evaluation

这两个概念非常重要。

### Offline = 上线前考试

自己准备测试集：

```text
Test Case 1
查天气

Test Case 2
查路线

Test Case 3
API timeout

Test Case 4
错误参数

Test Case 5
权限攻击

...
```

然后每次改：

```text
Prompt
Model
Tools
RAG
Agent Workflow
```

都重新跑。

文章将离线测试进一步分成：

```text
Smoke Test
Regression Test
Behavior Test
Safety Test
```

其中 Smoke Test 是“核心功能有没有直接坏掉”，Regression Test 是“以前修过的问题有没有重新出现”。([Aliyun Developer Community][1])

### Online = 上线后真实考试

上线之后看真实用户：

```text
Success Rate
Latency
Cost
Error Rate
Tool Failure
用户重试率
用户点踩
用户反馈
```

因为测试集永远覆盖不了真实用户的所有奇怪输入。([Aliyun Developer Community][1])

---

# 12. 最重要的是 Evaluation Loop

文章最后真正想表达的，其实是这个：

```text
开发 Agent
   ↓
Offline Eval
   ↓
上线
   ↓
Online Eval
   ↓
发现失败 Case
   ↓
分析 Trace
   ↓
找到原因
   ↓
修 Prompt / Tool / RAG / Workflow
   ↓
失败 Case 加入 Regression Dataset
   ↓
重新 Offline Eval
   ↓
再次上线
```

这就叫：

```text
Evaluation Loop
```

或者：

```text
持续评测闭环
```

非常像传统软件：

```text
发现 Bug
 ↓
修 Bug
 ↓
写 Regression Test
 ↓
以后不能再犯
```

只不过 Agent 的“Bug”除了代码 Bug，还包括：

```text
LLM判断错误
Tool选择错误
参数错误
RAG错误
幻觉
多余调用
权限错误
```

这也是文章所谓“失败样本回流”的核心。([Aliyun Developer Community][1])

---

# 13. LLM-as-a-Judge 又是什么？

有些东西程序很好判断：

```text
SQL执行成功？ → True/False
API调用成功？ → True/False
测试通过？ → True/False
```

但是：

```text
回答是否清晰？
回答是否完整？
是否真正解决用户问题？
```

很难写规则。

于是可以让另一个 LLM 当“老师”：

```text
Agent Answer
     ↓
Judge LLM
     ↓
Correctness: 4/5
Completeness: 5/5
Relevance: 4/5
```

这就是：

```text
LLM-as-a-Judge
```

但文章提醒：**不能什么都交给 Judge LLM。**

优先：

```text
能用确定性规则 → Rule

能用业务结果验证 → Business Metric

不好自动判断 → LLM-as-Judge

特别重要 → Human Review
```

因为 LLM 自己也是概率模型，也可能评错。([Aliyun Developer Community][1])

---

# 最后把整篇文章压缩成一张脑图

你以后看到：

> **Agent 怎么评测？**

脑子里直接出现：

```text
                    Agent Evaluation
                           │
          ┌────────────────┴────────────────┐
          │                                 │
       Result                            Process
          │                                 │
   Task Success                     Tool Calling
   Correctness                      Planning
   Completeness                     Retrieval
   Hallucination                    Retry / Fallback
          │                                 │
          └────────────────┬────────────────┘
                           │
                    Engineering
                           │
              Latency / Cost / Stability
                           │
                        Safety
                           │
             Permission / Data / Approval
                           │
                           ↓
                 Offline Evaluation
                           ↓
                        Deploy
                           ↓
                  Online Evaluation
                           ↓
                    Failed Cases
                           ↓
                  Regression Dataset
                           ↓
                    Offline Eval ...
```

**你真正需要记住的面试逻辑就是：**

```text
Agent不是“回答问题的模型”
        ↓
而是“执行任务的系统”
        ↓
所以不能只评最终答案
        ↓
还要评执行过程
        ↓
Task + Result + Tool + RAG + Process
+ Cost + Latency + Stability + Safety
        ↓
Offline Eval + Online Eval
        ↓
失败样本回流
        ↓
持续评测闭环
```

你最近在准备 **Agent 后端/Agentic RAG 方向的面试**，这一篇其实值得收进你的面试笔记。尤其要会把 **trace/span、tool evaluation、RAG evaluation、offline/online evaluation、LLM-as-Judge、regression dataset** 这几个词用自己的话讲出来。

[1]: https://developer.aliyun.com/article/1739858 "面试官问：Agent 怎么评测？-阿里云开发者社区"