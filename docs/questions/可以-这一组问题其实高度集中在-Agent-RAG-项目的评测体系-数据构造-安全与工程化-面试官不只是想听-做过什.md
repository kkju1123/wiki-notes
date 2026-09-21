---
title: 可以。这一组问题其实高度集中在 Agent / RAG 项目的评测体系、数据构造、安全与工程化。面试官不只是想听“做过什
url: wikibar://summary/questions/可以-这一组问题其实高度集中在-Agent-RAG-项目的评测体系-数据构造-安全与工程化-面试官不只是想听-做过什
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T11:53:07.384317+00:00'
---

可以。这一组问题其实高度集中在 Agent / RAG 项目的评测体系、数据构造、安全与工程化。面试官不只是想听“做过什么”，而是看你能不能把一个 Agent 从“Demo 能跑”做到“可评测、可定位、可上线”。

我按面试时可以直接说出口的方式给你整理，并顺便告诉你每道题背后在考什么。

一、实习中针对什么场景进行评测？评测目标是什么？

可以先把场景拆成几类，而不是泛泛说“评测 Agent 效果”。

我们主要针对 Agent 的核心链路做评测，包括用户意图理解、知识检索、工具调用、答案生成，以及异常场景和安全场景。评测目标不是只看最终答案对不对，而是希望定位整个 Agent Pipeline 中到底是哪一个环节出了问题。

比如：

正常场景：用户问题 → 检索 → Agent → 正确回答。

边界场景：模糊问题、缺少参数、多轮上下文、超长输入。

异常场景：检索不到文档、工具 timeout、API 返回异常。

安全场景：Prompt Injection、越权工具调用、敏感信息泄露、不安全内容。

指标一般分层：

Retrieval → Recall@K / Precision@K / MRR / Hit Rate

Generation → Correctness / Faithfulness / Relevance / Completeness

Agent → Tool Selection Accuracy / Tool Call Success Rate / Task Success Rate

Safety → Unsafe Response Rate / Attack Success Rate

这样回答会比单纯说“准确率”专业很多。

⸻

二、评测集如何构建？

推荐回答一个完整的数据来源链路：

我会把评测集分成三部分：真实线上 Case、人工构造 Case、模型生成 Case。真实 Case 保证数据分布接近实际用户；人工 Case 主要覆盖关键业务场景和边界条件；LLM 生成 Case 用来快速扩充长尾场景。

例如 1000 条：

真实用户 Case        50%
人工设计 Case        20%
LLM 合成 Case        30%

每个 Case 不应该只有：

{
  "question": "退款多久到账？",
  "answer": "3-5个工作日"
}

而应该包含：

{
  "question": "退款多久到账？",
  "expected_answer": "3-5个工作日",
  "source_document": "refund_policy_v3",
  "expected_chunks": ["chunk_21"],
  "expected_tool": null,
  "category": "refund",
  "difficulty": "normal",
  "risk_type": null
}

这样后面才能做 Retrieval / Generation / Agent 分层评测和错误归因。

⸻

三、如何保证评测集覆盖全部场景且可用？

这里面试官真正想听的是：Coverage 怎么量化？

不要回答“人工检查”。

我会先建立一个 Evaluation Matrix：

维度	分类
Intent	查询 / 操作 / 澄清 / 拒答
Knowledge	FAQ / 文档 / Case
Difficulty	Easy / Medium / Hard
Conversation	Single / Multi-turn
Retrieval	Single-hop / Multi-hop / No-answer
Tool	No-tool / Single-tool / Multi-tool
Safety	Normal / Injection / PII / Unauthorized

然后要求重要组合都有测试 Case。

例如：

退款 + 单轮 + 文档检索
退款 + 多轮 + 文档检索
退款 + 信息缺失
退款 + Tool Failure
退款 + Prompt Injection

除此之外还要做三件事：去重、污染检查、人工抽检。

训练数据和评测数据尽量隔离，LLM 生成数据还需要检查重复率、答案是否可由 source 支撑。

