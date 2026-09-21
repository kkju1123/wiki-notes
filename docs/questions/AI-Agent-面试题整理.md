---
title: AI Agent 面试题整理
url: wikibar://summary/questions/AI-Agent-面试题整理
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T11:56:16.389238+00:00'
---

# AI Agent 面试题整理

## 1. 实习中针对什么场景进行评测？评测目标是什么？

主要针对 Agent 核心链路进行评测，包括：

- 用户意图理解
- 知识检索
- Agent Planning
- Tool Calling
- 答案生成
- 异常场景
- 安全场景

评测目标不仅是判断最终答案是否正确，还要定位 Agent Pipeline 中具体哪个环节出现问题。

### 常见场景

- 正常场景：用户问题 → 检索 → Agent → 正确回答
- 边界场景：模糊问题、缺少参数、多轮上下文、超长输入
- 异常场景：检索不到文档、Tool Timeout、API Error
- 安全场景：Prompt Injection、越权工具调用、敏感信息泄露

### 常见指标

**Retrieval**

- Recall@K
- Precision@K
- Hit Rate@K
- MRR
- NDCG

**Generation**

- Correctness
- Faithfulness
- Relevance
- Completeness

**Agent**

- Tool Selection Accuracy
- Tool Call Success Rate
- Task Success Rate

**Safety**

- Unsafe Response Rate
- Attack Success Rate
- False Positive Rate
- False Negative Rate

---

## 2. 评测集如何构建？

评测集主要来自三类数据：

1. 真实线上 Case：保证数据分布接近真实用户
2. 人工构造 Case：覆盖核心业务、边界和异常场景
3. LLM 合成 Case：扩充长尾和复杂场景

例如：

```text
真实用户 Case    50%
人工 Case        20%
LLM 合成 Case    30%
```

一个完整 Case 不应该只有 Question 和 Answer，而应该保存完整 Ground Truth：

```json
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
```

这样可以分别进行 Retrieval、Generation、Agent 和 Safety 评测。

---

## 3. 如何保证评测集覆盖全部场景且可用？

首先建立 Evaluation Matrix。

| 维度 | 分类 |
|---|---|
| Intent | 查询 / 操作 / 澄清 / 拒答 |
| Knowledge | FAQ / 文档 / Case |
| Difficulty | Easy / Medium / Hard |
| Conversation | Single-turn / Multi-turn |
| Retrieval | Single-hop / Multi-hop / No-answer |
| Tool | No-tool / Single-tool / Multi-tool |
| Safety | Normal / Injection / PII / Unauthorized |

然后检查重要组合是否都有 Case，例如：

```text
退款 + 单轮 + 文档检索
退款 + 多轮 + 文档检索
退款 + 信息缺失
退款 + Tool Failure
退款 + Prompt Injection
```

此外需要：

- 数据去重
- Train / Eval 数据隔离
- 数据污染检查
- Ground Truth 验证
- 人工抽检
- 长尾 Case 补充

核心思想：

> Coverage 不能靠感觉判断，而应该通过 Evaluation Matrix 量化。

---

## 4. 人工如何进行问题归因？归因标准是什么？

按照 Agent Pipeline 进行错误归因：

```text
User Query
    ↓
Intent / Planning
    ↓
Retrieval
    ↓
Ranking
    ↓
Tool Selection
    ↓
Tool Execution
    ↓
Generation
    ↓
Safety
```

建立 Error Taxonomy：

```text
E1  Intent Error
E2  Retrieval Error
E3  Ranking Error
E4  Planning Error
E5  Tool Selection Error
E6  Tool Execution Error
E7  Hallucination
E8  Citation Error
E9  Safety Error
E10 Formatting Error
```

例如：

```text
正确文档没有进入 Top-K
→ Retrieval Error

正确文档已经进入 Context，但模型仍然回答错误
→ Generation / Reasoning Error

选择了错误 Tool
→ Tool Selection Error

Tool 正确，但是 API Timeout
→ Tool Execution Error
```

