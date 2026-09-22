---
title: 大模型面试题整理：LLM 基础、架构、训练、微调、推理、强化学习
url: wikibar://summary/questions/大模型面试题整理-LLM-基础-架构-训练-微调-推理-强化学习
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-22T06:34:41.448485+00:00'
---

# 大模型面试题整理：LLM 基础、架构、训练、微调、推理、强化学习

> 下面不是简单照抄截图，而是按照面试回答的思路重新整理。重点是：先给一句话结论，再解释原理，最后补充面试中容易被追问的点。

---

# 一、大语言模型基础

## 1. 目前主流的开源模型体系有哪些？

目前常见的大语言模型体系包括：

- Llama 系列
- Qwen 系列
- DeepSeek 系列
- GLM / ChatGLM 系列
- Mistral / Mixtral 系列
- InternLM 系列
- Yi 系列
- Baichuan 系列

面试中不建议只是背模型名字，更重要的是知道如何比较模型。

通常从以下几个维度比较：

1. **参数规模**：例如 7B、14B、32B、70B 等。
2. **上下文长度**：模型一次最多能够处理多少 token，例如 32K、128K 等。
3. **语言能力**：中文、英文、多语言能力。
4. **推理能力**：数学、代码、逻辑推理等。
5. **Tool Calling 能力**：能不能稳定生成结构化工具调用参数。
6. **部署成本**：显存需求、吞吐量、推理速度。
7. **量化支持**：FP16、BF16、INT8、INT4 等。
8. **License**：是否允许商业使用。

对于实际 Agent 项目来说，除了 benchmark 分数，更应该关注：

`模型效果 + Tool Calling + 上下文长度 + 推理速度 + 部署成本`

---

## 2. Prefix LM 和 Causal LM 有什么区别？

### Causal LM

Causal LM 就是典型的 GPT / Llama 类模型。

核心规则：

> 当前 token 只能看到自己以及前面的 token，不能看到未来 token。

例如：

```text
我 今天 去 北京
````

预测“北京”时，可以看到：

```text
我 今天 去
```

但是不能提前看到“北京”。

Attention Mask 大致是：

```text
      我 今天 去 北京
我    ✓  ×  ×  ×
今天  ✓  ✓  ×  ×
去    ✓  ✓  ✓  ×
北京  ✓  ✓  ✓  ✓
```

这就是 **Causal Mask / 下三角 Mask**。

训练目标：

```text
P(x) = P(x1) P(x2|x1) P(x3|x1,x2) ...
```

也就是：

> 不断预测下一个 token。

---

### Prefix LM

Prefix LM 把序列分成：

```text
[Prefix 输入区域] + [Generation 生成区域]
```

Prefix 区域内部通常可以双向 Attention。

Generation 区域仍然使用因果 Attention。

例如：

```text
问题：法国首都是哪里？ | 巴黎
<------ Prefix ------>   <-生成->
```

问题部分可以互相看，而生成答案时不能看到未来答案。

一句话区别：

```text
Causal LM：整个序列都是从左向右看。
Prefix LM：输入部分可以双向看，输出部分仍然从左向右生成。
```

---

## 3. 为什么现在的大语言模型大量采用 Decoder-only？

GPT、Llama、Qwen 等生成式 LLM 大量采用 Decoder-only Transformer。

最核心的原因是：

> **架构简单，而且与 next-token prediction 的预训练目标天然匹配。**

### 原因 1：训练目标统一

模型只需要学习：

```text
给定前面的 token → 预测下一个 token
```

例如：

```text
中国的首都是 → 北京
```

海量互联网文本天然就可以构造训练数据，不需要人工标注。

---

### 原因 2：大量任务可以统一成“文本生成”

问答：

```text
Question → Answer
```

翻译：

```text
English → Chinese
```

代码：

```text
Requirement → Code
```

摘要：

```text
Document → Summary
```

甚至分类也可以写成：

```text
评论：这个电影很好看
情感：正面
```

因此很多 NLP 任务都可以统一为：

```text
输入文本 → 继续生成文本
```

---

### 原因 3：Scaling 比较简单

增加：

```text
模型参数
训练数据
训练计算量
```

通常可以持续提高模型能力。

---

### 原因 4：工程生态成熟

现在很多推理优化都围绕 Decoder-only：

```text
KV Cache
Continuous Batching
PagedAttention
FlashAttention
Speculative Decoding
Tensor Parallel
```

所以训练和部署生态非常成熟。

---

## 4. 什么是 LLM“复读机”问题？为什么模型会重复？

例如模型生成：

```text
这个方法非常重要，非常重要，非常重要，非常重要……
```

本质上是：

> 自回归生成进入了重复概率循环。

因为模型生成的新 token 又会成为下一步输入。

假设模型已经生成：

```text
非常重要，非常重要
```

模型可能认为：

```text
P("非常重要" | 前文)
```

依然很高。

于是：

```text
重复 → 提高下一次重复概率 → 再重复
```

形成循环。

常见原因：

* temperature 太低
* top_p / top_k 太保守
* prompt 本身重复
* 模型训练数据问题
* generation 参数不合理
* repetition_penalty 没有设置
* 长文本生成进入局部概率循环

可以使用：

```text
repetition_penalty
frequency_penalty
presence_penalty
合理的 temperature / top_p
stop sequence
```

缓解。

---

## 5. 如何让大模型处理更长的文本？

这是一个非常重要的工程问题。

常见方法可以分为五类。

### 方法一：直接使用长上下文模型

例如模型支持：

```text
32K
128K
甚至更长 Context Window
```

最简单，但是：

> 能放进去 ≠ 模型一定能有效利用所有信息。

---

### 方法二：Chunk + RAG

把长文档：

```text
完整文档
↓
Chunk 1
Chunk 2
Chunk 3
...
Chunk N
```

用户提问后：

```text
Query
  ↓