⸻

四、人工如何进行问题归因？归因标准是什么？

这是非常重要的一题。

不要只分成：

正确 / 错误。

应该按照 Agent Pipeline 定位。

User Query
   ↓
Intent / Planning
   ↓
Retrieval
   ↓
Tool Selection
   ↓
Tool Execution
   ↓
Generation
   ↓
Safety

错误 taxonomy 可以设计成：

E1 Intent Error
E2 Retrieval Error
E3 Ranking Error
E4 Planning Error
E5 Tool Selection Error
E6 Tool Execution Error
E7 Hallucination
E8 Citation Error
E9 Safety Error
E10 Formatting Error

比如用户问：

公司年假是多少？

系统检索到了错误文档。

即使最终答案错了，也应该归因：

Retrieval Error

而不是 Generation Error。

如果正确文档已经进入 Context，但模型还是答错：

Generation / Reasoning Error

这个区别非常关键。

⸻

五、Case 错误率和回归验证结果如何？

如果这是面试你自己的实习项目，千万不要编数字。

可以回答：

我们会统计整体 Case Pass Rate，同时按错误类型统计 Error Rate。修复问题之后不会只重新测试失败 Case，而是运行完整 Regression Set，确认修复没有影响已有能力。

例如你真实做过的话可以说：

Before:
Pass Rate = 82%
主要错误：
Retrieval 9%
Generation 5%
Tool Calling 3%
Others 1%
After:
Pass Rate = 91%
Regression Pass Rate = 98%

但没有真实数字，就说：

当时主要采用人工评测，没有建立非常完善的自动统计体系，这是我后来认为项目可以改进的地方。

这个回答反而可信。

⸻

六、LLM 自动生成评测集时，如何保证引用正确文档和案例？

关键思想：

不要让模型凭记忆生成 Case。

而是：

Document
   ↓
Chunk
   ↓
LLM
   ↓
Question + Answer + Evidence

要求模型输出：

{
  "question": "...",
  "answer": "...",
  "document_id": "doc_123",
  "chunk_id": "chunk_45",
  "evidence": "...",
  "case_type": "..."
}

然后做验证：

document_id 是否存在
        ↓
chunk_id 是否属于 document
        ↓
evidence 是否真的存在
        ↓
answer 是否被 evidence 支撑

最后可以再使用一个 Judge Model 判断：

Is answer fully supported by evidence?

但 Judge 只能作为过滤器之一，重要数据最好人工抽检。

⸻

七、如何组织“文档”和“案例”两类数据生成测试 Case？

这题可以回答成两个数据池：

Knowledge Documents
├── Product
├── FAQ
├── Policy
└── Manual
Historical Cases
├── User Query
├── Agent Response
├── Expected Response
├── Error Type
└── Feedback

然后生成测试数据时：

Document → factual QA
Historical Case → paraphrase / boundary / adversarial case
Document + Historical Case
        ↓
      LLM
        ↓
new realistic test case

文档负责提供 Ground Truth。

历史 Case 负责提供 真实用户表达方式和问题分布。

这个区分非常值得讲。

⸻

八、为什么实习项目没有采用自动化评测？

别说：

因为没时间。

可以说：

当时项目处于比较早期的迭代阶段，业务标准还在变化，而且很多 Case 涉及业务语义，Ground Truth 并不是简单 Exact Match，因此最初选择人工评测建立标准。自动化评测最大的前提其实是先定义稳定的 rubric 和高质量 Golden Dataset，否则只是把人工的不确定性转移给 Judge Model。

然后补一句：

如果现在重新设计，我会采用 Human Evaluation + LLM-as-a-Judge + deterministic metrics 的混合方案。

这个答案会好很多。

⸻

九、云端 Agent 输出如何设计安全校验？

不要只做一个敏感词过滤器。

推荐说 Input → Execution → Output 三层防御：

User Input
 ↓
Input Guard
 ↓
Agent / LLM
 ↓
