---
title: Embeddings
url: wikibar://summary/summary/Embeddings
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T07:52:38.187036+00:00'
---

# Embeddings
这页 **Embeddings（嵌入）** 对你做 RAG 特别关键。你之前项目里的 **Qdrant 向量检索**，底层最核心的东西就是 Embedding。

先记一句：

> **Embedding = 把一段文字的“语义”转换成一串数字组成的向量，让计算机可以计算两段文字在意思上有多像。**

Anthropic 官方目前**没有自己的 embedding 模型**，这篇 Claude 文档主要推荐并演示 Voyage AI 的 embedding 模型。([Claude Platform][1])

## 1. Embedding 到底是什么？

例如有三句话：

```text
A: "I love dogs."
B: "I really like puppies."
C: "How do I install Docker?"
```

人一眼就知道：

```text
A 和 B → 意思很接近

A 和 C → 完全没关系
```

但计算机看到的是字符串。

Embedding Model 会把它们转换成向量：

```text
"I love dogs."
        ↓
Embedding Model
        ↓
[0.12, -0.37, 0.84, 0.19, ...]


"I really like puppies."
        ↓
Embedding Model
        ↓
[0.14, -0.35, 0.81, 0.21, ...]


"How do I install Docker?"
        ↓
Embedding Model
        ↓
[-0.72, 0.18, -0.13, 0.91, ...]
```

真正的向量通常不是 4 个数字，而可能是：

```text
256维
512维
1024维
2048维
```

例如文档当前推荐的 Voyage 4 系列默认是 **1024 维**，也支持 256、512、2048 维。([Claude Platform][1])

---

## 2. 为什么转成数字以后就有用了？

因为数字可以算距离。

例如：

```text
Embedding("dog")
          ↓
       vector A

Embedding("puppy")
          ↓
       vector B

计算 similarity(A, B)
          ↓
          高
```

而：

```text
Embedding("dog")
          ↓
       vector A

Embedding("database")
          ↓
       vector C

similarity(A, C)
          ↓
          低
```

于是：

```text
语义相近
↓
向量空间里比较接近

语义不同
↓
向量空间里比较远
```

这就是 **Semantic Search（语义搜索）** 的基础。

---

# 3. 和普通关键词搜索有什么区别？

假设你的文档里写：

```text
The puppy needs food.
```

用户搜索：

```text
How should I feed my dog?
```

传统关键词搜索可能发现：

```text
query: dog
document: puppy

dog != puppy
```

匹配不强。

Embedding：

```text
dog
 ↓
语义
 ↓
puppy

很接近
```

所以即使**一个词都不一样**，仍然可以找到。

这也是为什么 RAG 通常需要 embedding。

---

# 4. 最基本的代码

官方当前示例使用 Voyage：

```bash
pip install -U voyageai
```

然后：

```python
import voyageai

vo = voyageai.Client()

texts = [
    "Sample text 1",
    "Sample text 2"
]

result = vo.embed(
    texts,
    model="voyage-4",
    input_type="document"
)

print(result.embeddings[0])
print(result.embeddings[1])
```

API Key 可以放环境变量：

```bash
export VOYAGE_API_KEY="<your secret key>"
```

`voyageai.Client()` 会自动读取它。([Claude Platform][1])

结果大概：

```python
[
    -0.0131,
     0.0198,
     ...
]
```

这：

```python
result.embeddings[0]
```

就是第一段文字的 **embedding vector**。

---

# 5. RAG 里到底怎么用？

这是最重要的。

假设你有一本 PDF：

```text
Machine Learning Book
```

先切块：

```text
PDF

↓ chunking

Chunk 1:
"Gradient descent is..."

Chunk 2:
"Transformer attention..."

Chunk 3:
"Convolutional networks..."

Chunk 4:
"Reinforcement learning..."
```

然后每一个 chunk 做 embedding：

```text
Chunk 1 → [0.12, 0.73, ...]
Chunk 2 → [-0.22, 0.19, ...]
Chunk 3 → [0.88, -0.31, ...]
Chunk 4 → [0.14, 0.91, ...]
```

然后存进你熟悉的：

```text
Qdrant
```

或者：

```text
Pinecone
Milvus
Weaviate
pgvector
```

这种 Vector Database。