Embedding
  ↓
Vector Search
  ↓
Top-K Chunks
  ↓
LLM
```

这样不需要把整个文档塞给模型。

---

### 方法三：Sliding Window

例如：

```text
Chunk1: token 0-1000
Chunk2: token 800-1800
Chunk3: token 1600-2600
```

保留 overlap，减少切块导致的上下文丢失。

---

### 方法四：Hierarchical Summarization

例如一本 300 页文档：

```text
每页总结
   ↓
每章总结
   ↓
整本书总结
```

也就是：

```text
Map → Reduce
```

适合超长文档摘要。

---

### 方法五：长上下文位置编码技术

Transformer 需要知道 token 的位置。

现代 LLM 经常使用 RoPE。

为了扩展 Context Window，可以使用一些 RoPE Scaling / Position Interpolation 类技术。

一句话总结：

> 工程上最常见的是“长 Context + RAG + Chunk + Summary”，而不是无脑把所有文本塞进 Prompt。

---

# 二、大语言模型架构

## 1. Attention 到底是什么？

一句人话：

> **Attention 就是在当前 token 需要理解上下文时，判断其他 token 谁更重要，然后按重要程度加权读取信息。**

例如：

```text
小明把苹果放在桌子上，因为它很重。
```

模型处理“它”时，需要判断：

```text
它 → 苹果？
它 → 桌子？
```

Attention 就是在计算这些 token 之间的相关程度。

---

### Q、K、V

每个 token 会生成三个向量：

```text
Q = Query
K = Key
V = Value
```

可以理解为：

```text
Q：我正在找什么？
K：我这里有什么信息？
V：我的具体信息是什么？
```

首先计算：

```text
QK^T
```

表示：

> Query 和各个 Key 有多匹配？

然后：

```text
Attention(Q,K,V)
=
softmax(QK^T / sqrt(d_k))V
```

完整过程：

```text
Q
 ↓
QK^T
 ↓
/ sqrt(d_k)
 ↓
softmax
 ↓
Attention Score
 ↓
× V
 ↓
加权后的上下文表示
```

### 为什么除以 `sqrt(d_k)`？

维度越大，点积数值可能越大，softmax 容易过度饱和，梯度变小。

所以使用：

```text
QK^T / sqrt(d_k)
```

控制数值范围。

---

## 2. Self-Attention 是什么？

Self-Attention 的特点是：

```text
Q、K、V
```

都来自同一个序列。

例如：

```text
I love machine learning
```

每一个 token 都会和这个序列中的其他 token 计算关系。

所以叫：

> Self Attention —— 序列自己关注自己。

---

## 3. Attention 和传统 Seq2Seq 有什么区别？

传统 RNN Seq2Seq：

```text
Input
 ↓