Tool Permission Check
 ↓
Tool Execution
 ↓
Output Guard
 ↓
User

Input Guard：

Prompt Injection / Jailbreak / PII / malicious content

Tool Guard：

allowlist + parameter validation + RBAC + confirmation

例如：

read_file → 自动允许
delete_file → 用户确认
transfer_money → 用户确认 + 权限验证

Output Guard：

PII leakage / policy violation / secret leakage / citation validation

对于高风险行为，还可以增加：

LLM Judge + Rule Engine

而不是完全依赖另一个 LLM。

⸻

十、了解哪些自动化评测论文或项目？

面试 Agent/RAG，至少建议你知道这些：

RAGAS

主要评估：

Faithfulness
Answer Relevancy
Context Precision
Context Recall

DeepEval

比较像 LLM 应用的 pytest，可以把 evaluation 放进 CI/CD。

LangSmith

可以做 Dataset、Tracing、Experiment、LLM-as-a-Judge。

OpenAI Evals

用于构建模型/LLM 系统评测。

另外建议知道：

AgentBench：Agent 综合能力 benchmark。

SWE-bench：Coding Agent benchmark。

τ-bench / tau-bench：Tool-Agent-User interaction evaluation。

面试不用疯狂背论文，重点是知道：

Benchmark 是研究模型能力；项目里的 Evaluation Pipeline 是为了发现 regression 和定位系统问题。

⸻

十一、人工问题归因怎么自动化？

这是很好的 Agent 系统设计题。

可以把 Trace 保存下来：

Query
 ↓
Retrieved Documents
 ↓
Agent Reasoning / State
 ↓
Tool Calls
 ↓
Tool Results
 ↓
Final Answer

然后建立自动 Error Attribution Pipeline：

Retrieval Judge
      ↓
Planning Judge
      ↓
Tool Judge
      ↓
Answer Judge
      ↓
Safety Judge

例如：

expected doc 没进入 Top-K
→ Retrieval Error
expected doc 已进入 Context
但答案错误
→ Generation Error
Tool 选择错误
→ Tool Selection Error
Tool 正确但返回 timeout
→ Tool Execution Error

这里最好强调：

能用 deterministic rule 判断的不要用 LLM。

LLM Judge 主要负责语义问题。

⸻

十二、是否做过模型本地部署？

如果没真正做过，不要硬说做过。

你可以结合你自己的项目经验回答：

我没有做过大规模生产级 LLM 本地部署，但对完整流程比较熟悉。我使用过 PyTorch/Hugging Face 相关模型，也做过 Docker/Kubernetes 服务部署。我的理解是本地部署一般会经过模型选择、量化、推理框架、API Serving、GPU 调度和监控几个阶段。

然后讲：

Model
 ↓
Quantization
FP16 / BF16 / INT8 / INT4
 ↓
Inference Engine
vLLM / TensorRT-LLM / llama.cpp
 ↓
Serving
OpenAI-compatible API
 ↓
Docker
 ↓
Kubernetes
 ↓
GPU scheduling + monitoring

你这种回答比假装自己生产部署过 70B 强很多。

⸻

十三、Agent Loop 怎么设计？

这题腾讯 Agent 岗非常重要。

经典：

User
 ↓
Context
 ↓
Planner
 ↓
Reasoning
 ↓
Tool Selection
 ↓
Tool Execution
 ↓
Observation
 ↓
Reflection
 ↓
Continue / Finish

伪代码：

while step < max_steps:
    context = build_context()
    action = agent.plan(context)
    if action.type == "finish":
        return action.answer
    result = execute_tool(action)
    memory.append(result)
return fallback()

工程上必须有：

max_steps
timeout
retry
tool permission
schema validation
fallback
logging / tracing

否则 Agent 很容易死循环。

⸻

十四、项目最困难的部分是什么？

不要回答：

调 API 很困难。

可以结合你的 Agentic RAG 项目讲：

