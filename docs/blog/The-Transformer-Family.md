---
title: The Transformer Family
url: https://lilianweng.github.io/posts/2020-04-07-the-transformer-family/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:46:08.744393+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
The Transformer Family
Date: April 7, 2020 | Estimated Reading Time: 25 min | Author: Lilian Weng
Table of Contents

[Updated on 2023-01-27: After almost three years, I did a big refactoring update of this post to incorporate a bunch of new Transformer models since 2020. The enhanced version of this post is here: The Transformer Family Version 2.0. Please refer to that post on this topic.]


It has been almost two years since my last post on attention. Recent progress on new and enhanced versions of Transformer motivates me to write another post on this specific topic, focusing on how the vanilla Transformer can be improved for longer-term attention span, less memory and computation consumption, RL task solving and more.

Notations
Symbol	Meaning

𝑑
	The model size / hidden state dimension / positional encoding size.

ℎ
	The number of heads in multi-head attention layer.

𝐿
	The segment length of input sequence.

𝑋
∈
𝑅
𝐿
×
𝑑
	The input sequence where each element has been mapped into an embedding vector of shape 
𝑑
, same as the model size.

𝑊
𝑘
∈
𝑅
𝑑
×
𝑑
𝑘
	The key weight matrix.

𝑊
𝑞
∈
𝑅
𝑑
×
𝑑
𝑘
	The query weight matrix.

𝑊
𝑣
∈
𝑅
𝑑
×
𝑑
𝑣
	The value weight matrix. Often we have 
𝑑
𝑘
=
𝑑
𝑣
=
𝑑
.

𝑊
𝑖
𝑘
,
𝑊
𝑖
𝑞
∈
𝑅
𝑑
×
𝑑
𝑘
/
ℎ
;
𝑊
𝑖
𝑣
∈
𝑅
𝑑
×
𝑑
𝑣
/
ℎ
	The weight matrices per head.

𝑊
𝑜
∈
𝑅
𝑑
𝑣
×
𝑑
	The output weight matrix.

𝑄
=
𝑋
𝑊
𝑞
∈
𝑅
𝐿
×
𝑑
𝑘
	The query embedding inputs.

𝐾
=
𝑋
𝑊
𝑘
∈
𝑅
𝐿
×
𝑑
𝑘
	The key embedding inputs.

𝑉
=
𝑋
𝑊
𝑣
∈
𝑅
𝐿
×
𝑑
𝑣
	The value embedding inputs.

𝑆
𝑖
	A collection of key positions for the 
𝑖
-th query 
𝑞
𝑖
 to attend to.

𝐴
∈
𝑅
𝐿
×
𝐿
	The self-attention matrix between a input sequence of length 
𝐿
 and itself. 
𝐴
=
softmax
(
𝑄
𝐾
⊤
/
𝑑
𝑘
)
.

𝑎
𝑖
𝑗
∈
𝐴
	The scalar attention score between query 
𝑞
𝑖
 and key 
𝑘
𝑗
.

𝑃
∈
𝑅
𝐿
×
𝑑
	position encoding matrix, where the 
𝑖
-th row 
𝑝
𝑖
 is the positional encoding for input 
𝑥
𝑖
.
Attention and Self-Attention

Attention is a mechanism in the neural network that a model can learn to make predictions by selectively attending to a given set of data. The amount of attention is quantified by learned weights and thus the output is usually formed as a weighted average.

Self-attention is a type of attention mechanism where the model makes prediction for one part of a data sample using other parts of the observation about the same sample. Conceptually, it feels quite similar to non-local means. Also note that self-attention is permutation-invariant; in other words, it is an operation on sets.

There are various forms of attention / self-attention, Transformer (Vaswani et al., 2017) relies on the scaled dot-product attention: given a query matrix 
𝑄
, a key matrix 
𝐾
 and a value matrix 
𝑉
, the output is a weighted sum of the value vectors, where the weight assigned to each value slot is determined by the dot-product of the query with the corresponding key:

Attention
(
𝑄
,
𝐾
,
𝑉
)
=
softmax
(
𝑄
𝐾
⊤
𝑑
𝑘
)
𝑉

And for a query and a key vector 
𝑞
𝑖
,
𝑘
𝑗
∈
𝑅
𝑑
 (row vectors in query and key matrices), we have a scalar score:

𝑎
𝑖
𝑗
=
softmax
(
𝑞
𝑖
𝑘
𝑗
⊤
𝑑
𝑘
)
=
exp
⁡
(
𝑞
𝑖
𝑘
𝑗
⊤
)
𝑑
𝑘
∑
𝑟
∈
𝑆
𝑖
exp
⁡
(
𝑞
𝑖
𝑘
𝑟
⊤
)

See my old post for other types of attention if interested.

Multi-Head Self-Attention

The multi-head self-attention module is a key component in Transformer. Rather than only computing the attention once, the multi-head mechanism splits the inputs into smaller chunks and then computes the scaled dot-product attention over each subspace in parallel. The independent attention outputs are simply concatenated and linearly transformed into expected dimensions.

	
	
MultiHeadAttention
(
𝑋
𝑞
,
𝑋
𝑘
,
𝑋
𝑣
)
	
=
[
head
1
;
…
;
head
ℎ
]
𝑊
𝑜


where head
𝑖
	
=
Attention
(
𝑋
𝑞
𝑊
𝑖
𝑞
,
𝑋
𝑘
𝑊
𝑖
𝑘
,
𝑋
𝑣
𝑊
𝑖
𝑣
)