整个 ingestion：

```text
PDF
 ↓
解析文本
 ↓
Chunking
 ↓
Embedding Model
 ↓
Vectors
 ↓
Vector Database
```

---

# 6. 用户提问的时候再 Embedding 一次

用户：

```text
How does attention work?
```

先：

```text
"How does attention work?"
          ↓
Embedding Model
          ↓
[0.21, -0.17, 0.72, ...]
```

然后拿这个 query vector 去 Qdrant：

```text
Query Vector
     ↓
Vector Search
     ↓
和数据库里的 vectors 比较
     ↓
找最相似的 Top K
```

可能得到：

```text
Top 1:
"Transformer attention..."

Top 2:
"Self-attention computes..."

Top 3:
"Multi-head attention..."
```

然后才轮到 Claude：

```text
Question
+
Retrieved Chunks
       ↓
Claude
       ↓
Answer
```

完整 RAG：

```text
                  离线 / ingestion

Documents
    ↓
Chunking
    ↓
Embedding
    ↓
Vector DB


                  在线 / retrieval

User Question
    ↓
Embedding
    ↓
Vector Search
    ↓
Top-K Chunks
    ↓
Question + Chunks
    ↓
Claude
    ↓
Answer
```

**Embedding 负责“找资料”；Claude 负责“读资料并回答”。**

---

# 7. `input_type="document"` 和 `"query"`

Voyage 这里有一个非常重要的设计。

存文档时：

```python
doc_embds = vo.embed(
    documents,
    model="voyage-4",
    input_type="document"
).embeddings
```

用户搜索时：

```python
query_embd = vo.embed(
    [query],
    model="voyage-4",
    input_type="query"
).embeddings[0]
```

也就是：

```text
知识库里的 Chunk
→ input_type="document"

用户的问题
→ input_type="query"
```

官方明确建议检索/RAG 场景不要省略 `input_type`，因为模型会针对“查询”和“待检索文档”生成更适合检索的表示。([Claude Platform][1])

---

# 8. 怎么算两个 Embedding 像不像？

最常见：

**Cosine Similarity（余弦相似度）**

$$
\operatorname{sim}(A,B)=\frac{A\cdot B}{\|A\|\|B\|}
$$

人话：

> 不太关心向量有多长，主要看两个向量“方向像不像”。

例如：

```text
dog ↔ puppy

cosine similarity = 0.91
```

很像。

```text
dog ↔ Kubernetes

cosine similarity = 0.12
```

不太像。

Voyage 的 embedding 已经归一化成单位长度，因此官方说明对这些向量：

```text
dot product = cosine similarity
```

而点积计算更简单。([Claude Platform][1])

所以官方示例直接：

```python
import numpy as np

similarities = np.dot(
    doc_embds,
    query_embd
)

retrieved_id = np.argmax(similarities)

print(documents[retrieved_id])
```

这段非常值得理解。

假设：

```python
similarities = [
    0.12,
    0.35,
    0.91,
    0.08
]
```

那么：

```python
np.argmax(similarities)
```

得到：

```text
2
```

说明：

```text
documents[2]
```

和用户问题最相关。

---

# 9. Vector Database 在干嘛？

你可能会想：

> 那我直接 NumPy 算不就好了？

如果只有：

```text
100 个 vectors
```

当然可以。

但是如果：

```text
100,000,000 个 vectors
```

每次：

```text
query
 ↓
和一亿个 vector 全部比较
```

就太慢了。

Vector DB 会建立专门的索引，例如 ANN（Approximate Nearest Neighbor）结构：

```text
Query Vector
     ↓
Vector Index
     ↓
快速找到附近区域
     ↓
Top K Similar Vectors
```

所以你项目里的：

```text
Qdrant
```

核心职责之一就是：

> **存 embedding + 高效做相似向量检索。**

---

# 10. Embedding Model 和 LLM 不是一个东西

这个特别容易混。

Claude：

```text
文字
 ↓
Claude
 ↓
文字
```

Embedding Model：

```text
文字
 ↓
Embedding Model
 ↓
数字向量
```

例如：

```python
Claude("What is RAG?")
```

得到：

```text
RAG is a technique that...
```

而：

```python
Embedding("What is RAG?")
```

得到：

