---
title: Flow-based Deep Generative Models
url: https://lilianweng.github.io/posts/2018-10-13-flow-models/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:47:20.116655+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
Flow-based Deep Generative Models
Date: October 13, 2018 | Estimated Reading Time: 21 min | Author: Lilian Weng
Table of Contents

So far, I’ve written about two types of generative models, GAN and VAE. Neither of them explicitly learns the probability density function of real data, 
𝑝
(
𝑥
)
 (where 
𝑥
∈
𝐷
) — because it is really hard! Taking the generative model with latent variables as an example, 
𝑝
(
𝑥
)
=
∫
𝑝
(
𝑥
|
𝑧
)
𝑝
(
𝑧
)
𝑑
𝑧
 can hardly be calculated as it is intractable to go through all possible values of the latent code 
𝑧
.

Flow-based deep generative models conquer this hard problem with the help of normalizing flows, a powerful statistics tool for density estimation. A good estimation of 
𝑝
(
𝑥
)
 makes it possible to efficiently complete many downstream tasks: sample unobserved but realistic new data points (data generation), predict the rareness of future events (density estimation), infer latent variables, fill in incomplete data samples, etc.

Types of Generative Models

Here is a quick summary of the difference between GAN, VAE, and flow-based generative models:

Generative adversarial networks: GAN provides a smart solution to model the data generation, an unsupervised learning problem, as a supervised one. The discriminator model learns to distinguish the real data from the fake samples that are produced by the generator model. Two models are trained as they are playing a minimax game.
Variational autoencoders: VAE inexplicitly optimizes the log-likelihood of the data by maximizing the evidence lower bound (ELBO).
Flow-based generative models: A flow-based generative model is constructed by a sequence of invertible transformations. Unlike other two, the model explicitly learns the data distribution 
𝑝
(
𝑥
)
 and therefore the loss function is simply the negative log-likelihood.
Comparison of three categories of generative models.
Linear Algebra Basics Recap

We should understand two key concepts before getting into the flow-based generative model: the Jacobian determinant and the change of variable rule. Pretty basic, so feel free to skip.

Jacobian Matrix and Determinant

Given a function of mapping a 
𝑛
-dimensional input vector 
𝑥
 to a 
𝑚
-dimensional output vector, 
𝑓
:
𝑅
𝑛
↦
𝑅
𝑚
, the matrix of all first-order partial derivatives of this function is called the Jacobian matrix, 
𝐽
 where one entry on the i-th row and j-th column is 
𝐽
𝑖
𝑗
=
𝜕
𝑓
𝑖
𝜕
𝑥
𝑗
.

		

		

		
𝐽
=
[
𝜕
𝑓
1
𝜕
𝑥
1
	
…
	
𝜕
𝑓
1
𝜕
𝑥
𝑛


⋮
	
⋱
	
⋮


𝜕
𝑓
𝑚
𝜕
𝑥
1
	
…
	
𝜕
𝑓
𝑚
𝜕
𝑥
𝑛
]

The determinant is one real number computed as a function of all the elements in a squared matrix. Note that the determinant only exists for square matrices. The absolute value of the determinant can be thought of as a measure of “how much multiplication by the matrix expands or contracts space”.

The determinant of a nxn matrix 
𝑀
 is:

			
			
			
			


det
𝑀
=
det
[
𝑎
11
	
𝑎
12
	
…
	
𝑎
1
𝑛


𝑎
21
	
𝑎
22
	
…
	
𝑎
2
𝑛


⋮
	
⋮
		
⋮


𝑎
𝑛
1
	
𝑎
𝑛
2
	
…
	
𝑎
𝑛
𝑛
]
=
∑
𝑗
1
𝑗
2
…
𝑗
𝑛
(
−
1
)
𝜏
(
𝑗
1
𝑗
2
…
𝑗
𝑛
)
𝑎
1
𝑗
1
𝑎
2
𝑗
2
…
𝑎
𝑛
𝑗
𝑛

where the subscript under the summation 
𝑗
1
𝑗
2
…
𝑗
𝑛
 are all permutations of the set {1, 2, …, n}, so there are 
𝑛
!
 items in total; 
𝜏
(
.
)
 indicates the signature of a permutation.

The determinant of a square matrix 
𝑀
 detects whether it is invertible: If 
det
(
𝑀
)
=
0
 then 
𝑀
 is not invertible (a singular matrix with linearly dependent rows or columns; or any row or column is all 0); otherwise, if 
det
(
𝑀
)
≠
0
, 
𝑀
 is invertible.

The determinant of the product is equivalent to the product of the determinants: 
det
(
𝐴
𝐵
)
=
det
(
𝐴
)
det
(
𝐵
)
. (proof)

Change of Variable Theorem

Let’s review the change of variable theorem specifically in the context of probability density estimation, starting with a single variable case.

