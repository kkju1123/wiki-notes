---
title: Dense vs. MoE 模型：激活参数、吞吐量与选型指南
url: https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/
source_type: web
folder: questions
author: null
tags:
- MoE
- Dense Models
- Transformer
- Inference Optimization
- 模型选型
summary: 稠密模型每 token 激活全部参数；MoE 经路由只激活少量专家，用显存换计算，适合不同部署约束。
fetched_at: '2026-09-22T07:32:38.671477+00:00'
---

How can a 30B-parameter model activate only 3B parameters per token, and still use the capacity of the larger model? [Nemotron 3.5 Lightning](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) illustrates the answer: It uses a Mixture-of-Experts ([MoE](https://www.nvidia.com/en-us/glossary/mixture-of-experts/)) architecture that selects only a subset of its parameters for each [token](https://blogs.nvidia.com/blog/ai-tokens-explained/).

There are two dominant model architectures: Dense model and MoE. How a model organizes its parameters matters as much as how many it has. The right choice affects throughput, memory cost, and serving complexity more than raw parameter count does. Therefore, choosing between them comes down to your deployment constraints.

Think of the difference like two engines with the same total displacement. One fires every cylinder on every cycle; the other activates only the cylinders it needs.

This post explains:

*   How dense and MoE architectures work
*   How they affect performance
*   When to choose each one

## Dense models vs. MoE models[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#dense_models_vs_moe_models)

In simple terms, dense and MoE models differ in how they use their parameters. A dense model activates all its parameters for every token. An MoE model stores multiple expert networks but routes each token through only a selected subset.

Dense models typically favor simpler, more predictable deployment, while MoE models can deliver greater capacity and throughput when memory and serving complexity are manageable.

[Video 3](https://www.youtube.com/watch?v=Nc481t18dKw)

_Video 1. Dense vs MoE: How to choose the right AI architecture_

### How parameters differ[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#how_parameters_differ)

In a dense model every parameter participates in every forward pass. All 27B parameters of a 27B model fire for every token, through a single shared feed-forward network (FFN) block per decoder layer.

An [MoE model replaces that single shared FFN with multiple FFN blocks](https://research.google/pubs/outrageously-large-neural-networks-the-sparsely-gated-mixture-of-experts-layer/), also known as experts. Through an internal routing process, incoming tokens are processed through a small subset of experts rather than every single parameter. Structurally, inside each decoder layer where a dense model would have one FFN, an MoE layer has multiple (e.g. 8, 64, or 128). A learned **[gate network](https://arxiv.org/abs/2101.03961)**, usually called a **router** network, sits in front of all of them and assigns each incoming token to the top k scoring experts. The selected FFN blocks are the ones that run for that token while the rest are skipped for that particular layer. Most modern MoEs though such as [Mistral Small 4](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) also [run one “shared” expert](https://arxiv.org/abs/2401.06066) that every token is routed to regardless.

### How MoE routing works[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#how_moe_routing_works)

For MoE models, the routing decision made is separate for every layer, meaning a token doesn’t get routed to an expert and stays there for the rest of the computations. At each decoder layer it gets rerouted based on what the token represents at that point in the network. These experts at every layer aren’t experts in the traditional sense where they [specialize](https://www.nvidia.com/en-us/glossary/specialized-ai/) in a subject, but rather their [specializations lie primarily in syntax and token-type patterns](https://arxiv.org/abs/2401.04088) (punctuation, numbers, etc), though this varies by architecture or training methods.

While the router mechanism decides which FFN blocks are skipped at each decoding layer, the tokens still pass through the full attention mechanism as normal. When a model card says “3B active parameters,” it includes both the attention and embedding weights for every token along with the selected FFN weights.

### MoE variants[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#moe_variants)

There are variants to MoE as well, one of them being the NVIDIA model card for Nemotron 3.5 Lightning specifying a [Mamba-2 + MoE + Attention hybrid](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16). The Mamba-2 layers stand in for attention in most layers, carrying a constant-size recurrent state rather than a growing KV cache. That changes the memory profile at long context in a way sparsity alone doesn’t account for.

Rather than Lightning making these routing decisions using the full model’s breadth, it compresses them into a smaller space first to make the routing decision cheaper for the model. Figure 1, below, describes the general transformer-MoE case. Note that an MoE model is a type of sparse model.

![Image 1: Side by side decoder layer diagrams. Left (dense): single shared FFN that every token passes through. Right (MoE): multiple FFN blocks (experts) with a router selecting top-k per token per layer. The router governs the FFN selection only, attention weights are dense in both cases.](https://developer-blogs.nvidia.com/wp-content/uploads/2026/09/image3-8.webp)

_Figure 1. A dense model and sparse model decoder block with different routing mechanisms ([Source: Medium](https://medium.com/@thommaskevin/edgeai-mixture-of-experts-moe-on-raspberry-pi-5e43bbdc91b9))_

## Which is faster: Dense or MoE models?[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#which_is_faster_dense_or_moe_models)

MoE models are often faster for token throughput because they activate only a subset of feed-forward parameters for each token. Dense models activate the full network, but can offer simpler serving and more predictable latency.

At high concurrency, routing and memory movement can narrow MoE’s advantage. Results also depend on hardware, precision, inference framework, and model design. For example, Nemotron 3.5 Lightning has Mamba-2 layers and speculative decoding.

### Key model performance differences[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#key_model_performance_differences)

The difference in performance between the two comes down to two main things. For one, at equal total parameter count the skipped FFN blocks make MoE faster. The more consequential difference between the two is the fact that MoE decouples memory (total params → VRAM to host) from compute (active params → FLOPs per token).

In a dense model hosting and [inference](https://www.nvidia.com/en-us/glossary/ai-inference/) costs scale together, and MoE breaks that link. With MoE, VRAM is paid up front. When every expert loads into memory the compute is paid per token and scales only with the experts that fire. Idle experts cost nothing to run and while still paying the required memory to store it, making that shift from variable per-token compute to fixed memory the main tradeoff.

At batch size 1 decoding is bound by available memory rather than compute, and this is where MoE performs well since it reads fewer weight bytes per token. As the batch size grows, tokens collectively end up using most of the experts in the network and that advantage narrows while the reduction in work per token persists. MoE holds a throughput advantage across batch sizes, but its latency margin compared to a well-optimized dense model compresses at high concurrency.

Modern inference frameworks typically handle routing without discarding tokens, though all experts must live in the GPU memory simultaneously. This leaves less room for the KV cache than a dense model of comparable size would typically have.

### Example comparison[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#example_comparison)

Table 1, below, shows a direct comparison. At equal total parameter size, sparsity shows up plainly in serving throughput: [Gemma 4 31B](https://artificialanalysis.ai/models/gemma-4-31b/providers) and [Nemotron 3.5 Lightning](https://artificialanalysis.ai/models/nemotron-3-5-lightning/providers) are both ~30B total, yet across NVIDIA GPU providers their output-speed ranges don’t overlap. Activating 3B parameters per token is a major reason, though not the only one. [Lightning’s Mamba-2 layers](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) and its [speculative decoding](https://www.youtube.com/shorts/ny3oChJvTDs) contribute independently of the hybrid MoE architecture.

| Model | Architecture | Total params | Active params | VRAM (native) | VRAM (4-bit) | Artificial Analysis Intelligence Index | Output speed‡ | $/M output‡ |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [Gemma 4 31B](https://ai.google.dev/gemma/docs/core/model_card_4) | Dense; multimodal | 31B | 31B | ~61 GB BF16 (1×H100) | ~16 GB | 30 | 36.9 – 222.4 t/s | $0.40 |
| Qwen3.8-27B | Dense; hybrid linear/full attention; MTP head; multimodal | 27B | 27B | ~56 GB BF16 (1×H100) | ~14 GB | 52 | 46.8 t/s | $3.00 |
| [Nemotron 3.5 Lightning](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) | MoE + Mamba-2 attention hybrid | 30B | 3B | ~60 GB BF16 (1×H100) | ~20 GB | 24 | 235.7 – 494.2 t/s | $0.22 |
| [Mistral Small 4](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) | MoE; multimodal | 119B | 6B (8B incl. embeddings) | ~121 GB FP8 (4×H100) | ~71 GB | 20 | 147.3 t/s | $0.60 |

_Table 1. Median output speed and cost data from Artificial Analysis (10K-token input, retrieved Aug 31, 2026); benchmarks cover NVIDIA GPU providers only_

The tradeoffs for both can be observed; for example, Lightning delivers four to five times [Qwen3.8-27B’s](https://artificialanalysis.ai/models/qwen3-8-27b/providers) output speed at a fourteenth of the price, and scores less than half as well on general capability. That is right for an [agentic](https://www.nvidia.com/en-us/glossary/ai-agents/) execution layer running well-specified steps at volume, and wrong where one hard [reasoning](https://www.nvidia.com/en-us/glossary/ai-reasoning/) pass decides the outcome.

## When should you use a dense or MoE model?[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#when_should_you_use_a_dense_or_moe_model)

When it comes to deciding which model to use, it depends on which one will best serve your deployment context.

There are several factors to consider:

*   **Memory Budget:**The memory footprint tracks total parameters rather than active ones, so a 30B MoE and a 30B dense model both need roughly 60 GB. The real question is what you get for those gigabytes: dense converts them into capability, MoE into throughput.
*   **Concurrency:** MoE wins clearly for a single request. Its throughput advantage holds as concurrency scales, but the latency gap narrows. If you’re latency-sensitive at high concurrency, benchmark both before making a decision.
*   **Fine Tuning Plans:** Dense model fine-tuning is simpler; all parameters activate, gradients flow uniformly. A full fine-tune on an MoE model can unbalance the router, making some experts more popular than others or completely phasing out certain ones. LoRA/PEFT approaches help avoid this along with simply freezing the router. NeMo’s supervised fine-tuning recipe for Lightning handles it correctly for that model specifically
*   **Quantization:**The compression ratio isn’t where the architectural difference lives. Two things matter more in practice. First, check what precision the checkpoint already ships at: [Mistral Small 4](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) is natively FP8, so 4-bit buys another ~1.7× (121GB → 71GB), not 4×. Second, both architectures have modules that quantize badly, and they aren’t the same ones. In an MoE it’s the router, where small perturbations flip discrete routing decisions. Recipes hold the router, embeddings and output head higher while in hybrid-attention models it’s the recurrent projections that flip routing decisions. For example, [quantized Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) builds keep the linear-attention block in BF16 for exactly that reason.

## The core tradeoff[](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/#the_core_tradeoff)

Dense and MoE are two answers to one tradeoff: capability per parameter against compute per token. Dense keeps everything simple and everything active, which makes it easier to fine-tune and to serve. MoE buys throughput with memory, and pays for it in serving complexity.

Nemotron 3.5 Lightning is fully open with weights, data, and recipes so you can adapt it to your workflows and deploy it anywhere. To get started, try it on [build.nvidia.com](https://build.nvidia.com/nvidia/nemotron-3.5-lightning-30b-a3b) or through [OpenRouter](https://openrouter.ai/nvidia/nemotron-3.5-lightning:free). Download the weights from [Hugging Face](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4), and [ModelScope](https://modelscope.ai/collections/nv-community/Nemotron-35-Lightning).

For [physical AI](https://www.nvidia.com/en-us/glossary/generative-physical-ai/) workloads, [AgiBot GO-1](https://www.globenewswire.com/news-release/2025/03/10/3040128/0/en/agibot-go-1-the-evolution-of-generalist-embodied-foundation-model-from-vla-to-villa.html) and [Tencent Hy-Embodied-VLM-1.0](https://huggingface.co/tencent/Hy-Embodied-VLM-1.0) are popular options in the ecosystem.