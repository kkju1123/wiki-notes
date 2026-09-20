---
title: Embeddings：文本嵌入与向量检索实战指南
url: https://platform.claude.com/docs/en/build-with-claude/embeddings
source_type: web
folder: claude/message
author: null
tags:
- Embeddings
- 向量检索
- 语义搜索
- Voyage AI
- RAG
summary: 介绍文本嵌入如何将语义映射为向量，并基于 Voyage AI 讲解模型选型、API 调用及搜索、推荐与异常检测应用。
fetched_at: '2026-09-20T02:52:36.818527+00:00'
---

Text embeddings are numerical representations of text that enable measuring semantic similarity. This guide introduces embeddings, their applications, and how to use embedding models for tasks like search, recommendations, and anomaly detection.

## Before implementing embeddings

When selecting an embeddings provider, there are several factors you can consider depending on your needs and preferences:

*   Dataset size & domain specificity: size of the model training dataset and its relevance to the domain you want to embed. Larger or more domain-specific data generally produces better in-domain embeddings
*   Inference performance: embedding lookup speed and end-to-end latency. This is a particularly important consideration for large scale production deployments
*   Customization: options for continued training on private data, or specialization of models for very specific domains. This can improve performance on unique vocabularies

## How to get embeddings with Anthropic

Anthropic does not offer its own embedding model. One embeddings provider that has a wide variety of options and capabilities encompassing all of the preceding considerations is Voyage AI.

Voyage AI makes state-of-the-art embedding models and offers customized models for specific industry domains such as finance and healthcare, or bespoke fine-tuned models for individual customers.

The rest of this guide is for Voyage AI, but you should assess a variety of embeddings vendors to find the best fit for your specific use case.

## Available models

Voyage recommends using the following text embedding models:

**Voyage 4 (latest generation)**