Given a random variable 
𝑧
 and its known probability density function 
𝑧
∼
𝜋
(
𝑧
)
, we would like to construct a new random variable using a 1-1 mapping function 
𝑥
=
𝑓
(
𝑧
)
. The function 
𝑓
 is invertible, so 
𝑧
=
𝑓
−
1
(
𝑥
)
. Now the question is how to infer the unknown probability density function of the new variable, 
𝑝
(
𝑥
)
?

	
	
	
∫
𝑝
(
𝑥
)
𝑑
𝑥
=
∫
𝜋
(
𝑧
)
𝑑
𝑧
=
1
 ; Definition of probability distribution.

	
𝑝
(
𝑥
)
=
𝜋
(
𝑧
)
|
𝑑
𝑧
𝑑
𝑥
|
=
𝜋
(
𝑓
−
1
(
𝑥
)
)
|
𝑑
𝑓
−
1
𝑑
𝑥
|
=
𝜋
(
𝑓
−
1
(
𝑥
)
)
|
(
𝑓
−
1
)
′
(
𝑥
)
|

By definition, the integral 
∫
𝜋
(
𝑧
)
𝑑
𝑧
 is the sum of an infinite number of rectangles of infinitesimal width 
Δ
𝑧
. The height of such a rectangle at position 
𝑧
 is the value of the density function 
𝜋
(
𝑧
)
. When we substitute the variable, 
𝑧
=
𝑓
−
1
(
𝑥
)
 yields 
Δ
𝑧
Δ
𝑥
=
(
𝑓
−
1
(
𝑥
)
)
′
 and 
Δ
𝑧
=
(
𝑓
−
1
(
𝑥
)
)
′
Δ
𝑥
. Here 
|
(
𝑓
−
1
(
𝑥
)
)
′
|
 indicates the ratio between the area of rectangles defined in two different coordinate of variables 
𝑧
 and 
𝑥
 respectively.

The multivariable version has a similar format:

	
	
𝑧
	
∼
𝜋
(
𝑧
)
,
𝑥
=
𝑓
(
𝑧
)
,
𝑧
=
𝑓
−
1
(
𝑥
)


𝑝
(
𝑥
)
	
=
𝜋
(
𝑧
)
|
det
𝑑
𝑧
𝑑
𝑥
|
=
𝜋
(
𝑓
−
1
(
𝑥
)
)
|
det
𝑑
𝑓
−
1
𝑑
𝑥
|

where 
det
𝜕
𝑓
𝜕
𝑧
 is the Jacobian determinant of the function 
𝑓
. The full proof of the multivariate version is out of the scope of this post; ask Google if interested ;)

What is Normalizing Flows?

Being able to do good density estimation has direct applications in many machine learning problems, but it is very hard. For example, since we need to run backward propagation in deep learning models, the embedded probability distribution (i.e. posterior 
𝑝
(
𝑧
|
𝑥
)
) is expected to be simple enough to calculate the derivative easily and efficiently. That is why Gaussian distribution is often used in latent variable generative models, even though most of real world distributions are much more complicated than Gaussian.

Here comes a Normalizing Flow (NF) model for better and more powerful distribution approximation. A normalizing flow transforms a simple distribution into a complex one by applying a sequence of invertible transformation functions. Flowing through a chain of transformations, we repeatedly substitute the variable for the new one according to the change of variables theorem and eventually obtain a probability distribution of the final target variable.

Illustration of a normalizing flow model, transforming a simple distribution 
𝑝
_
0
(
𝑧
_
0
)
 to a complex one 
𝑝
_
𝐾
(
𝑧
_
𝐾
)
 step by step.

As defined in Fig. 2,

	
	

	
𝑧
𝑖
−
1
	
∼
𝑝
𝑖
−
1
(
𝑧
𝑖
−
1
)


𝑧
𝑖
	
=
𝑓
𝑖
(
𝑧
𝑖
−
1
)
, thus 
𝑧
𝑖
−
1
=
𝑓
𝑖
−
1
(
𝑧
𝑖
)


𝑝
𝑖
(
𝑧
𝑖
)
	
=
𝑝
𝑖
−
1
(
𝑓
𝑖
−
1
(
𝑧
𝑖
)
)
|
det
𝑑
𝑓
𝑖
−
1
𝑑
𝑧
𝑖
|

Then let’s convert the equation to be a function of 
𝑧
𝑖
 so that we can do inference with the base distribution.

	
	
	
	
	
	
	
	
𝑝
𝑖
(
𝑧
𝑖
)
	
=
𝑝
𝑖
−
1
(
𝑓
𝑖
−
1
(
𝑧
𝑖
)
)
|
det
𝑑
𝑓
𝑖
−
1
𝑑
𝑧
𝑖
|

	
=
𝑝
𝑖
−
1
(
𝑧
𝑖
−
1
)
|
det
(
𝑑
𝑓
𝑖
𝑑
𝑧
𝑖
−
1
)
−
1
|
	