Encoder
 ↓
一个固定长度 Context Vector
 ↓
Decoder
```

问题是：

> 一整句话的信息被压缩到一个固定向量里。

句子越长越容易丢信息。

加入 Attention 后：

```text
Encoder Hidden State 1
Encoder Hidden State 2
Encoder Hidden State 3
...
          ↑
       Attention
          ↑
       Decoder
```

Decoder 每一步都可以重新访问 Encoder 的不同位置。

Transformer 更进一步：

> 直接用 Attention 作为主要的信息交互机制，不再依赖 RNN 的逐步递归计算。

因此能够更好地并行训练。

---

## 4. Multi-Head Attention 为什么需要多个 Head？

如果只有一个 Attention：

```text
所有关系 → 一个 Attention 空间
```

而自然语言存在很多不同关系：

```text
语法关系
指代关系
位置关系
实体关系
语义关系
```

所以 Transformer 使用多个 Head：

```text
Head 1 → 学语法
Head 2 → 学指代
Head 3 → 学实体关系
Head 4 → 学其他模式
```

这里是直觉理解，并不是规定每个 Head 一定负责某一种关系。

---

### 为什么每个 Head 要降维？

假设：

```text
d_model = 4096
num_heads = 32
```

通常：

```text
d_head = d_model / num_heads = 128
```

每个 Head：

```text
Q_i = XW_i^Q
K_i = XW_i^K
V_i = XW_i^V
```

然后：

```text
head_i = Attention(Q_i,K_i,V_i)
```

最后：

```text
Concat(head_1,...,head_h) W^O
```

重新投影回：

```text
d_model
```

这样做既允许不同 Head 学习不同表示子空间，又把总体维度和计算规模控制住。

---

## 5. Encoder 和 Decoder 的主要区别？

### Encoder

Encoder Self-Attention 通常是：

```text
Bidirectional Attention
```

例如：

```text
A B C D
```

B 可以看：

```text
A B C D
```

即前后文都能利用。

BERT 就是典型 Encoder 模型。

---

### Decoder

Decoder 使用：

```text
Causal / Masked Self-Attention
```

例如生成：

```text
A B C D
```

生成 C 时：

```text
可以看：A B
不能看：D
```

因为 D 是未来 token。

GPT / Llama 等就是典型 Decoder-only 模型。

---

## 6. 为什么 BERT Mask 15% 左右的 token？

BERT 的 Masked Language Modeling 会随机选择一部分 token 作为预测目标。

原论文采用：

```text
15%
```

直觉上是：

```text
Mask 太少
→ 学习信号不足

Mask 太多
→ 原句破坏严重
→ 上下文信息不足
```

15% 是实验选择出的折中值，并不是数学上唯一正确的比例。

而且被选中的 15% token 并不全部直接替换为 `[MASK]`，经典 BERT 的处理是：

```text
80% → [MASK]
10% → 随机 token
10% → 保持原 token
```

这样可以减少预训练和实际使用之间 `[MASK]` token 带来的差异。

---

## 7. BERT 的非线性来自哪里？

最主要来自：

```text
FFN 中的 GELU
Attention 中的 Softmax
```

其中 Transformer Block 中：

```text
Attention
↓
FFN
↓
GELU
```

FFN 可以写成：

```text
FFN(x) = W2 GELU(W1x + b1) + b2
```

如果整个神经网络只有线性层：

```text
Linear(Linear(x))
```

无论堆多少层，本质仍然可以合并成一个线性变换。

所以激活函数提供非线性表达能力。

注意：

> Residual Connection 和 LayerNorm 对深层网络训练非常重要，但“非线性的主要来源”更准确地说是 GELU 和 Softmax，而不是残差连接本身。

---

## 8. 为什么 Transformer 要使用 LayerNorm？

LayerNorm 对单个 token 的 hidden features 做归一化。

直觉上：

```text
某层输出特别大
↓
下一层输入分布剧烈变化
↓
训练不稳定
```

LayerNorm 可以帮助控制激活尺度，提高训练稳定性。

与 BatchNorm 最大区别：

```text
BatchNorm → 依赖 Batch 统计量
LayerNorm → 不依赖 Batch 大小
```

NLP 中：

```text
batch size 可能变化
sequence length 可能变化
```

因此 LayerNorm 更适合 Transformer。

现代 LLM 中还经常使用：

```text
RMSNorm
```

例如：

```text
Llama
```

等架构大量使用 RMSNorm。

---

# 三、训练数据与对齐

## 1. SFT 是什么？数据是什么格式？

SFT：

```text
Supervised Fine-Tuning
```

即：

> 使用高质量“输入 → 标准回答”数据继续训练预训练模型。

最简单的数据：

```json
{
  "instruction": "解释什么是过拟合",
  "input": "",
  "output": "过拟合是指模型在训练集表现很好，但在未见数据上泛化能力较差……"
}
```

现代 Chat Model 更常见：

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "什么是过拟合？"
    },
    {
      "role": "assistant",
      "content": "过拟合是……"
    }
  ]
}
```