核心原则：

> 找到最早发生错误的 Pipeline Stage，而不是简单把所有错误归因给 LLM。

---

## 5. Case 错误率和回归验证结果如何？

通常统计：

```text
Case Pass Rate
Case Error Rate
Error Type Distribution
Regression Pass Rate
```

例如：

```text
Before:

Pass Rate = 82%

Retrieval Error = 9%
Generation Error = 5%
Tool Calling Error = 3%
Others = 1%

After Fix:

Pass Rate = 91%
Regression Pass Rate = 98%
```

实际面试时不要编造自己没有统计过的数据。

如果实习中没有完整统计，可以回答：

> 当时主要采用人工评测，没有建立非常完善的自动统计体系，这是项目中可以进一步优化的地方。如果重新设计，我会建立固定 Golden Dataset，并在每次版本更新后运行 Regression Evaluation。

---

## 6. 使用模型自动生成评测集时，如何确保引用正确的文档和案例信息？

核心原则：

> 不让模型依靠自己的知识生成 Ground Truth，而是 Grounded Generation。

流程：

```text
Document
    ↓
Chunk
    ↓
LLM
    ↓
Question + Answer + Evidence
```

要求模型输出：

```json
{
  "question": "...",
  "answer": "...",
  "document_id": "doc_123",
  "chunk_id": "chunk_45",
  "evidence": "...",
  "case_type": "..."
}
```

然后自动验证：

```text
document_id 是否存在
        ↓
chunk_id 是否属于该 Document
        ↓
Evidence 是否存在于 Chunk
        ↓
Answer 是否被 Evidence 支撑
```

还可以使用 Judge Model 判断：

```text
Is the answer fully supported by the evidence?
```

最终对部分数据进行人工抽检。

---

## 7. 如何组织文档和案例两类数据生成测试 Case？

维护两个数据池：

```text
Knowledge Documents
├── Product
├── FAQ
├── Policy
└── Manual
```

以及：

```text
Historical Cases
├── User Query
├── Agent Response
├── Expected Response
├── Error Type
└── User Feedback
```

生成方式：

```text
Document
    ↓
生成事实型 QA

Historical Case
    ↓
生成 Paraphrase / Boundary / Adversarial Case

Document + Historical Case
    ↓
LLM
    ↓
新的真实风格 Test Case
```

其中：

- Document 提供 Ground Truth
- Historical Case 提供真实用户表达和问题分布

---

## 8. 为什么实习项目中没有采用自动化评测？

可以回答：

> 当时项目处于比较早期的迭代阶段，业务标准仍然在变化，而且很多 Case 涉及复杂业务语义，Ground Truth 并不是简单的 Exact Match。因此最初使用人工评测建立评价标准。

自动化评测的前提是：

```text
Stable Rubric
+
High-quality Golden Dataset
```

否则只是把人工的不确定性转移给 Judge Model。

如果重新设计，可以采用：

```text
Deterministic Metrics
        +
LLM-as-a-Judge
        +
Human Evaluation
```

三者结合。

---

## 9. 如何为云端 Agent 的输出设计安全校验方案？

采用多层安全架构：

```text
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
```

### Input Guard

检测：

- Prompt Injection
- Jailbreak
- PII
- Malicious Content

### Tool Guard

采用：

- Tool Allowlist
- Parameter Validation
- RBAC
- User Confirmation
- Sandbox
- Rate Limit

例如：

```text
read_file
→ 自动执行

delete_file
→ 用户确认

transfer_money
→ 权限验证 + 用户确认
```

### Output Guard

检查：

- PII Leakage
- Secret Leakage
- Unsafe Content
- Citation Validity
- Hallucination
- Policy Violation

高风险操作采用：

```text
Rule Engine + Safety Model / LLM Judge
```