; According to the inverse func theorem.

	
=
𝑝
𝑖
−
1
(
𝑧
𝑖
−
1
)
|
det
𝑑
𝑓
𝑖
𝑑
𝑧
𝑖
−
1
|
−
1
	
; According to a property of Jacobians of invertible func.


log
⁡
𝑝
𝑖
(
𝑧
𝑖
)
	
=
log
⁡
𝑝
𝑖
−
1
(
𝑧
𝑖
−
1
)
−
log
⁡
|
det
𝑑
𝑓
𝑖
𝑑
𝑧
𝑖
−
1
|

(*) A note on the “inverse function theorem”: If 
𝑦
=
𝑓
(
𝑥
)
 and 
𝑥
=
𝑓
−
1
(
𝑦
)
, we have:

𝑑
𝑓
−
1
(
𝑦
)
𝑑
𝑦
=
𝑑
𝑥
𝑑
𝑦
=
(
𝑑
𝑦
𝑑
𝑥
)
−
1
=
(
𝑑
𝑓
(
𝑥
)
𝑑
𝑥
)
−
1

(*) A note on “Jacobians of invertible function”: The determinant of the inverse of an invertible matrix is the inverse of the determinant: 
det
(
𝑀
−
1
)
=
(
det
(
𝑀
)
)
−
1
, because 
det
(
𝑀
)
det
(
𝑀
−
1
)
=
det
(
𝑀
⋅
𝑀
−
1
)
=
det
(
𝐼
)
=
1
.

Given such a chain of probability density functions, we know the relationship between each pair of consecutive variables. We can expand the equation of the output 
𝑥
 step by step until tracing back to the initial distribution 
𝑧
0
.

	
	

	

	
	


𝑥
=
𝑧
𝐾
	
=
𝑓
𝐾
∘
𝑓
𝐾
−
1
∘
⋯
∘
𝑓
1
(
𝑧
0
)


log
⁡
𝑝
(
𝑥
)
=
log
⁡
𝜋
𝐾
(
𝑧
𝐾
)
	
=
log
⁡
𝜋
𝐾
−
1
(
𝑧
𝐾
−
1
)
−
log
⁡
|
det
𝑑
𝑓
𝐾
𝑑
𝑧
𝐾
−
1
|

	
=
log
⁡
𝜋
𝐾
−
2
(
𝑧
𝐾
−
2
)
−
log
⁡
|
det
𝑑
𝑓
𝐾
−
1
𝑑
𝑧
𝐾
−
2
|
−
log
⁡
|
det
𝑑
𝑓
𝐾
𝑑
𝑧
𝐾
−
1
|

	
=
…

	
=
log
⁡
𝜋
0
(
𝑧
0
)
−
∑
𝑖
=
1
𝐾
log
⁡
|
det
𝑑
𝑓
𝑖
𝑑
𝑧
𝑖
−
1
|

The path traversed by the random variables 
𝑧
𝑖
=
𝑓
𝑖
(
𝑧
𝑖
−
1
)
 is the flow and the full chain formed by the successive distributions 
𝜋
𝑖
 is called a normalizing flow. Required by the computation in the equation, a transformation function 
𝑓
𝑖
 should satisfy two properties:

It is easily invertible.
Its Jacobian determinant is easy to compute.
Models with Normalizing Flows

With normalizing flows in our toolbox, the exact log-likelihood of input data 
log
⁡
𝑝
(
𝑥
)
 becomes tractable. As a result, the training criterion of flow-based generative model is simply the negative log-likelihood (NLL) over the training dataset 
𝐷
:



𝐿
(
𝐷
)
=
−
1
|
𝐷
|
∑
𝑥
∈
𝐷
log
⁡
𝑝
(
𝑥
)
RealNVP

The RealNVP (Real-valued Non-Volume Preserving; Dinh et al., 2017) model implements a normalizing flow by stacking a sequence of invertible bijective transformation functions. In each bijection 
𝑓
:
𝑥
↦
𝑦
, known as affine coupling layer, the input dimensions are split into two parts:

The first 
𝑑
 dimensions stay same;
The second part, 
𝑑
+
1
 to 
𝐷
 dimensions, undergo an affine transformation (“scale-and-shift”) and both the scale and shift parameters are functions of the first 
𝑑
 dimensions.
	
	
𝑦
1
:
𝑑
	
=
𝑥
1
:
𝑑


𝑦
𝑑
+
1
:
𝐷
	
=
𝑥
𝑑
+
1
:
𝐷
⊙
exp
⁡
(
𝑠
(
𝑥
1
:
𝑑
)
)
+
𝑡
(
𝑥
1
:
𝑑
)