训练目标依然通常是：

```text
Next Token Prediction
```

只是训练数据从普通互联网文本变成了：

```text
Instruction / Conversation
```

---

## 2. Reward Model 的训练数据是什么？

Reward Model 学习：

> 哪个回答更符合人类偏好？

数据通常是：

```text
Prompt
Chosen Response
Rejected Response
```

例如：

```json
{
  "prompt": "如何学习 NLP？",
  "chosen": "可以先学习机器学习基础，然后学习 Transformer……",
  "rejected": "随便看看就行。"
}
```

训练目标：

```text
Reward(chosen) > Reward(rejected)
```

常见 pairwise loss：

```text
L = -log σ(r_chosen - r_rejected)
```

意思就是：

> 让好回答的 reward 比差回答高。

---

## 3. PPO 阶段的数据是什么？

RLHF 的 PPO 阶段核心流程不是一个固定的离线 `prompt-response` 数据集，而是：

```text
Prompt
 ↓
Policy Model 生成 Response
 ↓
Reward Model 打分
 ↓
计算 Reward / Advantage
 ↓
PPO 更新 Policy
```

也就是说 response 往往是：

> 当前策略模型在线采样生成的。

同时通常还会加入：

```text
KL Penalty
```

限制模型不要偏离 Reference Model 太远。

---

# 四、模型微调

## 1. 什么是微调？怎么微调？

微调就是：

> 在已经预训练好的模型基础上，用自己的任务数据继续训练。

流程：

```text
Base Model
   ↓
准备 Dataset
   ↓
数据清洗
   ↓
Tokenization
   ↓
选择 Full Fine-tuning / PEFT
   ↓
Training
   ↓
Validation
   ↓
Save / Merge Adapter
   ↓
Deploy
```

不需要从头训练一个 LLM。

例如：

```text
Qwen Base
+
医疗问答数据
↓
医疗领域模型
```

---

## 2. 全量微调和 PEFT 有什么区别？

### Full Fine-tuning

更新：

```text
模型几乎所有参数
```

优点：

```text
模型适应能力强
```

缺点：

```text
GPU 显存高
训练成本高
Checkpoint 大
```

---

### PEFT

PEFT：

```text
Parameter-Efficient Fine-Tuning
```

冻结大部分 Base Model，只训练少量新增参数。

例如：

```text
LoRA
Prefix Tuning
Prompt Tuning
Adapter
```

优点：

```text
显存低
训练快
Checkpoint 小
一个 Base Model 可以挂多个 Adapter
```

---

## 3. PEFT 有什么优点？

最重要的几个：

```text
减少训练参数
降低显存占用
降低训练成本
训练速度更快
Adapter 文件小
方便多任务切换
```

例如：

```text
Base Model
   ├── Finance LoRA
   ├── Medical LoRA
   └── Code LoRA
```

不需要存：

```text
3 个完整模型
```

而是：

```text
1 个 Base Model + 3 个小 Adapter
```

---

## 4. LoRA 是什么？

LoRA：

```text
Low-Rank Adaptation
```

假设原模型有一个权重矩阵：

```text
W
```

Full Fine-tuning：

```text
直接训练 W
```

LoRA：

```text
W' = W + ΔW
```

并假设：

```text
ΔW = BA
```

其中：

```text
A ∈ R^(r × d)
B ∈ R^(d × r)
```

而：

```text
r << d
```

所以：

```text
巨大矩阵 ΔW
```