where 
[
.
;
.
]
 is a concatenation operation. 
𝑊
𝑖
𝑞
,
𝑊
𝑖
𝑘
∈
𝑅
𝑑
×
𝑑
𝑘
/
ℎ
,
𝑊
𝑖
𝑣
∈
𝑅
𝑑
×
𝑑
𝑣
/
ℎ
 are weight matrices to map input embeddings of size 
𝐿
×
𝑑
 into query, key and value matrices. And 
𝑊
𝑜
∈
𝑅
𝑑
𝑣
×
𝑑
 is the output linear transformation. All the weights should be learned during training.

Illustration of the multi-head scaled dot-product attention mechanism. (Image source: Figure 2 in Vaswani, et al., 2017)
Transformer

The Transformer (which will be referred to as “vanilla Transformer” to distinguish it from other enhanced versions; Vaswani, et al., 2017) model has an encoder-decoder architecture, as commonly used in many NMT models. Later simplified Transformer was shown to achieve great performance in language modeling tasks, like in encoder-only BERT or decoder-only GPT.

Encoder-Decoder Architecture

The encoder generates an attention-based representation with capability to locate a specific piece of information from a large context. It consists of a stack of 6 identity modules, each containing two submodules, a multi-head self-attention layer and a point-wise fully connected feed-forward network. By point-wise, it means that it applies the same linear transformation (with same weights) to each element in the sequence. This can also be viewed as a convolutional layer with filter size 1. Each submodule has a residual connection and layer normalization. All the submodules output data of the same dimension 
𝑑
.

The function of Transformer decoder is to retrieve information from the encoded representation. The architecture is quite similar to the encoder, except that the decoder contains two multi-head attention submodules instead of one in each identical repeating module. The first multi-head attention submodule is masked to prevent positions from attending to the future.

The architecture of the vanilla Transformer model. (Image source: Figure 17)

Positional Encoding

Because self-attention operation is permutation invariant, it is important to use proper positional encodingto provide order information to the model. The positional encoding 
𝑃
∈
𝑅
𝐿
×
𝑑
 has the same dimension as the input embedding, so it can be added on the input directly. The vanilla Transformer considered two types of encodings:

(1) Sinusoidal positional encoding is defined as follows, given the token position 
𝑖
=
1
,
…
,
𝐿
 and the dimension 