where 
𝑠
(
.
)
 and 
𝑡
(
.
)
 are scale and translation functions and both map 
𝑅
𝑑
↦
𝑅
𝐷
−
𝑑
. The 
⊙
 operation is the element-wise product.

Now let’s check whether this transformation satisfy two basic properties for a flow transformation.

Condition 1: “It is easily invertible.”

Yes and it is fairly straightforward.

	
		
	
{
𝑦
1
:
𝑑
	
=
𝑥
1
:
𝑑


𝑦
𝑑
+
1
:
𝐷
	
=
𝑥
𝑑
+
1
:
𝐷
⊙
exp
⁡
(
𝑠
(
𝑥
1
:
𝑑
)
)
+
𝑡
(
𝑥
1
:
𝑑
)
⇔
{
𝑥
1
:
𝑑
	
=
𝑦
1
:
𝑑


𝑥
𝑑
+
1
:
𝐷
	
=
(
𝑦
𝑑
+
1
:
𝐷
−
𝑡
(
𝑦
1
:
𝑑
)
)
⊙
exp
⁡
(
−
𝑠
(
𝑦
1
:
𝑑
)
)

Condition 2: “Its Jacobian determinant is easy to compute.”

Yes. It is not hard to get the Jacobian matrix and determinant of this transformation. The Jacobian is a lower triangular matrix.

	

	
𝐽
=
[
𝐼
𝑑
	
0
𝑑
×
(
𝐷
−
𝑑
)


𝜕
𝑦
𝑑
+
1
:
𝐷
𝜕
𝑥
1
:
𝑑
	
diag
(
exp
⁡
(
𝑠
(
𝑥
1
:
𝑑
)
)
)
]

Hence the determinant is simply the product of terms on the diagonal.





det
(
𝐽
)
=
∏
𝑗
=
1
𝐷
−
𝑑
exp
⁡
(
𝑠
(
𝑥
1
:
𝑑
)
)
𝑗
=
exp
⁡
(
∑
𝑗
=
1
𝐷
−
𝑑
𝑠
(
𝑥
1
:
𝑑
)
𝑗
)

So far, the affine coupling layer looks perfect for constructing a normalizing flow :)

Even better, since (i) computing 
𝑓
−
1
 does not require computing the inverse of 
𝑠
 or 
𝑡
 and (ii) computing the Jacobian determinant does not involve computing the Jacobian of 
𝑠
 or 
𝑡
, those functions can be arbitrarily complex; i.e. both 
𝑠
 and 
𝑡
 can be modeled by deep neural networks.

In one affine coupling layer, some dimensions (channels) remain unchanged. To make sure all the inputs have a chance to be altered, the model reverses the ordering in each layer so that different components are left unchanged. Following such an alternating pattern, the set of units which remain identical in one transformation layer are always modified in the next. Batch normalization is found to help training models with a very deep stack of coupling layers.

Furthermore, RealNVP can work in a multi-scale architecture to build a more efficient model for large inputs. The multi-scale architecture applies several “sampling” operations to normal affine layers, including spatial checkerboard pattern masking, squeezing operation, and channel-wise masking. Read the paper for more details on the multi-scale architecture.

NICE

The NICE (Non-linear Independent Component Estimation; Dinh, et al. 2015) model is a predecessor of RealNVP. The transformation in NICE is the affine coupling layer without the scale term, known as additive coupling layer.

	
		
	
{
𝑦
1
:
𝑑
	
=
𝑥
1
:
𝑑


𝑦
𝑑
+
1
:
𝐷
	
=
𝑥
𝑑
+
1
:
𝐷
+
𝑚
(
𝑥
1
:
𝑑
)
⇔
{
𝑥
1
:
𝑑
	
=
𝑦
1
:
𝑑


𝑥
𝑑
+
1
:
𝐷
	
=
𝑦
𝑑
+
1
:
𝐷
−
𝑚
(
𝑦
1
:
𝑑
)
Glow

The Glow (Kingma and Dhariwal, 2018) model extends the previous reversible generative models, NICE and RealNVP, and simplifies the architecture by replacing the reverse permutation operation on the channel ordering with invertible 1x1 convolutions.

One step of flow in the Glow model. (Image source: Kingma and Dhariwal, 2018)

There are three substeps in one step of flow in Glow.

Substep 1: Activation normalization (short for “actnorm”)

It performs an affine transformation using a scale and bias parameter per channel, similar to batch normalization, but works for mini-batch size 1. The parameters are trainable but initialized so that the first minibatch of data have mean 0 and standard deviation 1 after actnorm.

Substep 2: Invertible 1x1 conv

Between layers of the RealNVP flow, the ordering of channels is reversed so that all the data dimensions have a chance to be altered. A 1×1 convolution with equal number of input and output channels is a generalization of any permutation of the channel ordering.

Say, we have an invertible 1x1 convolution of an input 
ℎ
×
𝑤
×
𝑐
 tensor 
ℎ
 with a weight matrix 
𝑊
 of size 
𝑐
×
𝑐
. The output is a 
ℎ
×
𝑤
×
𝑐
 tensor, labeled as 
𝑓
=
conv2d
(
ℎ
;
𝑊
)
. In order to apply the change of variable rule, we need to compute the Jacobian determinant 
|
det
𝜕
𝑓
/
𝜕
ℎ
|
.

Both the input and output of 1x1 convolution here can be viewed as a matrix of size 
ℎ
×
𝑤
. Each entry 
𝑥
𝑖
𝑗
 (
𝑖
=
1
,
…
,
ℎ
,
𝑗
=
1
,
…
,
𝑤
) in 
ℎ
 is a vector of 
𝑐
 channels and each entry is multiplied by the weight matrix 
𝑊
 to obtain the corresponding entry 
𝑦
𝑖
𝑗
 in the output matrix respectively. The derivative of each entry is 
𝜕
𝑥
𝑖
𝑗
𝑊
/
𝜕
𝑥
𝑖
𝑗
=
𝑊
 and there are 
ℎ
×
𝑤
 such entries in total:

log
⁡
|
det
𝜕
conv2d
(
ℎ
;
𝑊
)
𝜕
ℎ
|
=
log
⁡
(
|
det
𝑊
|
ℎ
⋅
𝑤
|
)
=
ℎ
⋅
𝑤
⋅
log
⁡
|
det
𝑊
|

The inverse 1x1 convolution depends on the inverse matrix 
𝑊
−
1
. Since the weight matrix is relatively small, the amount of computation for the matrix determinant (tf.linalg.det) and inversion (tf.linalg.inv) is still under control.

Substep 3: Affine coupling layer

The design is same as in RealNVP.

Three substeps in one step of flow in Glow. (Image source: Kingma and Dhariwal, 2018)
Models with Autoregressive Flows

The autoregressive constraint is a way to model sequential data, 
𝑥
=
[
𝑥
1
,
…
,
𝑥
𝐷
]
: each output only depends on the data observed in the past, but not on the future ones. In other words, the probability of observing 
𝑥
𝑖
 is conditioned on 
𝑥
1
,
…
,
𝑥
𝑖
−
1
 and the product of these conditional probabilities gives us the probability of observing the full sequence:





𝑝
(
𝑥
)
=
∏
𝑖
=
1
𝐷
𝑝
(
𝑥
𝑖
|
𝑥
1
,
…
,
𝑥
𝑖
−
1
)
=
∏
𝑖
=
1
𝐷
𝑝
(
𝑥
𝑖
|
𝑥
1
:
𝑖
−
1
)

How to model the conditional density is of your choice. It can be a univariate Gaussian with mean and standard deviation computed as a function of 
𝑥
1
:
𝑖
−
1
, or a multilayer neural network with 
𝑥
1
:
𝑖
−
1
 as the input.

If a flow transformation in a normalizing flow is framed as an autoregressive model — each dimension in a vector variable is conditioned on the previous dimensions — this is an autoregressive flow.

This section starts with several classic autoregressive models (MADE, PixelRNN, WaveNet) and then we dive into autoregressive flow models (MAF and IAF).

MADE

MADE (Masked Autoencoder for Distribution Estimation; Germain et al., 2015) is a specially designed architecture to enforce the autoregressive property in the autoencoder efficiently. When using an autoencoder to predict the conditional probabilities, rather than feeding the autoencoder with input of different observation windows 
𝐷
 times, MADE removes the contribution from certain hidden units by multiplying binary mask matrices so that each input dimension is reconstructed only from previous dimensions in a given ordering in a single pass.

In a multilayer fully-connected neural network, say, we have 
𝐿
 hidden layers with weight matrices 
𝑊
1
,
…
,
𝑊
𝐿
 and an output layer with weight matrix 
𝑉
. The output 
𝑥
^
 has each dimension 
𝑥
^
𝑖
=
𝑝
(
𝑥
𝑖
|
𝑥
1
:
𝑖
−
1
)
.

Without any mask, the computation through layers looks like the following:

	
	

	
ℎ
0
	
=
𝑥


ℎ
𝑙
	
=
activation
𝑙
(
𝑊
𝑙
ℎ
𝑙
−
1
+
𝑏
𝑙
)


𝑥
^
	
=
𝜎
(
𝑉
ℎ
𝐿
+
𝑐
)
Demonstration of how MADE works in a three-layer feed-forward neural network. (Image source: Germain et al., 2015)

To zero out some connections between layers, we can simply element-wise multiply every weight matrix by a binary mask matrix. Each hidden node is assigned with a random “connectivity integer” between 
1
 and 
𝐷
−
1
; the assigned value for the 
𝑘
-th unit in the 
𝑙
-th layer is denoted by 
𝑚
𝑘
𝑙
. The binary mask matrix is determined by element-wise comparing values of two nodes in two layers.

	

	

	
	

	

	
	

	
ℎ
𝑙
	
=
activation
𝑙
(
(
𝑊
𝑙
⊙
𝑀
𝑊
𝑙
)
ℎ
𝑙
−
1
+
𝑏
𝑙
)


𝑥
^
	
=
𝜎
(
(
𝑉
⊙
𝑀
𝑉
)
ℎ
𝐿
+
𝑐
)


𝑀
𝑘
′
,
𝑘
𝑊
𝑙
	
=
1
𝑚
𝑘
′
𝑙
≥
𝑚
𝑘
𝑙
−
1
=
{
1
,
	
if 
𝑚
𝑘
′
𝑙
≥
𝑚
𝑘
𝑙
−
1


0
,
	
otherwise


𝑀
𝑑
,
𝑘
𝑉
	
=
1
𝑑
≥
𝑚
𝑘
𝐿
=
{
1
,
	
if 
𝑑
>
𝑚
𝑘
𝐿


0
,
	
otherwise

A unit in the current layer can only be connected to other units with equal or smaller numbers in the previous layer and this type of dependency easily propagates through the network up to the output layer. Once the numbers are assigned to all the units and layers, the ordering of input dimensions is fixed and the conditional probability is produced with respect to it. See a great illustration in To make sure all the hidden units are connected to the input and output layers through some paths, the 
𝑚
𝑘
𝑙
 is sampled to be equal or greater than the minimal connectivity integer in the previous layer, 
min
𝑘
′
𝑚
𝑘
′
𝑙
−
1
.

MADE training can be further facilitated by:

Order-agnostic training: shuffle the input dimensions, so that MADE is able to model any arbitrary ordering; can create an ensemble of autoregressive models at the runtime.
Connectivity-agnostic training: to avoid a model being tied up to a specific connectivity pattern constraints, resample 
𝑚
𝑘
𝑙
 for each training minibatch.
PixelRNN

PixelRNN (Oord et al, 2016) is a deep generative model for images. The image is generated one pixel at a time and each new pixel is sampled conditional on the pixels that have been seen before.

Let’s consider an image of size 
𝑛
×
𝑛
, 
𝑥
=
{
𝑥
1
,
…
,
𝑥
𝑛
2
}
, the model starts generating pixels from the top left corner, from left to right and top to bottom (See Fig. 6).

The context for generating one pixel in PixelRNN. (Image source: Oord et al, 2016)

Every pixel 
𝑥
𝑖
 is sampled from a probability distribution conditional over the the past context: pixels above it or on the left of it when in the same row. The definition of such context looks pretty arbitrary, because how visual attention is attended to an image is more flexible. Somehow magically a generative model with such a strong assumption works.

One implementation that could capture the entire context is the Diagonal BiLSTM. First, apply the skewing operation by offsetting each row of the input feature map by one position with respect to the previous row, so that computation for each row can be parallelized. Then the LSTM states are computed with respect to the current pixel and the pixels on the left.

(a) PixelRNN with diagonal BiLSTM. (b) Skewing operation that offsets each row in the feature map by one with regards to the row above. (Image source: Oord et al, 2016)
		
		
		
[
𝑜
𝑖
,
𝑓
𝑖
,
𝑖
𝑖
,
𝑔
𝑖
]
	
=
𝜎
(
𝐾
𝑠
𝑠
⊛
ℎ
𝑖
−
1
+
𝐾
𝑖
𝑠
⊛
𝑥
𝑖
)
	
; 
𝜎
 is tanh for g, but otherwise sigmoid; 
⊛
 is convolution operation.


𝑐
𝑖
	
=
𝑓
𝑖
⊙
𝑐
𝑖
−
1
+
𝑖
𝑖
⊙
𝑔
𝑖
	
; 
⊙
 is elementwise product.


ℎ
𝑖
	
=
𝑜
𝑖
⊙
tanh
⁡
(
𝑐
𝑖
)

where 
⊛
 denotes the convolution operation and 
⊙
 is the element-wise multiplication. The input-to-state component 
𝐾
𝑖
𝑠
 is a 1x1 convolution, while the state-to-state recurrent component is computed with a column-wise convolution 
𝐾
𝑠
𝑠
 with a kernel of size 2x1.

The diagonal BiLSTM layers are capable of processing an unbounded context field, but expensive to compute due to the sequential dependency between states. A faster implementation uses multiple convolutional layers without pooling to define a bounded context box. The convolution kernel is masked so that the future context is not seen, similar to MADE. This convolution version is called PixelCNN.

PixelCNN with masked convolution constructed by an elementwise product of a mask tensor and the convolution kernel before applying it. (Image source: http://slazebni.cs.illinois.edu/spring17/lec13_advanced.pdf)
WaveNet

WaveNet (Van Den Oord, et al. 2016) is very similar to PixelCNN but applied to 1-D audio signals. WaveNet consists of a stack of causal convolution which is a convolution operation designed to respect the ordering: the prediction at a certain timestamp can only consume the data observed in the past, no dependency on the future. In PixelCNN, the causal convolution is implemented by masked convolution kernel. The causal convolution in WaveNet is simply to shift the output by a number of timestamps to the future so that the output is aligned with the last input element.

One big drawback of convolution layer is a very limited size of receptive field. The output can hardly depend on the input hundreds or thousands of timesteps ago, which can be a crucial requirement for modeling long sequences. WaveNet therefore adopts dilated convolution (animation), where the kernel is applied to an evenly-distributed subset of samples in a much larger receptive field of the input.

Visualization of WaveNet models with a stack of (top) causal convolution layers and (bottom) dilated convolution layers. (Image source: Van Den Oord, et al. 2016)

WaveNet uses the gated activation unit as the non-linear layer, as it is found to work significantly better than ReLU for modeling 1-D audio data. The residual connection is applied after the gated activation.

𝑧
=
tanh
⁡
(
𝑊
𝑓
,
𝑘
⊛
𝑥
)
⊙
𝜎
(
𝑊
𝑔
,
𝑘
⊛
𝑥
)

where 
𝑊
𝑓
,
𝑘
 and 
𝑊
𝑔
,
𝑘
 are convolution filter and gate weight matrix of the 
𝑘
-th layer, respectively; both are learnable.

Masked Autoregressive Flow

Masked Autoregressive Flow (MAF; Papamakarios et al., 2017) is a type of normalizing flows, where the transformation layer is built as an autoregressive neural network. MAF is very similar to Inverse Autoregressive Flow (IAF) introduced later. See more discussion on the relationship between MAF and IAF in the next section.

Given two random variables, 
𝑧
∼
𝜋
(
𝑧
)
 and 
𝑥
∼
𝑝
(
𝑥
)
 and the probability density function 
𝜋
(
𝑧
)
 is known, MAF aims to learn 
𝑝
(
𝑥
)
. MAF generates each 
𝑥
𝑖
 conditioned on the past dimensions 
𝑥
1
:
𝑖
−
1
.

Precisely the conditional probability is an affine transformation of 
𝑧
, where the scale and shift terms are functions of the observed part of 
𝑥
.

Data generation, producing a new 
𝑥
:

𝑥
𝑖
∼
𝑝
(
𝑥
𝑖
|
𝑥
1
:
𝑖
−
1
)
=
𝑧
𝑖
⊙
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)
+
𝜇
𝑖
(
𝑥
1
:
𝑖
−
1
)
, where 
𝑧
∼
𝜋
(
𝑧
)

Density estimation, given a known 
𝑥
:

𝑝
(
𝑥
)
=
∏
𝑖
=
1
𝐷
𝑝
(
𝑥
𝑖
|
𝑥
1
:
𝑖
−
1
)

The generation procedure is sequential, so it is slow by design. While density estimation only needs one pass the network using architecture like MADE. The transformation function is trivial to inverse and the Jacobian determinant is easy to compute too.

Inverse Autoregressive Flow

Similar to MAF, Inverse autoregressive flow (IAF; Kingma et al., 2016) models the conditional probability of the target variable as an autoregressive model too, but with a reversed flow, thus achieving a much efficient sampling process.

First, let’s reverse the affine transformation in MAF:

𝑧
𝑖
=
𝑥
𝑖
−
𝜇
𝑖
(
𝑥
1
:
𝑖
−
1
)
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)
=
−
𝜇
𝑖
(
𝑥
1
:
𝑖
−
1
)
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)
+
𝑥
𝑖
⊙
1
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)

If let:

	

	

	

	
	
𝑥
~
=
𝑧
, 
𝑝
~
(
.
)
=
𝜋
(
.
)
, 
𝑥
~
∼
𝑝
~
(
𝑥
~
)

	
𝑧
~
=
𝑥
, 
𝜋
~
(
.
)
=
𝑝
(
.
)
, 
𝑧
~
∼
𝜋
~
(
𝑧
~
)

	
𝜇
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
=
𝜇
~
𝑖
(
𝑥
1
:
𝑖
−
1
)
=
−
𝜇
𝑖
(
𝑥
1
:
𝑖
−
1
)
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)

	
𝜎
~
(
𝑧
~
1
:
𝑖
−
1
)
=
𝜎
~
(
𝑥
1
:
𝑖
−
1
)
=
1
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)