被分解成：

```text
两个很小的低秩矩阵 A、B
```

训练时：

```text
W 冻结
A、B 更新
```

因此训练参数量大幅减少。

一句话：

> LoRA 不去修改整个大矩阵，而是学习一个低秩的“小补丁”。

---

## 5. QLoRA 又是什么？

QLoRA 可以简单理解成：

```text
Quantization + LoRA
```

Base Model：

```text
4-bit 量化
```

LoRA Adapter：

```text
正常训练
```

因此：

```text
4-bit Base Model
+
Trainable LoRA
```

可以进一步减少显存。

注意：

> QLoRA 的重点不是“把 LoRA 本身变成 4bit”，而是将冻结的基础模型低比特量化，同时训练 LoRA 参数。

---

## 6. 常见 PEFT 方法有哪些？

### LoRA

```text
低秩矩阵更新
```

目前最常见。

### Prefix Tuning

在 Transformer 中学习一组可训练的：

```text
Prefix / Virtual Tokens
```

Base Model 参数基本冻结。

### Prompt Tuning

学习：

```text
Soft Prompt Embedding
```

而不是人工写 Prompt。

### Adapter

在 Transformer Block 中加入小型神经网络模块：

```text
Transformer
↓
Adapter
↓
Transformer
```

只训练 Adapter。

### P-Tuning / P-Tuning v2

也是可学习 Prompt/Prefix 的思路，P-Tuning v2 将可训练提示扩展到多层。

---

## 7. PEFT 有什么问题？

PEFT 虽然省资源，但不是万能的。

常见问题：

```text
对超参数敏感
跨任务泛化可能不稳定
多个 Adapter 管理复杂
Adapter 组合可能产生干扰
不同 Base Model / 版本兼容复杂
能力上限有时低于 Full Fine-tuning
```

LoRA 常见关键参数：

```text
r
alpha
dropout
target_modules
```

例如：

```text
target_modules = ["q_proj", "v_proj"]
```

不同配置会明显影响效果。

---

# 五、大模型推理

## 1. 为什么 LLM 推理很占显存？

显存主要消耗可以理解为：

```text
GPU Memory
=
Model Weights
+
KV Cache
+
Temporary Activations / Buffers
+
Runtime Overhead
```

---

### 第一部分：模型参数

假设：

```text
7B parameters
```

FP16 每个参数约：

```text
2 bytes
```

那么光权重理论上就约：

```text
7B × 2 bytes ≈ 14 GB
```

INT8：

```text
≈ 7 GB
```

INT4：

```text
≈ 3.5 GB
```

实际部署还会有额外开销。

---

### 第二部分：KV Cache

自回归生成时，如果每生成一个 token 都重新计算之前所有 token：

```text
计算量巨大
```

所以模型缓存每层之前 token 的：

```text
Key
Value
```

也就是：

```text
KV Cache
```

随着：

```text
Sequence Length ↑
Batch Size ↑
```

KV Cache 会不断增大。

所以长上下文 + 高并发场景尤其吃显存。

---

## 2. GPU 和 CPU 推理速度有什么区别？

LLM 的核心运算大量是：

```text
Matrix Multiplication
```

GPU 有大量并行计算单元，因此特别适合矩阵运算。

所以通常：

```text
GPU >> CPU
```

尤其是：

```text
大模型
高 Batch
高并发
长序列
```

场景。

CPU 的优势则是：

```text
成本可能更低
部署简单
内存容量大
边缘设备场景
```

所以 CPU 并不是不能跑 LLM，只是大型模型的吞吐通常远低于合适的 GPU/加速器。

---

## 3. INT8 和 FP16 推理有什么区别？

FP16：

```text
16 bit floating point
```

INT8：

```text
8 bit integer
```

INT8 理论上：

```text
显存更低
内存带宽压力更小
```

在硬件和推理框架对 INT8 有良好支持时，也可能获得更高吞吐。

但是不能简单说：

```text
INT8 一定比 FP16 快
```

因为真实速度取决于：

```text
GPU 型号
Tensor Core 支持
Quantization Kernel
Batch Size
模型结构
推理框架
```

精度方面通常：

```text
FP16/BF16 更接近原始模型
INT8 可能有少量精度损失
INT4 压缩更激进
```

---