𝛿
=
1
,
…
,
𝑑
:

	

	
PE
(
𝑖
,
𝛿
)
=
{
sin
⁡
(
𝑖
10000
2
𝛿
′
/
𝑑
)
	
if 
𝛿
=
2
𝛿
′


cos
⁡
(
𝑖
10000
2
𝛿
′
/
𝑑
)
	
if 
𝛿
=
2
𝛿
′
+
1

In this way each dimension of the positional encoding corresponds to a sinusoid of different wavelengths in different dimensions, from 
2
𝜋
 to 
10000
⋅
2
𝜋
.

Sinusoidal positional encoding with 
𝐿
=
32
 and 
𝑑
=
128
. The value is between -1 (black) and 1 (white) and the value 0 is in gray.

(2) Learned positional encoding, as its name suggested, assigns each element with a learned column vector which encodes its absolute position (Gehring, et al. 2017).

Quick Follow-ups

Following the vanilla Transformer, Al-Rfou et al. (2018) added a set of auxiliary losses to enable training a deep Transformer model on character-level language modeling which outperformed LSTMs. Several types of auxiliary tasks are used:

Instead of producing only one prediction at the sequence end, every immediate position is also asked to make a correct prediction, forcing the model to predict given smaller contexts (e.g. first couple tokens at the beginning of a context window).
Each intermediate Transformer layer is used for making predictions as well. Lower layers are weighted to contribute less and less to the total loss as training progresses.
Each position in the sequence can predict multiple targets, i.e. two or more predictions of the future tokens.
Auxiliary prediction tasks used in deep Transformer for character-level language modeling. (Image source: Al-Rfou et al. (2018))
Adaptive Computation Time (ACT)

Adaptive Computation Time (short for ACT; Graves, 2016) is a mechanism for dynamically deciding how many computational steps are needed in a recurrent neural network. Here is a cool tutorial on ACT from distill.pub.

Let’s say, we have a RNN model 
𝑅
 composed of input weights 
𝑊
𝑥
, a parametric state transition function 
𝑆
(
.
)
, a set of output weights 
𝑊
𝑦
 and an output bias 
𝑏
𝑦
. Given an input sequence 
(
𝑥
1
,
…
,
𝑥
𝐿
)
, the output sequence 
(
𝑦
1
,
…
,
𝑦
𝐿
)
 is computed by:

𝑠
𝑡
=
𝑆
(
𝑠
𝑡
−
1
,
𝑊
𝑥
𝑥
𝑡
)
,
𝑦
𝑡
=
𝑊
𝑦
𝑠
𝑡
+
𝑏
𝑦
for 
𝑡
=
1
,
…
,
𝐿

ACT enables the above RNN setup to perform a variable number of steps at each input element. Multiple computational steps lead to a sequence of intermediate states 
(
𝑠
𝑡
1
,
…
,
𝑠
𝑡
𝑁
(
𝑡
)
)
 and outputs 
(
𝑦
𝑡
1
,
…
,
𝑦
𝑡
𝑁
(
𝑡
)
)
 — they all share the same state transition function 
𝑆
(
.
)
, as well as the same output weights 
𝑊
𝑦
 and bias 
𝑏
𝑦
:

	

	


	
𝑠
𝑡
0
	
=
𝑠
𝑡
−
1


𝑠
𝑡
𝑛
	
=
𝑆
(
𝑠
𝑡
𝑛
−
1
,
𝑥
𝑡
𝑛
)
=
𝑆
(
𝑠
𝑡
𝑛
−
1
,
𝑥
𝑡
+
𝛿
𝑛
,
1
)
 for 
𝑛
=
1
,
…
,
𝑁
(
𝑡
)


𝑦
𝑡
𝑛
	
=
𝑊
𝑦
𝑠
𝑡
𝑛
+
𝑏
𝑦

where 
𝛿
𝑛
,
1
 is a binary flag indicating whether the input step has been incremented.

The number of steps 
𝑁
(
𝑡
)
 is determined by an extra sigmoidal halting unit 
ℎ
, with associated weight matrix 
𝑊
ℎ
 and bias 
𝑏
ℎ
, outputting a halting probability 
𝑝
𝑡
𝑛
 at immediate step 
𝑛
 for 
𝑡
-th input element:

ℎ
𝑡
𝑛
=
𝜎
(
𝑊
ℎ
𝑠
𝑡
𝑛
+
𝑏
ℎ
)

In order to allow the computation to halt after a single step, ACT introduces a small constant 
𝜖
 (e.g. 0.01), so that whenever the cumulative probability goes above 
1
−
𝜖
, the computation stops.

	




	
	

	
𝑁
(
𝑡
)
	
=
min
(
min
{
𝑛
′
:
∑
𝑛
=
1
𝑛
′
ℎ
𝑡
𝑛
≥
1
−
𝜖
}
,
𝑀
)


𝑝
𝑡
𝑛
	
=
{
ℎ
𝑡
𝑛
	
if 
𝑛
<
𝑁
(
𝑡
)


𝑅
(
𝑡
)
=
1
−
∑
𝑛
=
1
𝑁
(
𝑡
)
−
1
ℎ
𝑡
𝑛
	
if 
𝑛
=
𝑁
(
𝑡
)

where 
𝑀
 is an upper limit for the number of immediate steps allowed.

The final state and output are mean-field updates:





𝑠
𝑡
=
∑
𝑛
=
1
𝑁
(
𝑡
)
𝑝
𝑡
𝑛
𝑠
𝑡
𝑛
,
𝑦
𝑡
=
∑
𝑛
=
1
𝑁
(
𝑡
)
𝑝
𝑡
𝑛
𝑦
𝑡
𝑛
The computation graph of a RNN with ACT mechanism. (Image source: Graves, 2016)

To avoid unnecessary pondering over each input, ACT adds a ponder cost 
𝑃
(
𝑥
)
=
∑
𝑡
=
1
𝐿
𝑁
(
𝑡
)
+
𝑅
(
𝑡
)
 in the loss function to encourage a smaller number of intermediate computational steps.

Improved Attention Span

The goal of improving attention span is to make the context that can be used in self-attention longer, more efficient and flexible.

Longer Attention Span (Transformer-XL)

The vanilla Transformer has a fixed and limited attention span. The model can only attend to other elements in the same segments during each update step and no information can flow across separated fixed-length segments.

This context segmentation causes several issues:

The model cannot capture very long term dependencies.
It is hard to predict the first few tokens in each segment given no or thin context.
The evaluation is expensive. Whenever the segment is shifted to the right by one, the new segment is re-processed from scratch, although there are a lot of overlapped tokens.

Transformer-XL (Dai et al., 2019; “XL” means “extra long”) solves the context segmentation problem with two main modifications:

Reusing hidden states between segments.
Adopting a new positional encoding that is suitable for reused states.

Hidden State Reuse

The recurrent connection between segments is introduced into the model by continuously using the hidden states from the previous segments.

A comparison between the training phrase of vanilla Transformer & Transformer-XL with a segment length 4. (Image source: left part of Figure 2 in Dai et al., 2019).

Let’s label the hidden state of the 
𝑛
-th layer for the 
(
𝜏
+
1
)
-th segment in the model as 
ℎ
𝜏
+
1
(
𝑛
)
∈
𝑅
𝐿
×
𝑑
. In addition to the hidden state of the last layer for the same segment 
ℎ
𝜏
+
1
(
𝑛
−
1
)
, it also depends on the hidden state of the same layer for the previous segment 
ℎ
𝜏
(
𝑛
)
. By incorporating information from the previous hidden states, the model extends the attention span much longer in the past, over multiple segments.

	


	


	


	


	
ℎ
~
𝜏
+
1
(
𝑛
−
1
)
	
=
[
stop-gradient
(
ℎ
𝜏
(
𝑛
−
1
)
)
∘
ℎ
𝜏
+
1
(
𝑛
−
1
)
]


𝑄
𝜏
+
1
(
𝑛
)
	
=
ℎ
𝜏
+
1
(
𝑛
−
1
)
𝑊
𝑞


𝐾
𝜏
+
1
(
𝑛
)
	
=
ℎ
~
𝜏
+
1
(
𝑛
−
1
)
𝑊
𝑘


𝑉
𝜏
+
1
(
𝑛
)
	
=
ℎ
~
𝜏
+
1
(
𝑛
−
1
)
𝑊
𝑣


ℎ
𝜏
+
1
(
𝑛
)
	
=
transformer-layer
(
𝑄
𝜏
+
1
(
𝑛
)
,
𝐾
𝜏
+
1
(
𝑛
)
,
𝑉
𝜏
+
1
(
𝑛
)
)

Note that both key and value rely on the extended hidden state, while the query only consumes hidden state at current step. The concatenation operation 
[
.
∘
.
]
 is along the sequence length dimension.

Relative Positional Encoding

In order to work with this new form of attention span, Transformer-XL proposed a new type of positional encoding. If using the same approach by vanilla Transformer and encoding the absolute position, the previous and current segments will be assigned with the same encoding, which is undesired.

To keep the positional information flow coherently across segments, Transformer-XL encodes the relative position instead, as it could be sufficient enough to know the position offset for making good predictions, i.e. 
𝑖
−
𝑗
, between one key vector 
𝑘
𝜏
,
𝑗
 and its query 
𝑞
𝜏
,
𝑖
.

If omitting the scalar 
1
/
𝑑
𝑘
 and the normalizing term in softmax but including positional encodings, we can write the attention score between query at position 
𝑖
 and key at position 
𝑗
 as:

	
	
𝑎
𝑖
𝑗
	
=
𝑞
𝑖
𝑘
𝑗
⊤
=
(
𝑥
𝑖
+
𝑝
𝑖
)
𝑊
𝑞
(
(
𝑥
𝑗
+
𝑝
𝑗
)
𝑊
𝑘
)
⊤

	
=
𝑥
𝑖
𝑊
𝑞
𝑊
𝑘
⊤
𝑥
𝑗
⊤
+
𝑥
𝑖
𝑊
𝑞
𝑊
𝑘
⊤
𝑝
𝑗
⊤
+
𝑝
𝑖
𝑊
𝑞
𝑊
𝑘
⊤
𝑥
𝑗
⊤
+
𝑝
𝑖
𝑊
𝑞
𝑊
𝑘
⊤
𝑝
𝑗
⊤

Transformer-XL reparameterizes the above four terms as follows:


				


				


				


				

𝑎
𝑖
𝑗
rel
=
𝑥
𝑖
𝑊
𝑞
𝑊
𝐸
𝑘
⊤
𝑥
𝑗
⊤
⏟
content-based addressing
+
𝑥
𝑖
𝑊
𝑞
𝑊
𝑅
𝑘
⊤
𝑟
𝑖
−
𝑗
⊤
⏟
content-dependent positional bias
+
𝑢
𝑊
𝐸
𝑘
⊤
𝑥
𝑗
⊤
⏟
global content bias
+
𝑣
𝑊
𝑅
𝑘
⊤
𝑟
𝑖
−
𝑗
⊤
⏟
global positional bias
Replace 
𝑝
𝑗
 with relative positional encoding 
𝑟
𝑖
−
𝑗
∈
𝑅
𝑑
;
Replace 
𝑝
𝑖
𝑊
𝑞
 with two trainable parameters 
𝑢
 (for content) and 
𝑣
 (for location) in two different terms;
Split 
𝑊
𝑘
 into two matrices, 
𝑊
𝐸
𝑘
 for content information and 
𝑊
𝑅
𝑘
 for location information.
Adaptive Attention Span

One key advantage of Transformer is the capability of capturing long-term dependencies. Depending on the context, the model may prefer to attend further sometime than others; or one attention head may had different attention pattern from the other. If the attention span could adapt its length flexibly and only attend further back when needed, it would help reduce both computation and memory cost to support longer maximum context size in the model.

This is the motivation for Adaptive Attention Span. Sukhbaatar, et al., (2019) proposed a self-attention mechanism that seeks an optimal attention span. They hypothesized that different attention heads might assign scores differently within the same context window (See Fig. 7) and thus the optimal span would be trained separately per head.

Two attention heads in the same model, A & B, assign attention differently within the same context window. Head A attends more to the recent tokens, while head B look further back into the past uniformly. (Image source: Sukhbaatar, et al. 2019)

Given the 
𝑖
-th token, we need to compute the attention weights between this token and other keys at positions 
𝑗
∈
𝑆
𝑖
, where 
𝑆
𝑖
 defineds the 
𝑖
-th token’s context window.

	
	

	




𝑒
𝑖
𝑗
	
=
𝑞
𝑖
𝑘
𝑗
⊤


𝑎
𝑖
𝑗
	
=
softmax
(
𝑒
𝑖
𝑗
)
=
exp
⁡
(
𝑒
𝑖
𝑗
)
∑
𝑟
=
𝑖
−
𝑠
𝑖
−
1
exp
⁡
(
𝑒
𝑖
𝑟
)


𝑦
𝑖
	
=
∑
𝑟
=
𝑖
−
𝑠
𝑖
−
1
𝑎
𝑖
𝑟
𝑣
𝑟
=
∑
𝑟
=
𝑖
−
𝑠
𝑖
−
1
𝑎
𝑖
𝑟
𝑥
𝑟
𝑊
𝑣

A soft mask function 
𝑚
𝑧
 is added to control for an effective adjustable attention span, which maps the distance between query and key into a [0, 1] value. 
𝑚
𝑧
 is parameterized by 
𝑧
∈
[
0
,
𝑠
]
 and 
𝑧
 is to be learned:

𝑚
𝑧
(
𝑥
)
=
clamp
(
1
𝑅
(
𝑅
+
𝑧
−
𝑥
)
,
0
,
1
)

where 
𝑅
 is a hyper-parameter which defines the softness of 
𝑚
𝑧
.

The soft masking function used in the adaptive attention span. (Image source: Sukhbaatar, et al. 2019.)

The soft mask function is applied to the softmax elements in the attention weights:

𝑎
𝑖
𝑗
=
𝑚
𝑧
(
𝑖
−
𝑗
)
exp
⁡
(
𝑠
𝑖
𝑗
)
∑
𝑟
=
𝑖
−
𝑠
𝑖
−
1
𝑚
𝑧
(
𝑖
−
𝑟
)
exp
⁡
(
𝑠
𝑖
𝑟
)

In the above equation, 
𝑧
 is differentiable so it is trained jointly with other parts of the model. Parameters 
𝑧
(
𝑖
)
,
𝑖
=
1
,
…
,
ℎ
 are learned separately per head. Moreover, the loss function has an extra L1 penalty on 
∑
𝑖
=
1
ℎ
𝑧
(
𝑖
)
.

Using Adaptive Computation Time, the approach can be further enhanced to have flexible attention span length, adaptive to the current input dynamically. The span parameter 
𝑧
𝑡
 of an attention head at time 
𝑡
 is a sigmoidal function, 
𝑧
𝑡
=
𝑆
𝜎
(
𝑣
⋅
𝑥
𝑡
+
𝑏
)
, where the vector 
𝑣
 and the bias scalar 
𝑏
 are learned jointly with other parameters.

In the experiments of Transformer with adaptive attention span, Sukhbaatar, et al. (2019) found a general tendency that lower layers do not require very long attention spans, while a few attention heads in higher layers may use exceptionally long spans. Adaptive attention span also helps greatly reduce the number of FLOPS, especially in a big model with many attention layers and a large context length.

Localized Attention Span (Image Transformer)

The original, also the most popular, use case for Transformer is to do language modeling. The text sequence is one-dimensional in a clearly defined chronological order and thus the attention span grows linearly with increased context size.

However, if we want to use Transformer on images, it is unclear how to define the scope of context or the order. Image Transformer (Parmer, et al 2018) embraces a formulation of image generation similar to sequence modeling within the Transformer framework. Additionally, Image Transformer restricts the self-attention span to only local neighborhoods, so that the model can scale up to process more images in parallel and keep the likelihood loss tractable.

The encoder-decoder architecture remains for image-conditioned generation:

The encoder generates a contextualized, per-pixel-channel representation of the source image;
The decoder autoregressively generates an output image, one channel per pixel at each time step.

Let’s label the representation of the current pixel to be generated as the query 
𝑞
. Other positions whose representations will be used for computing 
𝑞
 are key vector 
𝑘
1
,
𝑘
2
,
…
 and they together form a memory matrix 
𝑀
. The scope of 
𝑀
 defines the context window for pixel query 
𝑞
.

Image Transformer introduced two types of localized 
𝑀
, as illustrated below.

Illustration of 1D and 2D attention span for visual inputs in Image Transformer. The black line marks a query block and the cyan outlines the actual attention span for pixel q. (Image source: Figure 2 in Parmer et al, 2018)

(1) 1D Local Attention: The input image is flattened in the raster scanning order, that is, from left to right and top to bottom. The linearized image is then partitioned into non-overlapping query blocks. The context window consists of pixels in the same query block as 
𝑞
 and a fixed number of additional pixels generated before this query block.

(2) 2D Local Attention: The image is partitioned into multiple non-overlapping rectangular query blocks. The query pixel can attend to all others in the same memory blocks. To make sure the pixel at the top-left corner can also have a valid context window, the memory block is extended to the top, left and right by a fixed amount, respectively.

Less Time and Memory Cost

This section introduces several improvements made on Transformer to reduce the computation time and memory consumption.

Sparse Attention Matrix Factorization (Sparse Transformers)

The compute and memory cost of the vanilla Transformer grows quadratically with sequence length and thus it is hard to be applied on very long sequences.

Sparse Transformer (Child et al., 2019) introduced factorized self-attention, through sparse matrix factorization, making it possible to train dense attention networks with hundreds of layers on sequence length up to 16,384, which would be infeasible on modern hardware otherwise.

Given a set of attention connectivity pattern 
𝑆
=
{
𝑆
1
,
…
,
𝑆
𝑛
}
, where each 
𝑆
𝑖
 records a set of key positions that the 
𝑖
-th query vector attends to.

	
	
Attend
(
𝑋
,
𝑆
)
	
=
(
𝑎
(
𝑥
𝑖
,
𝑆
𝑖
)
)
𝑖
∈
{
1
,
…
,
𝐿
}


 where 
𝑎
(
𝑥
𝑖
,
𝑆
𝑖
)
	
=
softmax
(
(
𝑥
𝑖
𝑊
𝑞
)
(
𝑥
𝑗
𝑊
𝑘
)
𝑗
∈
𝑆
𝑖
⊤
𝑑
𝑘
)
(
𝑥
𝑗
𝑊
𝑣
)
𝑗
∈
𝑆
𝑖

Note that although the size of 
𝑆
𝑖
 is not fixed, 
𝑎
(
𝑥
𝑖
,
𝑆
𝑖
)
 is always of size 
𝑑
𝑣
 and thus 
Attend
(
𝑋
,
𝑆
)
∈
𝑅
𝐿
×
𝑑
𝑣
.

In anto-regressive models, one attention span is defined as 
𝑆
𝑖
=
{
𝑗
:
𝑗
≤
𝑖
}
 as it allows each token to attend to all the positions in the past.

In factorized self-attention, the set 
𝑆
𝑖
 is decomposed into a tree of dependencies, such that for every pair of 
(
𝑖
,
𝑗
)
 where 
𝑗
≤
𝑖
, there is a path connecting 
𝑖
 back to 
𝑗
 and 
𝑖
 can attend to 
𝑗
 either directly or indirectly.

Precisely, the set 
𝑆
𝑖
 is divided into 
𝑝
 non-overlapping subsets, where the 
𝑚
-th subset is denoted as 
𝐴
𝑖
(
𝑚
)
⊂
𝑆
𝑖
,
𝑚
=
1
,
…
,
𝑝
. Therefore the path between the output position 
𝑖
 and any 
𝑗
 has a maximum length 
𝑝
+
1
. For example, if 
(
𝑗
,
𝑎
,
𝑏
,
𝑐
,
…
,
𝑖
)
 is a path of indices between 
𝑖
 and 
𝑗
, we would have 
𝑗
∈
𝐴
𝑎
(
1
)
,
𝑎
∈
𝐴
𝑏
(
2
)
,
𝑏
∈
𝐴
𝑐
(
3
)
,
…
, so on and so forth.

Sparse Factorized Attention

Sparse Transformer proposed two types of fractorized attention. It is easier to understand the concepts as illustrated in Fig. 10 with 2D image inputs as examples.

The top row illustrates the attention connectivity patterns in (a) Transformer, (b) Sparse Transformer with strided attention, and (c) Sparse Transformer with fixed attention. The bottom row contains corresponding self-attention connectivity matrices. Note that the top and bottom rows are not in the same scale. (Image source: Child et al., 2019 + a few of extra annotations.)

(1) Strided attention with stride 
ℓ
∼
𝑛
. This works well with image data as the structure is aligned with strides. In the image case, each pixel would attend to all the previous 
ℓ
 pixels in the raster scanning order (naturally cover the entire width of the image) and then those pixels attend to others in the same column (defined by another attention connectivity subset).

	

	
𝐴
𝑖
(
1
)
	
=
{
𝑡
,
𝑡
+
1
,
…
,
𝑖
}
, where 
𝑡
=
max
(
0
,
𝑖
−
ℓ
)


𝐴
𝑖
(
2
)
	
=
{
𝑗
:
(
𝑖
−
𝑗
)
mod
ℓ
=
0
}

(2) Fixed attention. A small set of tokens summarize previous locations and propagate that information to all future locations.

	


	
𝐴
𝑖
(
1
)
	
=
{
𝑗
:
⌊
𝑗
ℓ
⌋
=
⌊
𝑖
ℓ
⌋
}


𝐴
𝑖
(
2
)
	
=
{
𝑗
:
𝑗
mod
ℓ
∈
{
ℓ
−
𝑐
,
…
,
ℓ
−
1
}
}

where 
𝑐
 is a hyperparameter. If 
𝑐
=
1
, it restricts the representation whereas many depend on a few positions. The paper chose 
𝑐
∈
{
8
,
16
,
32
}
 for 
ℓ
∈
{
128
,
256
}
.

Use Factorized Self-Attention in Transformer

There are three ways to use sparse factorized attention patterns in Transformer architecture:

One attention type per residual block and then interleave them,

attention
(
𝑋
)
=
Attend
(
𝑋
,
𝐴
(
𝑛
mod
𝑝
)
)
𝑊
𝑜
, where 
𝑛
 is the index of the current residual block.
Set up a single head which attends to locations that all the factorized heads attend to,

attention
(
𝑋
)
=
Attend
(
𝑋
,
∪
𝑚
=
1
𝑝
𝐴
(
𝑚
)
)
𝑊
𝑜
.
Use a multi-head attention mechanism, but different from vanilla Transformer, each head might adopt a pattern presented above, 1 or 2. => This option often performs the best.

Sparse Transformer also proposed a set of changes so as to train the Transformer up to hundreds of layers, including gradient checkpointing, recomputing attention & FF layers during the backward pass, mixed precision training, efficient block-sparse implementation, etc. Please check the paper for more details.

Locality-Sensitive Hashing (Reformer)

The improvements proposed by the Reformer model (Kitaev, et al. 2020) aim to solve the following pain points in Transformer:

Memory in a model with 
𝑁
 layers is 
𝑁
-times larger than in a single-layer model because we need to store activations for back-propagation.
The intermediate FF layers are often quite large.
The attention matrix on sequences of length 
𝐿
 often requires 
𝑂
(
𝐿
2
)
 in both memory and time.

Reformer proposed two main changes:

Replace the dot-product attention with locality-sensitive hashing (LSH) attention, reducing the complexity from 
𝑂
(
𝐿
2
)
 to 
𝑂
(
𝐿
log
⁡
𝐿
)
.
Replace the standard residual blocks with reversible residual layers, which allows storing activations only once during training instead of 
𝑁
 times (i.e. proportional to the number of layers).

Locality-Sensitive Hashing Attention

In 
𝑄
𝐾
⊤
 part of the attention formula, we are only interested in the largest elements as only large elements contribute a lot after softmax. For each query 
𝑞
𝑖
∈
𝑄
, we are looking for row vectors in 
𝐾
 closest to 
𝑞
𝑖
. In order to find nearest neighbors quickly in high-dimensional space, Reformer incorporates Locality-Sensitive Hashing (LSH) into its attention mechanism.

A hashing scheme 
𝑥
↦
ℎ
(
𝑥
)
 is locality-sensitive if it preserves the distancing information between data points, such that close vectors obtain similar hashes while distant vectors have very different ones. The Reformer adopts a hashing scheme as such, given a fixed random matrix 
𝑅
∈
𝑅
𝑑
×
𝑏
/
2
 (where 
𝑏
 is a hyperparam), the hash function is 
ℎ
(
𝑥
)
=
arg
⁡
max
(
[
𝑥
𝑅
;
−
𝑥
𝑅
]
)
.

Illustration of Locality-Sensitive Hashing (LSH) attention. (Image source: right part of Figure 1 in Kitaev, et al. 2020).

In LSH attention, a query can only attend to positions in the same hashing bucket, 
𝑆
𝑖
=
{
𝑗
:
ℎ
(
𝑞
𝑖
)
=
ℎ
(
𝑘
𝑗
)
}
. It is carried out in the following process, as illustrated in Fig. 11:

(a) The attention matrix for full attention is often sparse.
(b) Using LSH, we can sort the keys and queries to be aligned according to their hash buckets.
(c) Set 
𝑄
=
𝐾
 (precisely 
𝑘
𝑗
=
𝑞
𝑗
/
|
𝑞
𝑗
|
), so that there are equal numbers of keys and queries in one bucket, easier for batching. Interestingly, this “shared-QK” config does not affect the performance of the Transformer.
(d) Apply batching where chunks of 
𝑚
 consecutive queries are grouped together.
The LSH attention consists of 4 steps: bucketing, sorting, chunking, and attention computation. (Image source: left part of Figure 1 in Kitaev, et al. 2020).

Reversible Residual Network

Another improvement by Reformer is to use reversible residual layers (Gomez et al. 2017). The motivation for reversible residual network is to design the architecture in a way that activations at any given layer can be recovered from the activations at the following layer, using only the model parameters. Hence, we can save memory by recomputing the activation during backprop rather than storing all the activations.

Given a layer 
𝑥
↦
𝑦
, the normal residual layer does 
𝑦
=
𝑥
+
𝐹
(
𝑥
)
, but the reversible layer splits both input and output into pairs 
(
𝑥
1
,
𝑥
2
)
↦
(
𝑦
1
,
𝑦
2
)
 and then executes the following:

𝑦
1
=
𝑥
1
+
𝐹
(
𝑥
2
)
,
𝑦
2
=
𝑥
2
+
𝐺
(
𝑦
1
)

and reversing is easy:

𝑥
2
=
𝑦
2
−
𝐺
(
𝑦
1
)
,
𝑥
1
=
𝑦
1
−
𝐹
(
𝑥
2
)

Reformer applies the same idea to Transformer by combination attention (
𝐹
) and feed-forward layers (
𝐺
) within a reversible net block:

𝑌
1
=
𝑋
1
+
Attention
(
𝑋
2
)
,
𝑌
2
=
𝑋
2
+
FeedForward
(
𝑌
1
)

The memory can be further reduced by chunking the feed-forward computation:

𝑌
2
=
[
𝑌
2
(
1
)
;
…
;
𝑌
2
(
𝑐
)
]
=
[
𝑋
2
(
1
)
+
FeedForward
(
𝑌
1
(
1
)
)
;
…
;
𝑋
2
(
𝑐
)
+
FeedForward
(
𝑌
1
(
𝑐
)
)
]

The resulting reversible Transformer does not need to store activation in every layer.

Make it Recurrent (Universal Transformer)

The Universal Transformer (Dehghani, et al. 2019) combines self-attention in Transformer with the recurrent mechanism in RNN, aiming to benefit from both a long-term global receptive field of Transformer and learned inductive biases of RNN.

Rather than going through a fixed number of layers, Universal Transformer dynamically adjusts the number of steps using adaptive computation time. If we fix the number of steps, an Universal Transformer is equivalent to a multi-layer Transformer with shared parameters across layers.

On a high level, the universal transformer can be viewed as a recurrent function for learning the hidden state representation per token. The recurrent function evolves in parallel across token positions and the information between positions is shared through self-attention.

How the Universal Transformer refines a set of hidden state representations repeatedly for every position in parallel. (Image source: Figure 1 in Dehghani, et al. 2019).

Given an input sequence of length 
𝐿
, Universal Transformer iteratively updates the representation 
𝐻
𝑡
∈
𝑅
𝐿
×
𝑑
 at step 
𝑡
 for an adjustable number of steps. At step 0, 
𝐻
0
 is initialized to be same as the input embedding matrix. All the positions are processed in parallel in the multi-head self-attention mechanism and then go through a recurrent transition function.

	
	
𝐴
𝑡
	
=
LayerNorm
(
𝐻
𝑡
−
1
+
MultiHeadAttention
(
𝐻
𝑡
−
1
+
𝑃
𝑡
)


𝐻
𝑡
	
=
LayerNorm
(
𝐴
𝑡
−
1
+
Transition
(
𝐴
𝑡
)
)

where 
Transition
(
.
)
 is either a separable convolution or a fully-connected neural network that consists of two position-wise (i.e. applied to each row of 
𝐴
𝑡
 individually) affine transformation + one ReLU.

The positional encoding 
𝑃
𝑡
 uses sinusoidal position signal but with an additional time dimension:

	

	
PE
(
𝑖
,
𝑡
,
𝛿
)
=
{
sin
⁡
(
𝑖
10000
2
𝛿
′
/
𝑑
)
⊕
sin
⁡
(
𝑡
10000
2
𝛿
′
/
𝑑
)
	
if 
𝛿
=
2
𝛿
′


cos
⁡
(
𝑖
10000
2
𝛿
′
/
𝑑
)
⊕
cos
⁡
(
𝑡
10000
2
𝛿
′
/
𝑑
)
	
if 
𝛿
=
2
𝛿
′
+
1
A simplified illustration of Universal Transformer. The encoder and decoder share the same basic recurrent structure. But the decoder also attends to final encoder representation 
𝐻
𝑇
. (Image source: Figure 2 in Dehghani, et al. 2019)

In the adaptive version of Universal Transformer, the number of recurrent steps 
𝑇
 is dynamically determined by ACT. Each position is equipped with a dynamic ACT halting mechanism. Once a per-token recurrent block halts, it stops taking more recurrent updates but simply copies the current value to the next step until all the blocks halt or until the model reaches a maximum step limit.

Stabilization for RL (GTrXL)

The self-attention mechanism avoids compressing the whole past into a fixed-size hidden state and does not suffer from vanishing or exploding gradients as much as RNNs. Reinforcement Learning tasks can for sure benefit from these traits. However, it is quite difficult to train Transformer even in supervised learning, let alone in the RL context. It could be quite challenging to stabilize and train a LSTM agent by itself, after all.

The Gated Transformer-XL (GTrXL; Parisotto, et al. 2019) is one attempt to use Transformer for RL. GTrXL succeeded in stabilizing training with two changes on top of Transformer-XL:

The layer normalization is only applied on the input stream in a residual module, but NOT on the shortcut stream. A key benefit to this reordering is to allow the original input to flow from the first to last layer.
The residual connection is replaced with a GRU-style (Gated Recurrent Unit; Chung et al., 2014) gating mechanism.
	

	


	

	
𝑟
	
=
𝜎
(
𝑊
𝑟
(
𝑙
)
𝑦
+
𝑈
𝑟
(
𝑙
)
𝑥
)


𝑧
	
=
𝜎
(
𝑊
𝑧
(
𝑙
)
𝑦
+
𝑈
𝑧
(
𝑙
)
𝑥
−
𝑏
𝑔
(
𝑙
)
)


ℎ
^
	
=
tanh
⁡
(
𝑊
𝑔
(
𝑙
)
𝑦
+
𝑈
𝑔
(
𝑙
)
(
𝑟
⊙
𝑥
)
)


𝑔
(
𝑙
)
(
𝑥
,
𝑦
)
	
=
(
1
−
𝑧
)
⊙
𝑥
+
𝑧
⊙
ℎ
^

The gating function parameters are explicitly initialized to be close to an identity map - this is why there is a 
𝑏
𝑔
 term. A 
𝑏
𝑔
>
0
 greatly helps with the learning speedup.

Comparison of the model architecture of Transformer-XL, Transformer-XL with the layer norm reordered, and Gated Transformer-XL. (Image source: Figure 1 in Parisotto, et al. 2019)
Citation

Cited as:

Weng, Lilian. (Apr 2020). The transformer family. Lil’Log. https://lilianweng.github.io/posts/2020-04-07-the-transformer-family/.

Or

@article{weng2020transformer,
  title   = "The Transformer Family",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2020",
  month   = "Apr",
  url     = "https://lilianweng.github.io/posts/2020-04-07-the-transformer-family/"
}

Reference

[1] Ashish Vaswani, et al. “Attention is all you need.” NIPS 2017.

[2] Rami Al-Rfou, et al. “Character-level language modeling with deeper self-attention.” AAAI 2019.

[3] Olah & Carter, “Attention and Augmented Recurrent Neural Networks”, Distill, 2016.

[4] Sainbayar Sukhbaatar, et al. “Adaptive Attention Span in Transformers”. ACL 2019.

[5] Rewon Child, et al. “Generating Long Sequences with Sparse Transformers” arXiv:1904.10509 (2019).

[6] Nikita Kitaev, et al. “Reformer: The Efficient Transformer” ICLR 2020.

[7] Alex Graves. (“Adaptive Computation Time for Recurrent Neural Networks”)[https://arxiv.org/abs/1603.08983]

[8] Niki Parmar, et al. “Image Transformer” ICML 2018.

[9] Zihang Dai, et al. “Transformer-XL: Attentive Language Models Beyond a Fixed-Length Context.” ACL 2019.

[10] Aidan N. Gomez, et al. “The Reversible Residual Network: Backpropagation Without Storing Activations” NIPS 2017.

[11] Mostafa Dehghani, et al. “Universal Transformers” ICLR 2019.

[12] Emilio Parisotto, et al. “Stabilizing Transformers for Reinforcement Learning” arXiv:1910.06764 (2019).

Architecture
 
Attention
 
Transformer
 
Foundation
 
Reinforcement-Learning
«
Exploration Strategies in Deep Reinforcement Learning
»
Curriculum for Reinforcement Learning
© 2026 Lil'Log Powered by Hugo & PaperMod