Then we would have,

𝑥
~
𝑖
∼
𝑝
(
𝑥
~
𝑖
|
𝑧
~
1
:
𝑖
)
=
𝑧
~
𝑖
⊙
𝜎
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
+
𝜇
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
, where 
𝑧
~
∼
𝜋
~
(
𝑧
~
)

IAF intends to estimate the probability density function of 
𝑥
~
 given that 
𝜋
~
(
𝑧
~
)
 is already known. The inverse flow is an autoregressive affine transformation too, same as in MAF, but the scale and shift terms are autoregressive functions of observed variables from the known distribution 
𝜋
~
(
𝑧
~
)
. See the comparison between MAF and IAF in

Comparison of MAF and IAF. The variable with known density is in green while the unknown one is in red.

Computations of the individual elements 
𝑥
~
𝑖
 do not depend on each other, so they are easily parallelizable (only one pass using MADE). The density estimation for a known 
𝑥
~
 is not efficient, because we have to recover the value of 
𝑧
~
𝑖
 in a sequential order, 
𝑧
~
𝑖
=
(
𝑥
~
𝑖
−
𝜇
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
)
/
𝜎
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
, thus D times in total.

	Base distribution	Target distribution	Model	Data generation	Density estimation
MAF	
𝑧
∼
𝜋
(
𝑧
)
	
