# Dense vs. MoE 模型：激活参数、吞吐量与选型指南

*原文: [https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/) · 来源: web · 生成时间: 2026-09-22T07:32:38.671477+00:00*

## 背景

随着大模型参数规模增长，Dense 模型的算力与显存开销同步上涨，导致推理成本高、扩容难。MoE 借鉴稀疏激活思想，希望把“模型容量”和“每 token 计算量”解耦：总参数可以很大，但单次前向只激活部分专家，从而在不等比增加计算的前提下提升模型容量。

## 痛点

如果不懂 Dense 与 MoE 的取舍，容易只按总参数选型，结果要么用 Dense 导致吞吐低、推理贵，要么用 MoE 却发现显存不够或高并发下延迟不可控。也无法解释“30B 总参数、3B 激活参数”这类模型卡。

## 解决办法

MoE 把 Transformer 每层的单一 FFN 替换成多个专家 FFN，并在前面加一个 router 门控网络。每个 token 在每一层都会重新打分，只路由到 top-k 个专家（可再加一个共享专家）。这样每 token 激活参数 = attention/embedding + 被选专家，而不是全部 FFN。由于所有专家都驻留显存，MoE 用固定显存换更低的 per-token 计算；类似同排量引擎，一个所有气缸每轮都点火，另一个只点火需要的少数气缸。

## 关键代码示例

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MoELayer(nn.Module):
    def __init__(self, d_model, d_ff, num_experts, top_k=2):
        super().__init__()
        self.top_k = top_k
        self.router = nn.Linear(d_model, num_experts, bias=False)
        self.experts = nn.ModuleList([
            nn.Sequential(nn.Linear(d_model, d_ff), nn.GELU(), nn.Linear(d_ff, d_model))
            for _ in range(num_experts)
        ])

    def forward(self, x):
        B, T, D = x.shape
        flat = x.view(B*T, D)                     # (N, D)
        gate_logits = self.router(flat)           # (N, E)
        probs = F.softmax(gate_logits, dim=-1)
        topk_probs, topk_ids = probs.topk(self.top_k, dim=-1)   # (N, K)
        out = torch.zeros_like(flat)

        for e_id, expert in enumerate(self.experts):
            mask = (topk_ids == e_id)             # (N, K) bool
            if not mask.any():
                continue
            tok_indices = mask.nonzero(as_tuple=True)[0]  # token 行号
            tok_weights = topk_probs[mask]        # (M,)
            expert_in = flat[tok_indices]         # (M, D)
            expert_out = expert(expert_in)        # (M, D)
            out[tok_indices] += tok_weights.unsqueeze(-1) * expert_out
        return out.view(B, T, D)

```

这段代码实现了一个简化的 MoE FFN 层：router 对每个 token 打分并取 top-k 专家；按专家 id 建立 mask，把被路由到该专家的 token 取出来送进该专家；专家输出按路由权重加权累加到 out。未被选中的专家完全不运行，体现“激活参数少、总参数大”的原理。真实工程还会加负载均衡损失、专家并行和容量限制。

## 关键流程

1. 每个 token 进入某层后，先经过 router 计算每个专家的 logits/score。
2. 用 softmax 得到专家概率，并选择 top-k 个专家；也可路由到共享专家。
3. 被选中的专家对属于自己的 token 计算输出，按路由权重加权累加，未选中的专家不计算。
4. token 继续经过 attention（或 Mamba 等结构），进入下一层后在新的表示上重新路由。

## 关键点

- MoE 的“active parameters”指每 token 实际经过的 attention/embedding 权重与所选专家权重之和，不是总参数；这是理解推理 FLOPs 的关键。
- MoE 把显存成本（总参数必须加载）和计算成本（只算被激活专家）解耦，因此适合需要高容量、可接受高显存的部署。
- 每个 token 在每一层都会重新路由，专家不会在全网络保持同一个 token；专家专业化更多体现在语法/token 类型而非领域主题。
- batch size 很小时 MoE 的吞吐优势最明显，因为每 token 读取的权重字节更少；高并发下所有专家都可能被命中，吞吐优势仍在但延迟优势收窄。
- 现代 MoE 常加入共享专家（如 Mistral Small 4）或混合 Mamba 层（如 Nemotron 3.5 Lightning），以缓解路由不稳定并优化长上下文 KV cache。

## 对比与权衡

- 相比 Dense 模型，MoE 在相同总参数下每 token 计算量更低、吞吐通常更高，但需要把所有专家驻留显存，VRAM 占用和部署复杂度更高。
- 相比每 token 只选一个专家的 Top-1 MoE，Top-2/共享专家 MoE 在训练稳定性和专家利用上更好，但每 token 激活参数更多、计算略增。
- 相比标准 Transformer MoE（每层都做 attention），Mamba-2 混合 MoE 在长上下文的 KV cache 显存上更好，但引入了 recurrent state，工程实现更复杂。
- 相比 Dense 模型的可预测延迟，MoE 在低并发下延迟可能更低，但路由和专家并行通信开销在高并发或小 batch 下可能让尾延迟更复杂。

## 自测问题

**问: 为什么模型可以标注 30B 总参数但只有 3B 激活参数？**

MoE 每层有多个专家 FFN，token 经 router 只选 top-k；激活参数包含 attention/embedding 和被选专家，其余专家不参与该 token 的 forward，所以每 token 计算量远小于总参数。

**问: MoE 的 router 是每层共享还是每层独立？**

每层独立。token 表示在每层会变化，所以需要在每一层重新路由；这样做可以学习层次化的 token-to-expert 分配。

**问: 既然 MoE 激活参数少，为什么显存占用反而大？**

因为所有专家权重都必须加载到显存供不同 token 随时使用；激活参数少只降低 per-token 计算，不降低总参数存储。

**问: 为什么 batch size 增大后 MoE 的吞吐优势会缩小？**

batch 越大，不同 token 命中不同专家，最终几乎所有专家都会被激活；此时 per-token 计算优势还在，但相对 Dense 不再只读少量权重，路由和通信开销占比上升，所以优势收窄。

**问: 训练 MoE 时为什么需要负载均衡损失？**

router 容易退化为只选少数专家，导致大多数专家闲置且出现训练热点；负载均衡损失或专家容量约束会鼓励 token 更均匀地分配，提升参数利用率和训练稳定性。

## 适用场景

- 推理吞吐优先、显存预算充足的在线服务：选择 MoE，可以用大容量模型服务更多 token/s。
- 需要大容量模型处理多任务/多语言，但单 token 延迟预算有限：选择 MoE，在不增加 per-token FLOPs 的情况下扩大容量。
- 长上下文生成且 KV cache 是瓶颈：选择 Mamba-2+MoE 混合架构或优化 KV cache 的 MoE 模型。
- 需要简单部署、可预测延迟、显存受限的边缘/单卡场景：选择 Dense 模型。

## 标签

`MoE` `Dense Models` `Transformer` `Inference Optimization` `模型选型`