最困难的不是调用 LLM，而是保证整个 Agent Pipeline 的稳定性。因为最终答案错误可能来自 retrieval、reranking、tool calling、context construction 或 generation，单纯看最终输出很难定位。

然后介绍你怎么：

Tracing → Error Taxonomy → Evaluation Dataset → Regression Test

这能自然把前面所有问题串起来。

⸻

十五、线上元宝内容安全怎么设计指标？

安全检测本质可以看成一个分类问题：

Input
 ↓
Safety Classifier
 ↓
Safe / Unsafe

因此核心：

Precision / Recall / F1 / FPR / FNR

这里尤其需要理解两个错误：

False Negative：

危险内容没有检测出来。

通常安全风险很高。

False Positive：

正常内容被拦截。

用户体验下降。

所以阈值不是越严格越好，而是业务 trade-off。

线上还可以监控：

Unsafe Pass Rate
Over-block Rate
Attack Success Rate
Appeal / User Complaint Rate
Latency

面试时特别值得说：

不同风险类别应该使用不同 threshold，而不是全局统一 threshold。

⸻

十六、内容安全评测集怎么构建？

需要同时有：

Positive Set
正常内容
Negative Set
违规内容
Boundary Set
灰色/模糊内容
Adversarial Set
绕过攻击

尤其需要 Hard Negative。

例如一句话包含敏感词，但实际上是在讨论新闻、教育或者引用内容。

否则模型很容易学成：

看到某个关键词 → 拦截

评测结果看起来很好，线上误杀却非常严重。

⸻

十七、文档切分采用什么策略？

不要回答：

每 500 token 切一次。

成熟回答应该是：

我会优先采用结构感知切分，比如 Markdown Heading、Paragraph、Section，然后对过长 section 再进行 token-based recursive splitting，并设置一定 overlap。

例如：

Document
 ↓
Heading
 ↓
Section
 ↓
Paragraph
 ↓
Token Split

代码、表格、FAQ 等特殊结构最好单独处理。

⸻

十八、Chunk Size 根据什么确定？

Chunk Size 本质是 trade-off：

太小
→ semantic information 不完整
→ retrieval precision ↑
→ context completeness ↓
太大
→ 信息完整
→ irrelevant information ↑
→ retrieval precision ↓

所以不能拍脑袋决定 512。

应该实验：

128
256
512
1024 tokens

然后在固定 embedding / retriever / Top-K 的情况下比较最终指标。

⸻

十九、怎么判断 Chunking 是否合理？

这一题最好不要只回答 Recall@K。

因为 Chunking 最终服务的是整个 RAG。

第一层看 Retrieval：

Hit Rate@K
Recall@K
Precision@K
MRR
NDCG

第二层看 Generation：

Answer Correctness
Faithfulness
Answer Relevance

最终看：

End-to-End Task Success Rate

例如：

Chunk 256 → Recall@5 91%, Answer Acc 82%
Chunk 512 → Recall@5 94%, Answer Acc 89%
Chunk 1024 → Recall@5 95%, Answer Acc 84%

那 1024 虽然 Retrieval Recall 更高，却可能因为 Context Noise 导致最终回答变差。

所以真正应该优化的是：

End-to-End performance，而不是单独最大化 Retrieval Recall。

⸻

你可以把这 19 道题串成一套统一框架

面试的时候脑子里记住这一条链就行：

Data → Evaluation → Attribution → Regression → Safety → Production

也就是：

业务场景
  ↓
Evaluation Matrix
  ↓
Golden Dataset
  ↓
Agent Evaluation
  ↓
Error Attribution
  ↓
Fix
  ↓
Regression Test
  ↓
Online Monitoring

这其实就是这些面试题背后的同一个问题：

“你有没有能力建立一个可靠的 Agent，而不仅仅是调用 LLM API？”

对于你准备的 Agentic RAG + FastAPI + Qdrant + K8s 这类项目，这套回答尤其适合。你最好把这些问题全部往自己的项目上套，而不是背标准答案，这样追问的时候才接得住。