𝑥
∼
𝑝
(
𝑥
)
	
𝑥
𝑖
=
𝑧
𝑖
⊙
𝜎
𝑖
(
𝑥
1
:
𝑖
−
1
)
+
𝜇
𝑖
(
𝑥
1
:
𝑖
−
1
)
	Sequential; slow	One pass; fast
IAF	
𝑧
~
∼
𝜋
~
(
𝑧
~
)
	
𝑥
~
∼
𝑝
~
(
𝑥
~
)
	
𝑥
~
𝑖
=
𝑧
~
𝑖
⊙
𝜎
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
+
𝜇
~
𝑖
(
𝑧
~
1
:
𝑖
−
1
)
	One pass; fast	Sequential; slow
———-	———-	———-	———-	———-	———-
VAE + Flows

In Variational Autoencoder, if we want to model the posterior 
𝑝
(
𝑧
|
𝑥
)
 as a more complicated distribution rather than simple Gaussian. Intuitively we can use normalizing flow to transform the base Gaussian for better density approximation. The encoder then would predict a set of scale and shift terms 
(
𝜇
𝑖
,
𝜎
𝑖
)
 which are all functions of input 
𝑥
. Read the paper for more details if interested.

If you notice mistakes and errors in this post, don’t hesitate to contact me at [lilian dot wengweng at gmail dot com] and I would be very happy to correct them right away!