## 4. LLM 有“推理能力”吗？

可以说 LLM 能完成：

```text
多步问题求解
数学推导
代码推理
规划
逻辑任务
```

但它的生成机制本质仍然是：

```text
P(next_token | previous_tokens)
```

因此：

```text
能产生正确推理
≠
每次推理都可靠
```

它可能：

```text
推理错误
计算错误
遗漏条件
产生幻觉
```

所以真实 Agent 系统通常会结合：

```text
LLM
+
Tools
+
Retrieval
+
Code Execution
+
Validation
```

提高可靠性。

---

## 5. LLM 常见生成参数有哪些？

最常见：

```text
temperature
top_p
top_k
max_new_tokens
repetition_penalty
stop
```

---

### temperature

控制概率分布的随机程度。

原始 logits：

```text
z_i
```

调整：

```text
softmax(z_i / T)
```

T 小：

```text
概率分布更尖锐
→ 更确定
```

T 大：

```text
概率更平
→ 更多样
```

注意：

```text
事实问答 ≠ 固定必须 0.2~0.7
创意任务 ≠ 固定必须 0.8~1.1
```

这些只是经验范围，具体取决于模型和任务。

---

### top_k

只保留概率最高的：

```text
K 个 token
```

例如：

```text
top_k = 50
```

只从概率最高的 50 个 token 中采样。

---

### top_p

也叫：

```text
Nucleus Sampling
```

选择累计概率达到 p 的最小 token 集合。

例如：

```text
top_p = 0.9
```

选择累计概率达到 90% 的候选 token。

---

### max_new_tokens

限制：

```text
最多生成多少个新 token
```

---

### repetition_penalty

降低已经出现 token 再次出现的倾向，用于减少重复。

---

### stop

遇到指定 token/string 后停止生成。

例如：

```python
stop = ["</answer>"]
```

---

## 6. 有哪些节省显存的训练/微调/推理方法？

这个问题最好分成：

```text
训练优化
微调优化
推理优化
```

---

### 训练

可以使用：

```text
Mixed Precision
Gradient Accumulation
Gradient Checkpointing
ZeRO
FSDP
Tensor Parallel
Pipeline Parallel
```

---

### 微调

```text
LoRA
QLoRA
PEFT
4-bit / 8-bit Quantization
Gradient Checkpointing
```

---

### 推理

```text
INT8 / INT4 Quantization
KV Cache
PagedAttention
Continuous Batching
Tensor Parallel
Prefix Caching
FlashAttention
```

例如 vLLM 的一个核心思想就是：

```text
PagedAttention
```

更高效地管理 KV Cache 内存。

---

# 六、强化学习与 RLHF

## 1. 什么是强化学习？

强化学习的基本框架：

```text
Agent
  ↓ Action
Environment
  ↓
State + Reward
  ↓
Agent
```

核心元素：

```text
State
Action
Reward
Policy
Environment
```

Agent 根据状态：

```text
s_t
```

选择动作：

```text
a_t
```

环境返回：

```text
r_t
s_{t+1}
```

目标不是让每一步 Reward 最大，而是最大化长期累计回报：

```text
G_t = r_t + γr_{t+1} + γ²r_{t+2} + ...
```

即：

```text
maximize E[Σ γ^t r_t]
```

---

## 2. 什么是 RLHF？

RLHF：

```text
Reinforcement Learning from Human Feedback
```

经典 RLHF 流程可以理解为三步。

### Step 1：SFT

先让模型学会：

```text
按照指令正常回答
```

流程：

```text
Pretrained Model
      ↓
Instruction Dataset
      ↓
SFT Model
```

---

### Step 2：训练 Reward Model

对于同一个 Prompt：

```text
Response A
Response B
```

人类进行偏好选择：

```text
A > B
```

然后训练 Reward Model：

```text
RM(prompt, response) → scalar reward
```

让：

```text
Reward(A) > Reward(B)
```

---

### Step 3：强化学习优化 Policy

Policy Model 生成回答：

```text
Prompt
↓
Policy
↓
Response
↓
Reward Model
↓
Reward
```

再使用 PPO 等算法优化模型。

经典形式：

```text
Pretrained Model
      ↓
     SFT
      ↓
Reward Model
      ↓
PPO / RL
      ↓
Aligned Model
```