```text
[
  0.017,
 -0.283,
  0.619,
 ...
]
```

所以 embedding model 通常不是拿来聊天的。

---

# 11. Anthropic 自己没有 Embedding Model

这页一个很值得注意的地方：

> **Anthropic 当前不提供自己的 embedding model。**

所以不是：

```text
Claude Opus → embedding
Claude Fable → embedding
```

而是 Claude 文档推荐 Voyage AI。([Claude Platform][1])

因此真实 RAG 完全可能：

```text
User Query
    ↓
Voyage Embedding
    ↓
Qdrant
    ↓
Retrieved Documents
    ↓
Claude Opus
    ↓
Answer
```

不同公司的模型一起使用完全正常。

---

# 12. Voyage 现在有哪些模型？

文档当前推荐的主力通用系列是：

```text
voyage-4-large
→ 质量优先

voyage-4
→ 质量 / 效率平衡

voyage-4-lite
→ 成本 / latency 优先

voyage-4-nano
→ 开放权重
```

还有领域模型，比如：

```text
voyage-code-3
→ Code retrieval

voyage-finance-2
→ 金融

voyage-law-2
→ 法律
```

以及 `voyage-context-4` 这种上下文化 chunk embedding，和多模态 embedding。([Claude Platform][1])

---

# 13. Reranker 又是什么？

这页还提到了一个你做 RAG 很值得知道的东西：

```text
rerank-2.5
rerank-2.5-lite
```

Embedding 检索可能：

```text
Query
 ↓
Vector Search
 ↓
Top 100 documents
```

但 Top 100 排序不一定特别准。

于是：

```text
Top 100
   ↓
Reranker
   ↓
重新仔细判断相关性
   ↓
Top 5
```

然后：

```text
Top 5
 ↓
Claude
```

所以一个更成熟的 RAG：

```text
Query
  ↓
Embedding
  ↓
Vector Search
  ↓
Top 100
  ↓
Reranker
  ↓
Top 5
  ↓
Claude
  ↓
Answer
```

官方当前推荐 `rerank-2.5` 作为准确率优先的 reranker，`rerank-2.5-lite` 偏成本和延迟。([Claude Platform][1])

---

## 14. 你可以把自己的 Agentic RAG 项目这样理解

你之前项目里有：

```text
S3
↓
Ray ingestion
↓
Qdrant vector retrieval
↓
Graph retrieval
↓
FastAPI
↓
LLM
```

其中 vector retrieval 这条线可以展开成：

```text
S3 Documents
      ↓
Parser
      ↓
Chunking
      ↓
Embedding Model
      ↓
[0.13, -0.72, ...]
      ↓
Qdrant


User Question
      ↓
Embedding Model
      ↓
Query Vector
      ↓
Qdrant similarity search
      ↓
Top-K Chunks
      ↓
LLM
      ↓
Grounded Answer
```

这其实就是你项目里 **Embedding → Vector DB → Retrieval → LLM** 这条核心链路。

## Cheatsheet

| 概念                      | 人话                                |
| ----------------------- | --------------------------------- |
| Embedding               | 把语义变成数字向量                         |
| Embedding Model         | 文字 → vector                       |
| Vector                  | `[0.12, -0.37, ...]`              |
| Dimension               | vector 有多少个数字                     |
| Semantic Similarity     | 两段文字意思有多像                         |
| Cosine Similarity       | 常用的向量相似度                          |
| `input_type="document"` | 我要 embedding 知识库文档                |
| `input_type="query"`    | 我要 embedding 用户查询                 |
| Vector DB               | 存向量 + 快速找相似向量                     |
| Qdrant                  | Vector Database                   |
| Top-K                   | 找最相似的 K 个结果                       |
| Reranker                | 把初步检索结果再精排                        |
| RAG                     | Retrieve → 把资料给 LLM → Generate    |
| Claude                  | 生成答案，不是 Anthropic embedding 模型    |
| Voyage                  | Claude 文档当前推荐的 embedding provider |

最重要的一句话：

**Embedding 不是让模型“回答问题”，而是把“意思”变成坐标；RAG 再利用这些坐标找到意思最接近的资料，最后交给 Claude 回答。** ([Claude Platform][1])

[Claude 官方：Embeddings](https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings "嵌入 - Claude Platform Docs"