| Model | Context length | Embedding dimension | Description |
| --- | --- | --- | --- |
| `voyage-4-large` | 32,000 | 1024 (default), 256, 512, 2048 | The best general-purpose and multilingual retrieval quality. See the [Voyage 4 blog post](https://blog.voyageai.com/2026/01/15/voyage-4/) for details. |
| `voyage-4` | 32,000 | 1024 (default), 256, 512, 2048 | Optimized for general-purpose and multilingual retrieval quality. Balances quality and efficiency. See the [Voyage 4 blog post](https://blog.voyageai.com/2026/01/15/voyage-4/) for details. |
| `voyage-4-lite` | 32,000 | 1024 (default), 256, 512, 2048 | Optimized for latency and cost. See the [Voyage 4 blog post](https://blog.voyageai.com/2026/01/15/voyage-4/) for details. |
| `voyage-4-nano` | 32,000 | 1024 (default), 256, 512, 2048 | Open-weight model (Apache 2.0 license) available on Hugging Face. See the [Voyage 4 blog post](https://blog.voyageai.com/2026/01/15/voyage-4/) for details. |

**Previous generation**

| Model | Context length | Embedding dimension | Description |
| --- | --- | --- | --- |
| `voyage-3-large` | 32,000 | 1024 (default), 256, 512, 2048 | The best general-purpose and multilingual retrieval quality. See the [voyage-3-large blog post](https://blog.voyageai.com/2025/01/07/voyage-3-large/) for details. |
| `voyage-3.5` | 32,000 | 1024 (default), 256, 512, 2048 | Optimized for general-purpose and multilingual retrieval quality. See the [voyage-3.5 blog post](https://blog.voyageai.com/2025/05/20/voyage-3-5/) for details. |
| `voyage-3.5-lite` | 32,000 | 1024 (default), 256, 512, 2048 | Optimized for latency and cost. See the [voyage-3.5 blog post](https://blog.voyageai.com/2025/05/20/voyage-3-5/) for details. |
| `voyage-code-3` | 32,000 | 1024 (default), 256, 512, 2048 | Optimized for **code** retrieval. See the [voyage-code-3 blog post](https://blog.voyageai.com/2024/12/04/voyage-code-3/) for details. |
| `voyage-finance-2` | 32,000 | 1024 | Optimized for **finance** retrieval and RAG. See the [voyage-finance-2 blog post](https://blog.voyageai.com/2024/06/03/domain-specific-embeddings-finance-edition-voyage-finance-2/) for details. |
| `voyage-law-2` | 16,000 | 1024 | Optimized for **legal** and **long-context** retrieval and RAG. Also improved performance across all domains. See the [voyage-law-2 blog post](https://blog.voyageai.com/2024/04/15/domain-specific-embeddings-and-retrieval-legal-edition-voyage-law-2/) for details. |

Additionally, Voyage recommends the following multimodal embedding models:

| Model | Context length | Embedding dimension | Description |
| --- | --- | --- | --- |
| `voyage-multimodal-3.5` | 32,000 | 1024 (default), 256, 512, 2048 | Rich multimodal embedding model that can vectorize interleaved text, images, and videos. Includes video support as the first production-grade video embedding model. See the [voyage-multimodal-3.5 blog post](https://blog.voyageai.com/2026/01/15/voyage-multimodal-3-5/) for details. |
| `voyage-multimodal-3` | 32,000 | 1024 | Rich multimodal embedding model that can vectorize interleaved text and content-rich images, such as screenshots of PDFs, slides, tables, figures, and more. See the [voyage-multimodal-3 blog post](https://blog.voyageai.com/2024/11/12/voyage-multimodal-3/) for details. |

The following contextualized chunk embedding models produce chunk-level vectors that capture full document context without manual metadata augmentation. Call these models with `contextualized_embed()` instead of `embed()`:

| Model | Context length | Embedding dimension | Description |
| --- | --- | --- | --- |
| `voyage-context-4` | 120,000 | 1024 (default), 256, 512, 2048 | Contextualized chunk embeddings optimized for general-purpose and multilingual retrieval quality. See the [voyage-context-4 blog post](https://blog.voyageai.com/2026/06/29/voyage-context-4/) for details. |
| `voyage-context-3` | 120,000 | 1024 (default), 256, 512, 2048 | Contextualized chunk embeddings optimized for general-purpose and multilingual retrieval quality. See the [voyage-context-3 blog post](https://blog.voyageai.com/2025/07/23/voyage-context-3/) for details. |

Voyage AI also offers rerankers, which take a query and a list of documents and return them ranked by relevance to the query. Call these models with `rerank()`:

| Model | Context length | Description |
| --- | --- | --- |
| `rerank-2.5` | 32,000 | Highest accuracy. Recommended for most applications. See the [rerank-2.5 blog post](https://blog.voyageai.com/2025/08/11/rerank-2-5/) for details. |
| `rerank-2.5-lite` | 32,000 | Optimized for latency and cost. See the [rerank-2.5 blog post](https://blog.voyageai.com/2025/08/11/rerank-2-5/) for details. |

Need help deciding which text embedding model to use? Check out the [Voyage AI FAQ](https://docs.voyageai.com/docs/faq#what-embedding-models-are-available-and-which-one-should-i-use&ref=anthropic).

## Getting started with Voyage AI

To access Voyage embeddings:

1.   Sign up on Voyage AI's website.
2.   Obtain an API key.
3.   Set the API key as an environment variable for convenience:

You can obtain the embeddings by either using the official [`voyageai` Python package](https://github.com/voyage-ai/voyageai-python) or HTTP requests, as described in the following sections.

### Voyage Python library

Install the `voyageai` package using the following command:

Then, you can create a client object and start using it to embed your texts:

`result.embeddings` is a list of two embedding vectors, each containing 1024 floating-point numbers. After running the preceding code, the two embeddings are printed on the screen:

When creating the embeddings, you can specify a few other arguments to the `embed()` function.

For more information on the Voyage Python package, see the [Voyage Python package documentation](https://docs.voyageai.com/docs/embeddings#python-api).

### Voyage HTTP API

You can also get embeddings by requesting Voyage HTTP API. For example, you can send an HTTP request through the `curl` command in a terminal:

The response you would get is a JSON object containing the embeddings and the token usage:

For more information on the Voyage HTTP API, see the [Voyage HTTP API documentation](https://docs.voyageai.com/reference/embeddings-api).

### AWS Marketplace

Voyage embeddings are available on [AWS Marketplace](https://aws.amazon.com/marketplace/seller-profile?id=c9032c7b-70dd-459f-834f-c1e23cf3d092). Instructions for accessing Voyage on AWS are available in the [Voyage AWS Marketplace documentation](https://docs.voyageai.com/docs/aws-marketplace-mongodb-voyage?ref=anthropic).

## Quickstart example

The following brief example shows how to use embeddings.

Suppose you have a small corpus of six documents to retrieve from

First, use Voyage to convert each document into an embedding vector.

The embeddings allow you to do semantic search / retrieval in the vector space. Given an example query,

Next, convert it into an embedding and conduct a nearest neighbor search to find the most relevant document based on the distance in the embedding space.

Note that `input_type="document"` and `input_type="query"` are used for embedding the document and query, respectively. More specification can be found in [Voyage Python library](https://platform.claude.com/docs/en/build-with-claude/embeddings#voyage-python-library).

The output is the fifth document, which is indeed the most relevant to the query:

If you are looking for a detailed set of recipes on how to do RAG with embeddings, including vector databases, check out the [RAG recipe](https://platform.claude.com/cookbook/third-party-pinecone-rag-using-pinecone).

## FAQ

## Pricing

Visit Voyage's [pricing page](https://docs.voyageai.com/docs/pricing?ref=anthropic) for the most up to date pricing details.