---

## 3. PPO 在 RLHF 中做什么？

PPO：

```text
Proximal Policy Optimization
```

核心目标：

> 根据 Reward 提升高质量回答的概率，但不要一次更新得太猛烈。

如果直接根据 Reward 大幅更新模型：

```text
Policy 可能突然崩掉
```

PPO 使用 clipped objective 限制 Policy 更新幅度。

可以直觉理解为：

```text
旧模型：回答方式 A
↓
Reward 告诉模型：B 更好
↓
模型朝 B 调整一点
↓
不要一次变化太大
```

RLHF 中还通常加入：

```text
KL(Policy || Reference)
```

限制模型不要偏离原来的 SFT/Reference Model 太远。

---

# 七、把这些知识串起来

面试时不要把这些概念看成几十个独立知识点，它们实际上是一条完整的大模型技术链。

```text
                    大语言模型
                        │
              ┌─────────┴─────────┐
              │                   │
           模型架构              模型训练
              │                   │
        Transformer           Pretraining
              │                   │
      ┌───────┼───────┐           ↓
      │       │       │           SFT
 Attention   FFN   LayerNorm       │
      │                           ↓
 Q K V                         Alignment
      │                     ┌─────┴─────┐
Multi-Head                 RLHF        DPO等
                              │
                         RM + PPO
                              
                              
                 预训练模型
                     │
              ┌──────┴──────┐
              │             │
        Full Fine-tune      PEFT
                            │
                ┌───────────┼──────────┐
                │           │          │
              LoRA       Prefix     Adapter
                │
              QLoRA


                    模型部署
                       │
          ┌────────────┼────────────┐
          │            │            │
       Quantization  KV Cache   Batching
          │            │            │
      INT8/INT4    PagedAttention   vLLM
```

所以你可以把整个面试知识体系记成一句话：

> **Transformer 决定模型怎么计算 → Pretraining 让模型学语言和知识 → SFT 让模型学会按指令回答 → RLHF/DPO 等让模型进一步对齐偏好 → LoRA/QLoRA 让我们低成本定制模型 → Quantization、KV Cache、PagedAttention 等解决模型部署和推理效率问题。**

---

# 八、面试高频追问速记

如果面试官继续往下追，最容易从截图这些题延伸出下面这些问题：

| 问题                          | 一句话回答                                |
| --------------------------- | ------------------------------------ |
| Attention 为什么除 `sqrt(d_k)`？ | 防止 QK 点积过大导致 softmax 饱和              |
| Q/K/V 是什么？                  | Q 表示查询需求，K 用来匹配，V 携带真正的信息            |
| 为什么 Multi-Head？             | 在不同表示子空间学习不同关系                       |
| Decoder 为什么需要 Mask？         | 防止训练时偷看未来 token                      |
| GPT 为什么 Decoder-only？       | 与 next-token prediction 和生成任务天然匹配    |
| BERT 为什么 Encoder-only？      | 主要目标是学习双向上下文表示                       |
| KV Cache 是什么？               | 缓存历史 token 的 K/V，避免生成时重复计算           |
| KV Cache 缺点？                | 上下文和并发越大，占用显存越多                      |
| LoRA 为什么省显存？                | 冻结原参数，只训练低秩增量矩阵                      |
| QLoRA 为什么更省？                | Base Model 进一步采用低比特量化                |
| INT8 一定更快吗？                 | 不一定，取决于硬件和 Kernel                    |
| RAG 和 Fine-tuning 区别？       | RAG 注入外部知识，Fine-tuning 改模型参数/行为      |
| SFT 学什么？                    | 学习如何按照 Instruction / Conversation 回答 |
| RM 学什么？                     | 学习人类对不同回答的偏好                         |
| PPO 做什么？                    | 根据 Reward 更新 Policy，同时限制更新幅度         |
| LayerNorm 为什么不用 BatchNorm？  | 不依赖 Batch 统计，更适合序列模型                 |
| temperature 越低越好吗？          | 不是，只是输出更确定，任务不同需求不同                  |
| 长上下文一定比 RAG 好吗？             | 不一定，成本、检索精度、信息利用率都需要考虑               |
| 参数量越大一定越好吗？                 | 不一定，还取决于数据、训练方法、架构和具体任务              |

```
```