See you in the next post :D

Cited as:

@article{weng2018flow,
  title   = "Flow-based Deep Generative Models",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2018",
  url     = "https://lilianweng.github.io/posts/2018-10-13-flow-models/"
}

Reference

[1] Danilo Jimenez Rezende, and Shakir Mohamed. “Variational inference with normalizing flows.” ICML 2015.

[2] Normalizing Flows Tutorial, Part 1: Distributions and Determinants by Eric Jang.

[3] Normalizing Flows Tutorial, Part 2: Modern Normalizing Flows by Eric Jang.

[4] Normalizing Flows by Adam Kosiorek.

[5] Laurent Dinh, Jascha Sohl-Dickstein, and Samy Bengio. “Density estimation using Real NVP.” ICLR 2017.

[6] Laurent Dinh, David Krueger, and Yoshua Bengio. “NICE: Non-linear independent components estimation.” ICLR 2015 Workshop track.

[7] Diederik P. Kingma, and Prafulla Dhariwal. “Glow: Generative flow with invertible 1x1 convolutions.” arXiv:1807.03039 (2018).

[8] Germain, Mathieu, Karol Gregor, Iain Murray, and Hugo Larochelle. “Made: Masked autoencoder for distribution estimation.” ICML 2015.

[9] Aaron van den Oord, Nal Kalchbrenner, and Koray Kavukcuoglu. “Pixel recurrent neural networks.” ICML 2016.

[10] Diederik P. Kingma, et al. “Improved variational inference with inverse autoregressive flow.” NIPS. 2016.

[11] George Papamakarios, Iain Murray, and Theo Pavlakou. “Masked autoregressive flow for density estimation.” NIPS 2017.

[12] Jianlin Su, and Guang Wu. “f-VAEs: Improve VAEs with Conditional Flows.” arXiv:1809.05861 (2018).

[13] Van Den Oord, Aaron, et al. “WaveNet: A generative model for raw audio.” SSW. 2016.

Architecture
 
Generative-Model
 
Image-Generation
 
Math-Heavy
«
Meta-Learning: Learning to Learn Fast
»
From Autoencoder to Beta-VAE
© 2026 Lil'Log Powered by Hugo & PaperMod