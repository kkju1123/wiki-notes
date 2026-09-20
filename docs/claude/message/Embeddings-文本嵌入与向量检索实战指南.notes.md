# Embeddings：文本嵌入与向量检索实战指南

*原文: [https://platform.claude.com/docs/en/build-with-claude/embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings) · 来源: web · 生成时间: 2026-09-20T02:52:36.818527+00:00*

## 背景

传统关键词检索（如 BM25）基于字面匹配，无法理解同义改写和上下文语义。深度学习时代，通过 Transformer 等模型可将文本编码为稠密向量，使语义相近的文本在向量空间中距离更近。Embedding 因此成为 RAG、语义搜索、推荐系统和异常检测等任务的基础组件。

## 痛点

如果只依赖关键词匹配，用户搜索“便宜手机”时无法召回“性价比高的智能手机”这类语义相近内容。不懂嵌入模型选型与调用，容易在维度、上下文长度、领域适配上踩坑，导致检索质量差、推理延迟高或成本失控。

## 解决办法

将文本输入预训练编码器（如 Voyage 的 Transformer 模型），输出固定维度向量（如 1024 维），通过余弦相似度或内积度量语义距离。模型训练阶段通过对比学习让相似文本对向量靠近、不相似文本对远离，从而将语义编码进几何空间。实际使用时，对文档预先计算向量并存入向量数据库，查询时实时嵌入查询向量，通过 ANN 索引快速召回 Top-K，必要时用 reranker 精排。

## 关键代码示例

```python
import os
import voyageai
import numpy as np

vo = voyageai.Client(api_key=os.environ.get("VOYAGE_API_KEY"))

texts = ["我喜欢喝咖啡", "咖啡是我的最爱", "今天股票大跌"]
result = vo.embed(texts, model="voyage-4", input_type="document")
embs = result.embeddings

def cosine(a, b):
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print("维度:", len(embs[0]))           # 默认 1024
print("语义相近分数:", cosine(embs[0], embs[1]))  # 应接近 1
print("语义无关分数:", cosine(embs[0], embs[2]))  # 应较低
```

代码通过 voyageai 客户端调用 voyage-4 模型将三句文本编码为 1024 维向量；余弦相似度计算显示语义相近的两句分数高、无关句分数低，体现 embedding 捕捉语义而非字面。实际生产中会把文档向量预先入库，查询时对 query 调 embed 并用 ANN 检索。

## 关键流程

1. 明确任务（语义搜索、RAG、推荐等）并评估数据领域，选择合适模型（通用/领域专用/多模态）。
2. 在 Voyage AI 注册账号并创建 API Key，设为环境变量。
3. 安装 voyageai Python 包，创建客户端。
4. 对文档/查询调用 embed() 获取向量，对长文档可用 contextualized_embed() 保留上下文。
5. 计算相似度或接入向量数据库进行检索，必要时用 rerank() 精排。

## 关键点

- Embedding 将文本表示为稠密向量，通过向量距离度量语义相似度，是现代语义检索和 RAG 的基石。
- 选择嵌入模型要综合训练数据领域、上下文长度、维度、推理延迟和定制化能力，而不是只看榜单分数。
- Voyage 提供通用、领域专用（金融/法律/代码）、多模态和上下文分块嵌入模型，不同任务应选对应模型。
- 维度越高通常表达力越强，但存储和检索成本也越高；生产环境可根据精度/成本权衡选择 256/512/1024 维。
- 对长文档分块时，contextualized embedding 能把整篇文档上下文注入每个 chunk，减少信息丢失。
- Reranker 作为二阶段精排能显著提升检索精度，弥补双塔模型在精细语义交互上的不足。

## 对比与权衡

- 相比传统 BM25 关键词匹配，embedding 检索在语义理解和同义改写召回上更好，但计算和存储成本更高，且需要维护模型推理。
- 相比通用嵌入模型（如 OpenAI text-embedding-3），Voyage 的领域专用模型（金融/法律/代码）在对应领域准确率更高，但生态和社区支持可能不如主流云厂商广泛。
- 相比开源嵌入模型（如 BGE、E5），Voyage 作为托管 API 省去自部署和运维，但在数据隐私、离线可用性和长期成本上不如自托管开源模型。

## 自测问题

**问: Embedding 为什么能表示语义相似度？**

通过大规模预训练和对比学习，模型被训练为让语义相近文本的向量距离近、无关文本距离远；向量空间中的几何关系因此对应语义关系。

**问: 如何为 RAG 系统选择 embedding 模型？**

先看领域和语言，再看上下文长度是否覆盖 chunk 大小，评估召回指标（recall@k、nDCG），同时权衡延迟和成本；可以对候选模型做小规模标注集评测。

**问: embedding 检索和 reranker 有什么区别？**

embedding 是双塔模型，文档向量可离线预计算，查询时快速 ANN 召回；reranker 是交叉编码器，查询和文档同时输入做深度交互，精度高但计算慢，适合对召回 Top-K 精排。

**问: 长文档怎么做 embedding？**

通常先分块，但朴素分块会丢失文档级上下文；可以用 contextualized embedding 将整篇文档上下文编码到每个 chunk 向量中，或使用长上下文模型直接编码更长文本。

## 适用场景

- 语义搜索与 RAG：对文档库预计算向量，查询时召回语义相关片段。
- 推荐系统：将用户行为序列或物品描述嵌入后计算相似度做召回。
- 异常检测：对日志、文本聚类，将偏离簇心的样本判为异常。
- 多模态检索：用多模态嵌入模型统一向量化文本、图片和视频，支持跨模态搜索。

## 标签

`Embeddings` `向量检索` `语义搜索` `Voyage AI` `RAG`