而不是完全依赖 LLM。

---

## 10. 了解哪些自动化评测论文或项目？

### RAGAS

常见指标：

```text
Faithfulness
Answer Relevancy
Context Precision
Context Recall
```

主要用于 RAG 系统评测。

### DeepEval

类似 LLM 应用中的 pytest，可以建立自动 Evaluation Pipeline，并接入 CI/CD。

### LangSmith

支持：

```text
Tracing
Dataset
Experiment
LLM-as-a-Judge
Regression Evaluation
```

### OpenAI Evals

用于建立 LLM / AI System Evaluation。

### AgentBench

用于评测 Agent 在不同任务环境中的能力。

### SWE-bench

主要用于 Coding Agent / Software Engineering Agent 评测。

### τ-bench

用于评测：

```text
Agent
+
Tool
+
User Interaction
```

核心区别：

> Benchmark 主要衡量模型或 Agent 的能力；实际项目中的 Evaluation Pipeline 更关注发现 Regression、定位问题以及保证线上系统稳定性。

---

## 11. 如何把人工问题归因改造成自动化流程？

首先保存完整 Agent Trace：

```text
Query
    ↓
Retrieved Documents
    ↓
Planning
    ↓
Tool Calls
    ↓
Tool Results
    ↓
Final Answer
```

然后建立自动归因 Pipeline：

```text
Retrieval Judge
      ↓
Planning Judge
      ↓
Tool Judge
      ↓
Answer Judge
      ↓
Safety Judge
```

例如：

```text
Expected Document 没进入 Top-K
→ Retrieval Error

Expected Document 已进入 Context
但答案错误
→ Generation Error

Tool 选择错误
→ Tool Selection Error

Tool 正确但 Timeout
→ Tool Execution Error
```

原则：

> 能使用 Deterministic Rule 判断的问题优先使用 Rule，复杂语义问题再使用 LLM Judge。

---

## 12. 是否做过模型本地部署？

如果没有真正做过生产级部署，可以回答：

> 我没有做过大规模生产级 LLM 本地部署，但对完整部署流程比较熟悉。我使用过 PyTorch / Hugging Face 模型，也做过 Docker / Kubernetes 服务部署。

基本流程：

```text
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
GPU Scheduling + Monitoring
```

重点关注：

- GPU Memory
- Throughput
- Latency
- Batch Size
- KV Cache
- Quantization
- Concurrency
- Autoscaling

---

# 面经 02：腾讯 AI Agent 开发岗

## 13. 请介绍 Agent Loop 的设计

基本 Agent Loop：

```text
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
```

伪代码：

```python
while step < max_steps:

    context = build_context()

    action = agent.plan(context)

    if action.type == "finish":
        return action.answer

    result = execute_tool(action)

    memory.append(result)

return fallback()
```

工程上还需要：

```text
max_steps
timeout
retry
tool permission
schema validation
fallback
logging
tracing
```

其中 `max_steps` 非常重要，可以防止 Agent 无限循环。

---

## 14. 项目中最困难的部分是什么？

可以回答：

> Agent 项目中最困难的部分并不是调用 LLM，而是保证整个 Pipeline 的可靠性。最终答案错误可能来自 Retrieval、Reranking、Planning、Tool Calling、Context Construction 或 Generation，如果只观察最终输出，很难定位真正的问题。

因此需要建立：

```text
Tracing
    ↓
Error Taxonomy
    ↓
Evaluation Dataset
    ↓
Error Attribution
    ↓
Fix
    ↓
Regression Test
```

最终目标是把 Agent 从：

```text
Demo 能运行
```

变成：

```text
可评测
+
可观测
+
可定位
+
可回归
+
可上线
```

---

## 15. 如何为线上内容安全设计检测指标？

内容安全首先可以抽象成 Classification：

```text
Input
    ↓
Safety Classifier
    ↓
Safe / Unsafe
```

核心指标：

```text
Precision
Recall
F1
FPR
FNR
```

### False Negative

危险内容没有被检测出来：

```text
Unsafe → Safe
```

意味着风险内容被放过。

### False Positive

正常内容被识别成危险内容：

```text
Safe → Unsafe
```

意味着正常用户被误拦截。

因此 Threshold 是一个 Trade-off：

```text
Threshold ↓
→ Recall ↑
→ False Positive ↑

Threshold ↑
→ Precision ↑
→ False Negative ↑
```

线上还可以监控：

```text
Unsafe Pass Rate
Over-block Rate
Attack Success Rate
User Complaint Rate
Latency
```

不同风险类别应该使用不同 Threshold，而不是所有内容使用统一阈值。

---

## 16. 内容安全评测集应该如何构建？

至少包含四类：

```text
Positive Set
→ 正常内容

Negative Set
→ 明确违规内容

Boundary Set
→ 灰色 / 模糊内容

Adversarial Set
→ 绕过攻击
```

特别需要构造 Hard Negative。

例如：

```text
包含敏感关键词
但实际上是：

新闻讨论
学术讨论
安全研究
引用内容
正常教育内容
```

否则模型可能学习成：

```text
出现敏感关键词
→ Block
```

导致线上严重误杀。

---

## 17. 文档切分采用什么策略？

优先采用 Structure-aware Chunking：

```text
Document
    ↓
Heading
    ↓
Section
    ↓
Paragraph
    ↓
Token-based Recursive Split
```

例如 Markdown：

```text
# Heading 1

## Heading 2

Paragraph

### Heading 3
```

先根据文档结构切分。

如果 Section 太长，再进行 Token-based Recursive Splitting。

同时设置一定：

```text
Chunk Overlap
```

防止语义在 Chunk Boundary 被截断。

对于特殊内容单独处理：

```text
Table
Code
FAQ
List
JSON
```

---

## 18. Chunk Size 的取值依据是什么？

Chunk Size 是一个 Trade-off。

### Chunk 太小

优点：

```text
Retrieval Precision ↑
```

缺点：

```text
Semantic Completeness ↓
Context Fragmentation ↑
```

### Chunk 太大

优点：

```text
Semantic Completeness ↑
```

缺点：

```text
Irrelevant Context ↑
Retrieval Precision ↓
Token Cost ↑
```

因此不能直接拍脑袋选择 512。

应该实验不同 Chunk Size：

```text
128
256
512
1024
```

保持以下变量一致：

```text
Embedding Model
Retriever
Top-K
Evaluation Dataset
```

然后进行对照实验。

---

## 19. 用什么指标判断切分策略是否合理？

首先评估 Retrieval：

```text
Hit Rate@K
Recall@K
Precision@K
MRR
NDCG
```

然后评估 Generation：

```text
Answer Correctness
Faithfulness
Answer Relevance
```

最后评估 End-to-End：

```text
Task Success Rate
```

例如：

```text
Chunk 256
Recall@5 = 91%
Answer Accuracy = 82%

Chunk 512
Recall@5 = 94%
Answer Accuracy = 89%

Chunk 1024
Recall@5 = 95%
Answer Accuracy = 84%
```

虽然 Chunk 1024 的 Retrieval Recall 更高，但可能因为 Context Noise 增加，导致最终 Answer Accuracy 下降。

因此最终应该优化：

```text
End-to-End Performance
```

而不是单独最大化 Retrieval Recall。

---

# 总结：Agent Evaluation 完整框架

可以记住一条主线：

```text
Business Scenario
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
```

对应中文：

```text
业务场景
→ 定义评测维度
→ 构建评测集
→ 执行评测
→ 问题归因
→ 修复问题
→ 回归测试
→ 线上监控
```

核心思想：

> 一个真正可上线的 Agent，不仅要“能回答问题”，还需要做到可评测、可观测、可定位、可回归和可控制。