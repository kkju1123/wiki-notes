---
title: Contrastive Representation Learning
url: https://lilianweng.github.io/posts/2021-05-31-contrastive/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:45:25.603080+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
Contrastive Representation Learning
Date: May 31, 2021 | Estimated Reading Time: 39 min | Author: Lilian Weng
Table of Contents

The goal of contrastive representation learning is to learn such an embedding space in which similar sample pairs stay close to each other while dissimilar ones are far apart. Contrastive learning can be applied to both supervised and unsupervised settings. When working with unsupervised data, contrastive learning is one of the most powerful approaches in self-supervised learning.

Contrastive Training Objectives

In early versions of loss functions for contrastive learning, only one positive and one negative sample are involved. The trend in recent training objectives is to include multiple positive and negative pairs in one batch.

Contrastive Loss

Contrastive loss (Chopra et al. 2005) is one of the earliest training objectives used for deep metric learning in a contrastive fashion.

Given a list of input samples 
{
𝑥
𝑖
}
, each has a corresponding label 
𝑦
𝑖
∈
{
1
,
…
,
𝐿
}
 among 
𝐿
 classes. We would like to learn a function 
𝑓
𝜃
(
.
)
:
𝑋
→
𝑅
𝑑
 that encodes 
𝑥
𝑖
 into an embedding vector such that examples from the same class have similar embeddings and samples from different classes have very different ones. Thus, contrastive loss takes a pair of inputs 
(
𝑥
𝑖
,
𝑥
𝑗
)
 and minimizes the embedding distance when they are from the same class but maximizes the distance otherwise.

𝟙
𝟙
𝐿
cont
(
𝑥
𝑖
,
𝑥
𝑗
,
𝜃
)
=
1
[
𝑦
𝑖
=
𝑦
𝑗
]
‖
𝑓
𝜃
(
𝑥
𝑖
)
−
𝑓
𝜃
(
𝑥
𝑗
)
‖
2
2
+
1
[
𝑦
𝑖
≠
𝑦
𝑗
]
max
(
0
,
𝜖
−
‖
𝑓
𝜃
(
𝑥
𝑖
)
−
𝑓
𝜃
(
𝑥
𝑗
)
‖
2
)
2

where 
𝜖
 is a hyperparameter, defining the lower bound distance between samples of different classes.

Triplet Loss

Triplet loss was originally proposed in the FaceNet (Schroff et al. 2015) paper and was used to learn face recognition of the same person at different poses and angles.

Illustration of triplet loss given one positive and one negative per anchor. (Image source: Schroff et al. 2015)

Given one anchor input 
𝑥
, we select one positive sample 
𝑥
+
 and one negative 
𝑥
−
, meaning that 
𝑥
+
 and 
𝑥
 belong to the same class and 
𝑥
−
 is sampled from another different class. Triplet loss learns to minimize the distance between the anchor 
𝑥
 and positive 
𝑥
+
 and maximize the distance between the anchor 
𝑥
 and negative 
𝑥
−
 at the same time with the following equation:



𝐿
triplet
(
𝑥
,
𝑥
+
,
𝑥
−
)
=
∑
𝑥
∈
𝑋
max
(
0
,
‖
𝑓
(
𝑥
)
−
𝑓
(
𝑥
+
)
‖
2
2
−
‖
𝑓
(
𝑥
)
−
𝑓
(
𝑥
−
)
‖
2
2
+
𝜖
)

where the margin parameter 
𝜖
 is configured as the minimum offset between distances of similar vs dissimilar pairs.

It is crucial to select challenging 
𝑥
−
 to truly improve the model.

Lifted Structured Loss

Lifted Structured Loss (Song et al. 2015) utilizes all the pairwise edges within one training batch for better computational efficiency.

Illustration compares contrastive loss, triplet loss and lifted structured loss. Red and blue edges connect similar and dissimilar sample pairs respectively. (Image source: Song et al. 2015)

Let 
𝐷
𝑖
𝑗
=
|
𝑓
(
𝑥
𝑖
)
−
𝑓
(
𝑥
𝑗
)
|
2
, a structured loss function is defined as

	




	


𝐿
struct
	
=
1
2
|
𝑃
|
∑
(
𝑖
,
𝑗
)
∈
𝑃
max
(
0
,
𝐿
struct
(
𝑖
𝑗
)
)
2


where 
𝐿
struct
(
𝑖
𝑗
)
	
=
𝐷
𝑖
𝑗
+
max
(
max
(
𝑖
,
𝑘
)
∈
𝑁
𝜖
−
𝐷
𝑖
𝑘
,
max
(
𝑗
,
𝑙
)
∈
𝑁
𝜖
−
𝐷
𝑗
𝑙
)

where 
𝑃
 contains the set of positive pairs and 
𝑁
 is the set of negative pairs. Note that the dense pairwise squared distance matrix can be easily computed per training batch.

The red part in 
𝐿
struct
(
𝑖
𝑗
)
 is used for mining hard negatives. However, it is not smooth and may cause the convergence to a bad local optimum in practice. Thus, it is relaxed to be:




𝐿
struct
(
𝑖
𝑗
)
=
𝐷
𝑖
𝑗
+
log
⁡
(
∑
(
𝑖
,
𝑘
)
∈
𝑁
exp
⁡
(
𝜖
−
𝐷
𝑖
𝑘
)
+
∑
(
𝑗
,
𝑙
)
∈
𝑁
exp
⁡
(
𝜖
−
𝐷
𝑗
𝑙
)
)

In the paper, they also proposed to enhance the quality of negative samples in each batch by actively incorporating difficult negative samples given a few random positive pairs.

N-pair Loss

Multi-Class N-pair loss (Sohn 2016) generalizes triplet loss to include comparison with multiple negative samples.

Given a 
(
𝑁
+
1
)
-tuplet of training samples, 
{
𝑥
,
𝑥
+
,
𝑥
1
−
,
…
,
𝑥
𝑁
−
1
−
}
, including one positive and 
𝑁
−
1
 negative ones, N-pair loss is defined as:

	



	
𝐿
N-pair
(
𝑥
,
𝑥
+
,
{
𝑥
𝑖
−
}
𝑖
=
1
𝑁
−
1
)
	
=
log
⁡
(
1
+
∑
𝑖
=
1
𝑁
−
1
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
𝑖
−
)
−
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
)
)

	
=
−
log
⁡
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
)
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
)
+
∑
𝑖
=
1
𝑁
−
1
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
𝑖
−
)
)

If we only sample one negative sample per class, it is equivalent to the softmax loss for multi-class classification.

NCE

Noise Contrastive Estimation, short for NCE, is a method for estimating parameters of a statistical model, proposed by Gutmann & Hyvarinen in 2010. The idea is to run logistic regression to tell apart the target data from noise. Read more on how NCE is used for learning word embedding here.

Let 
𝑥
 be the target sample 
∼
𝑃
(
𝑥
|
𝐶
=
1
;
𝜃
)
=
𝑝
𝜃
(
𝑥
)
 and 
𝑥
~
 be the noise sample 
∼
𝑃
(
𝑥
~
|
𝐶
=
0
)
=
𝑞
(
𝑥
~
)
. Note that the logistic regression models the logit (i.e. log-odds) and in this case we would like to model the logit of a sample 
𝑢
 from the target data distribution instead of the noise distribution:

ℓ
𝜃
(
𝑢
)
=
log
⁡
𝑝
𝜃
(
𝑢
)
𝑞
(
𝑢
)
=
log
⁡
𝑝
𝜃
(
𝑢
)
−
log
⁡
𝑞
(
𝑢
)

After converting logits into probabilities with sigmoid 
𝜎
(
.
)
, we can apply cross entropy loss:

	



	
𝐿
NCE
	
=
−
1
𝑁
∑
𝑖
=
1
𝑁
[
log
⁡
𝜎
(
ℓ
𝜃
(
𝑥
𝑖
)
)
+
log
⁡
(
1
−
𝜎
(
ℓ
𝜃
(
𝑥
~
𝑖
)
)
)
]


 where 
𝜎
(
ℓ
)
	
=
1
1
+
exp
⁡
(
−
ℓ
)
=
𝑝
𝜃
𝑝
𝜃
+
𝑞

Here I listed the original form of NCE loss which works with only one positive and one noise sample. In many follow-up works, contrastive loss incorporating multiple negative samples is also broadly referred to as NCE.

InfoNCE

The InfoNCE loss in CPC (Contrastive Predictive Coding; van den Oord, et al. 2018), inspired by NCE, uses categorical cross-entropy loss to identify the positive sample amongst a set of unrelated noise samples.

Given a context vector 
𝑐
, the positive sample should be drawn from the conditional distribution 
𝑝
(
𝑥
|
𝑐
)
, while 
𝑁
−
1
 negative samples are drawn from the proposal distribution 
𝑝
(
𝑥
)
, independent from the context 
𝑐
. For brevity, let us label all the samples as 
𝑋
=
{
𝑥
𝑖
}
𝑖
=
1
𝑁
 among which only one of them 
𝑥
pos
 is a positive sample. The probability of we detecting the positive sample correctly is:

𝑝
(
𝐶
=
pos
|
𝑋
,
𝑐
)
=
𝑝
(
𝑥
pos
|
𝑐
)
∏
𝑖
=
1
,
…
,
𝑁
;
𝑖
≠
pos
𝑝
(
𝑥
𝑖
)
∑
𝑗
=
1
𝑁
[
𝑝
(
𝑥
𝑗
|
𝑐
)
∏
𝑖
=
1
,
…
,
𝑁
;
𝑖
≠
𝑗
𝑝
(
𝑥
𝑖
)
]
=
𝑝
(
𝑥
pos
|
𝑐
)
𝑝
(
𝑥
pos
)
∑
𝑗
=
1
𝑁
𝑝
(
𝑥
𝑗
|
𝑐
)
𝑝
(
𝑥
𝑗
)
=
𝑓
(
𝑥
pos
,
𝑐
)
∑
𝑗
=
1
𝑁
𝑓
(
𝑥
𝑗
,
𝑐
)

where the scoring function is 
𝑓
(
𝑥
,
𝑐
)
∝
𝑝
(
𝑥
|
𝑐
)
𝑝
(
𝑥
)
.

The InfoNCE loss optimizes the negative log probability of classifying the positive sample correctly:

𝐿
InfoNCE
=
−
𝐸
[
log
⁡
𝑓
(
𝑥
,
𝑐
)
∑
𝑥
′
∈
𝑋
𝑓
(
𝑥
′
,
𝑐
)
]

The fact that 
𝑓
(
𝑥
,
𝑐
)
 estimates the density ratio 
𝑝
(
𝑥
|
𝑐
)
𝑝
(
𝑥
)
 has a connection with mutual information optimization. To maximize the the mutual information between input 
𝑥
 and context vector 
𝑐
, we have:





𝐼
(
𝑥
;
𝑐
)
=
∑
𝑥
,
𝑐
𝑝
(
𝑥
,
𝑐
)
log
⁡
𝑝
(
𝑥
,
𝑐
)
𝑝
(
𝑥
)
𝑝
(
𝑐
)
=
∑
𝑥
,
𝑐
𝑝
(
𝑥
,
𝑐
)
log
𝑝
(
𝑥
|
𝑐
)
𝑝
(
𝑥
)

where the logarithmic term in blue is estimated by 
𝑓
.

For sequence prediction tasks, rather than modeling the future observations 
𝑝
𝑘
(
𝑥
𝑡
+
𝑘
|
𝑐
𝑡
)
 directly (which could be fairly expensive), CPC models a density function to preserve the mutual information between 
𝑥
𝑡
+
𝑘
 and 
𝑐
𝑡
:

𝑓
𝑘
(
𝑥
𝑡
+
𝑘
,
𝑐
𝑡
)
=
exp
⁡
(
𝑧
𝑡
+
𝑘
⊤
𝑊
𝑘
𝑐
𝑡
)
∝
𝑝
(
𝑥
𝑡
+
𝑘
|
𝑐
𝑡
)
𝑝
(
𝑥
𝑡
+
𝑘
)

where 
𝑧
𝑡
+
𝑘
 is the encoded input and 
𝑊
𝑘
 is a trainable weight matrix.

Soft-Nearest Neighbors Loss

Soft-Nearest Neighbors Loss (Salakhutdinov & Hinton 2007, Frosst et al. 2019) extends it to include multiple positive samples.

Given a batch of samples, 
{
𝑥
𝑖
,
𝑦
𝑖
)
}
𝑖
=
1
𝐵
 where 
𝑦
𝑖
 is the class label of 
𝑥
𝑖
 and a function 
𝑓
(
.
,
.
)
 for measuring similarity between two inputs, the soft nearest neighbor loss at temperature 
𝜏
 is defined as:



𝐿
snn
=
−
1
𝐵
∑
𝑖
=
1
𝐵
log
⁡
∑
𝑖
≠
𝑗
,
𝑦
𝑖
=
𝑦
𝑗
,
𝑗
=
1
,
…
,
𝐵
exp
⁡
(
−
𝑓
(
𝑥
𝑖
,
𝑥
𝑗
)
/
𝜏
)
∑
𝑖
≠
𝑘
,
𝑘
=
1
,
…
,
𝐵
exp
⁡
(
−
𝑓
(
𝑥
𝑖
,
𝑥
𝑘
)
/
𝜏
)

The temperature 
𝜏
 is used for tuning how concentrated the features are in the representation space. For example, when at low temperature, the loss is dominated by the small distances and widely separated representations cannot contribute much and become irrelevant.

Common Setup

We can loosen the definition of “classes” and “labels” in soft nearest-neighbor loss to create positive and negative sample pairs out of unsupervised data by, for example, applying data augmentation to create noise versions of original samples.

Most recent studies follow the following definition of contrastive learning objective to incorporate multiple positive and negative samples. According to the setup in (Wang & Isola 2020), let 
𝑝
data
(
.
)
 be the data distribution over 
𝑅
𝑛
 and 
𝑝
pos
(
.
,
.
)
 be the distribution of positive pairs over 
𝑅
𝑛
×
𝑛
. These two distributions should satisfy:

Symmetry: 
∀
𝑥
,
𝑥
+
,
𝑝
pos
(
𝑥
,
𝑥
+
)
=
𝑝
pos
(
𝑥
+
,
𝑥
)
Matching marginal: 
∀
𝑥
,
∫
𝑝
pos
(
𝑥
,
𝑥
+
)
𝑑
𝑥
+
=
𝑝
data
(
𝑥
)

To learn an encoder 
𝑓
(
𝑥
)
 to learn a L2-normalized feature vector, the contrastive learning objective is:

	
	
	


	
	


	
𝐿
contrastive
	
=
𝐸
(
𝑥
,
𝑥
+
)
∼
𝑝
pos
,
{
𝑥
𝑖
−
}
𝑖
=
1
𝑀
∼
i.i.d
𝑝
data
[
−
log
⁡
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
/
𝜏
)
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
/
𝜏
)
+
∑
𝑖
=
1
𝑀
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
𝑖
−
)
/
𝜏
)
]
	
	
≈
𝐸
(
𝑥
,
𝑥
+
)
∼
𝑝
pos
,
{
𝑥
𝑖
−
}
𝑖
=
1
𝑀
∼
i.i.d
𝑝
data
[
−
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
/
𝜏
+
log
⁡
(
∑
𝑖
=
1
𝑀
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
𝑖
−
)
/
𝜏
)
)
]
	
; Assuming infinite negatives

	
=
−
1
𝜏
𝐸
(
𝑥
,
𝑥
+
)
∼
𝑝
pos
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
+
𝐸
𝑥
∼
𝑝
data
[
log
⁡
𝐸
𝑥
−
∼
𝑝
data
[
∑
𝑖
=
1
𝑀
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
𝑖
−
)
/
𝜏
)
]
]
	
Key Ingredients
Heavy Data Augmentation

Given a training sample, data augmentation techniques are needed for creating noise versions of itself to feed into the loss as positive samples. Proper data augmentation setup is critical for learning good and generalizable embedding features. It introduces the non-essential variations into examples without modifying semantic meanings and thus encourages the model to learn the essential part of the representation. For example, experiments in SimCLR showed that the composition of random cropping and random color distortion is crucial for good performance on learning visual representation of images.

Large Batch Size

Using a large batch size during training is another key ingredient in the success of many contrastive learning methods (e.g. SimCLR, CLIP), especially when it relies on in-batch negatives. Only when the batch size is big enough, the loss function can cover a diverse enough collection of negative samples, challenging enough for the model to learn meaningful representation to distinguish different examples.

Hard Negative Mining

Hard negative samples should have different labels from the anchor sample, but have embedding features very close to the anchor embedding. With access to ground truth labels in supervised datasets, it is easy to identify task-specific hard negatives. For example when learning sentence embedding, we can treat sentence pairs labelled as “contradiction” in NLI datasets as hard negative pairs (e.g. SimCSE, or use top incorrect candidates returned by BM25 with most keywords matched as hard negative samples (DPR; Karpukhin et al., 2020).

However, it becomes tricky to do hard negative mining when we want to remain unsupervised. Increasing training batch size or memory bank size implicitly introduces more hard negative samples, but it leads to a heavy burden of large memory usage as a side effect.

Chuang et al. (2020) studied the sampling bias in contrastive learning and proposed debiased loss. In the unsupervised setting, since we do not know the ground truth labels, we may accidentally sample false negative samples. Sampling bias can lead to significant performance drop.

Sampling bias which refers to false negative samples in contrastive learning can lead to a big performance drop. (Image source: Chuang et al., 2020)

Let us assume the probability of anchor class 
𝑐
 is uniform 
𝜌
(
𝑐
)
=
𝜂
+
 and the probability of observing a different class is 
𝜂
−
=
1
−
𝜂
+
.

The probability of observing a positive example for 
𝑥
 is 
𝑝
𝑥
+
(
𝑥
′
)
=
𝑝
(
𝑥
′
|
ℎ
𝑥
′
=
ℎ
𝑥
)
;
The probability of getting a negative sample for 
𝑥
 is 
𝑝
𝑥
−
(
𝑥
′
)
=
𝑝
(
𝑥
′
|
ℎ
𝑥
′
≠
ℎ
𝑥
)
.

When we are sampling 
𝑥
−
 , we cannot access the true 
𝑝
𝑥
−
(
𝑥
−
)
 and thus 
𝑥
−
 may be sampled from the (undesired) anchor class 
𝑐
 with probability 
𝜂
+
. The actual sampling data distribution becomes:

𝑝
(
𝑥
′
)
=
𝜂
+
𝑝
𝑥
+
(
𝑥
′
)
+
𝜂
−
𝑝
𝑥
−
(
𝑥
′
)

Thus we can use 
𝑝
𝑥
−
(
𝑥
′
)
=
(
𝑝
(
𝑥
′
)
−
𝜂
+
𝑝
𝑥
+
(
𝑥
′
)
)
/
𝜂
−
 for sampling 
𝑥
−
 to debias the loss. With 
𝑁
 samples 
{
𝑢
𝑖
}
𝑖
=
1
𝑁
 from 
𝑝
 and 
𝑀
 samples 
{
𝑣
𝑖
}
𝑖
=
1
𝑀
 from 
𝑝
𝑥
+
 , we can estimate the expectation of the second term 
𝐸
𝑥
−
∼
𝑝
𝑥
−
[
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
−
)
)
]
 in the denominator of contrastive learning loss:





𝑔
(
𝑥
,
{
𝑢
𝑖
}
𝑖
=
1
𝑁
,
{
𝑣
𝑖
}
𝑖
=
1
𝑀
)
=
max
{
1
𝜂
−
(
1
𝑁
∑
𝑖
=
1
𝑁
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑢
𝑖
)
)
−
𝜂
+
𝑀
∑
𝑖
=
1
𝑀
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑣
𝑖
)
)
)
,
exp
⁡
(
−
1
/
𝜏
)
}

where 
𝜏
 is the temperature and 
exp
⁡
(
−
1
/
𝜏
)
 is the theoretical lower bound of 
𝐸
𝑥
−
∼
𝑝
𝑥
−
[
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
−
)
)
]
.

The final debiased contrastive loss looks like:

𝐿
debias
𝑁
,
𝑀
(
𝑓
)
=
𝐸
𝑥
,
{
𝑢
𝑖
}
𝑖
=
1
𝑁
∼
𝑝
;
𝑥
+
,
{
𝑣
𝑖
}
𝑖
=
1
𝑀
∼
𝑝
+
[
−
log
⁡
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
+
)
+
𝑁
𝑔
(
𝑥
,
{
𝑢
𝑖
}
𝑖
=
1
𝑁
,
{
𝑣
𝑖
}
𝑖
=
1
𝑀
)
]
t-SNE visualization of learned representation with debiased contrastive learning. (Image source: Chuang et al., 2020)

Following the above annotation, Robinson et al. (2021) modified the sampling probabilities to target at hard negatives by up-weighting the probability 
𝑝
𝑥
−
(
𝑥
′
)
 to be proportional to its similarity to the anchor sample. The new sampling probability 
𝑞
𝛽
(
𝑥
−
)
 is:

𝑞
𝛽
(
𝑥
−
)
∝
exp
⁡
(
𝛽
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
−
)
)
⋅
𝑝
(
𝑥
−
)

where 
𝛽
 is a hyperparameter to tune.

We can estimate the second term in the denominator 
𝐸
𝑥
−
∼
𝑞
𝛽
[
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑥
−
)
)
]
 using importance sampling where both the partition functions 
𝑍
𝛽
,
𝑍
𝛽
+
 can be estimated empirically.

	


	
𝐸
𝑢
∼
𝑞
𝛽
[
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑢
)
)
]
	
=
𝐸
𝑢
∼
𝑝
[
𝑞
𝛽
𝑝
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑢
)
)
]
=
𝐸
𝑢
∼
𝑝
[
1
𝑍
𝛽
exp
⁡
(
(
𝛽
+
1
)
𝑓
(
𝑥
)
⊤
𝑓
(
𝑢
)
)
]


𝐸
𝑣
∼
𝑞
𝛽
+
[
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑣
)
)
]
	
=
𝐸
𝑣
∼
𝑝
+
[
𝑞
𝛽
+
𝑝
exp
⁡
(
𝑓
(
𝑥
)
⊤
𝑓
(
𝑣
)
)
]
=
𝐸
𝑣
∼
𝑝
[
1
𝑍
𝛽
+
exp
⁡
(
(
𝛽
+
1
)
𝑓
(
𝑥
)
⊤
𝑓
(
𝑣
)
)
]
Pseudo code for computing NCE loss, debiased contrastive loss, and hard negative sample objective when setting 
𝑀
=
1
. (Image source: Robinson et al., 2021 )
Vision: Image Embedding
Image Augmentations

Most approaches for contrastive representation learning in the vision domain rely on creating a noise version of a sample by applying a sequence of data augmentation techniques. The augmentation should significantly change its visual appearance but keep the semantic meaning unchanged.

Basic Image Augmentation

There are many ways to modify an image while retaining its semantic meaning. We can use any one of the following augmentation or a composition of multiple operations.

Random cropping and then resize back to the original size.
Random color distortions
Random Gaussian blur
Random color jittering
Random horizontal flip
Random grayscale conversion
Multi-crop augmentation: Use two standard resolution crops and sample a set of additional low resolution crops that cover only small parts of the image. Using low resolution crops reduces the compute cost. (SwAV)
And many more …
Augmentation Strategies

Many frameworks are designed for learning good data augmentation strategies (i.e. a composition of multiple transforms). Here are a few common ones.

AutoAugment (Cubuk, et al. 2018): Inspired by NAS, AutoAugment frames the problem of learning best data augmentation operations (i.e. shearing, rotation, invert, etc.) for image classification as an RL problem and looks for the combination that leads to the highest accuracy on the evaluation set.
RandAugment (Cubuk et al., 2019): RandAugment greatly reduces the search space of AutoAugment by controlling the magnitudes of different transformation operations with a single magnitude parameter.
PBA (Population based augmentation; Ho et al., 2019): PBA combined PBT (Jaderberg et al, 2017) with AutoAugment, using the evolutionary algorithm to train a population of children models in parallel to evolve the best augmentation strategies.
UDA (Unsupervised Data Augmentation; Xie et al., 2019): Among a set of possible augmentation strategies, UDA selects those to minimize the KL divergence between the predicted distribution over an unlabelled example and its unlabelled augmented version.
Image Mixture

Image mixture methods can construct new training examples from existing data points.

Mixup (Zhang et al., 2018): It runs global-level mixture by creating a weighted pixel-wise combination of two existing images 
𝐼
1
 and 
𝐼
2
: 
𝐼
mixup
←
𝛼
𝐼
1
+
(
1
−
𝛼
)
𝐼
2
 and 
𝛼
∈
[
0
,
1
]
.
Cutmix (Yun et al., 2019): Cutmix does region-level mixture by generating a new example by combining a local region of one image with the rest of the other image. 
𝐼
cutmix
←
𝑀
𝑏
⊙
𝐼
1
+
(
1
−
𝑀
𝑏
)
⊙
𝐼
2
, where 
𝑀
𝑏
∈
{
0
,
1
}
𝐼
 is a binary mask and 
⊙
 is element-wise multiplication. It is equivalent to filling the cutout (DeVries & Taylor 2017) region with the same region from another image.
MoCHi (“Mixing of Contrastive Hard Negatives”; Kalantidis et al. 2020): Given a query 
𝑞
, MoCHi maintains a queue of 
𝐾
 negative features 
𝑄
=
{
𝑛
1
,
…
,
𝑛
𝐾
}
 and sorts these negative features by similarity to the query, 
𝑞
⊤
𝑛
, in descending order. The first 
𝑁
 items in the queue are considered as the hardest negatives, 
𝑄
𝑁
. Then synthetic hard examples can be generated by 
ℎ
=
ℎ
~
/
|
ℎ
~
|
 where 
ℎ
~
=
𝛼
𝑛
𝑖
+
(
1
−
𝛼
)
𝑛
𝑗
 and 
𝛼
∈
(
0
,
1
)
. Even harder examples can be created by mixing with the query feature, 
ℎ
′
=
ℎ
′
~
/
|
ℎ
′
~
|
2
 where 
ℎ
′
~
=
𝛽
𝑞
+
(
1
−
𝛽
)
𝑛
𝑗
 and 
𝛽
∈
(
0
,
0.5
)
.
Parallel Augmentation

This category of approaches produce two noise versions of one anchor image and aim to learn representation such that these two augmented samples share the same embedding.

SimCLR

SimCLR (Chen et al, 2020) proposed a simple framework for contrastive learning of visual representations. It learns representations for visual inputs by maximizing agreement between differently augmented views of the same sample via a contrastive loss in the latent space.

A simple framework for contrastive learning of visual representations. (Image source: Chen et al, 2020)
Randomly sample a minibatch of 
𝑁
 samples and each sample is applied with two different data augmentation operations, resulting in 
2
𝑁
 augmented samples in total.
𝑥
~
𝑖
=
𝑡
(
𝑥
)
,
𝑥
~
𝑗
=
𝑡
′
(
𝑥
)
,
𝑡
,
𝑡
′
∼
𝑇

where two separate data augmentation operators, 
𝑡
 and 
𝑡
′
, are sampled from the same family of augmentations 
𝑇
. Data augmentation includes random crop, resize with random flip, color distortions, and Gaussian blur.

Given one positive pair, other 
2
(
𝑁
−
1
)
 data points are treated as negative samples. The representation is produced by a base encoder 
𝑓
(
.
)
:
ℎ
𝑖
=
𝑓
(
𝑥
~
𝑖
)
,
ℎ
𝑗
=
𝑓
(
𝑥
~
𝑗
)
The contrastive learning loss is defined using cosine similarity 
sim
(
.
,
.
)
. Note that the loss operates on an extra projection layer of the representation 
𝑔
(
.
)
 rather than on the representation space directly. But only the representation 
ℎ
 is used for downstream tasks.
	

	
𝟙
𝑧
𝑖
	
=
𝑔
(
ℎ
𝑖
)
,
𝑧
𝑗
=
𝑔
(
ℎ
𝑗
)


𝐿
SimCLR
(
𝑖
,
𝑗
)
	
=
−
log
⁡
exp
⁡
(
sim
(
𝑧
𝑖
,
𝑧
𝑗
)
/
𝜏
)
∑
𝑘
=
1
2
𝑁
1
[
𝑘
≠
𝑖
]
exp
⁡
(
sim
(
𝑧
𝑖
,
𝑧
𝑘
)
/
𝜏
)

where 𝟙
1
[
𝑘
≠
𝑖
]
 is an indicator function: 1 if 
𝑘
≠
𝑖
 0 otherwise.

SimCLR needs a large batch size to incorporate enough negative samples to achieve good performance.

The algorithm for SimCLR. (Image source: Chen et al, 2020).
Barlow Twins

Barlow Twins (Zbontar et al. 2021) feeds two distorted versions of samples into the same network to extract features and learns to make the cross-correlation matrix between these two groups of output features close to the identity. The goal is to keep the representation vectors of different distorted versions of one sample similar, while minimizing the redundancy between these vectors.

Illustration of Barlow Twins learning pipeline. (Image source: Zbontar et al. 2021).

Let 
𝐶
 be a cross-correlation matrix computed between outputs from two identical networks along the batch dimension. 
𝐶
 is a square matrix with the size same as the feature network’s output dimensionality. Each entry in the matrix 
𝐶
𝑖
𝑗
 is the cosine similarity between network output vector dimension at index 
𝑖
,
𝑗
 and batch index 
𝑏
, 
𝑧
𝑏
,
𝑖
𝐴
 and 
𝑧
𝑏
,
𝑗
𝐵
, with a value between -1 (i.e. perfect anti-correlation) and 1 (i.e. perfect correlation).

	

				




				

	
𝐿
BT
	
=
∑
𝑖
(
1
−
𝐶
𝑖
𝑖
)
2
⏟
invariance term
+
𝜆
∑
𝑖
∑
𝑖
≠
𝑗
𝐶
𝑖
𝑗
2
⏟
redundancy reduction term


where 
𝐶
𝑖
𝑗
	
=
∑
𝑏
𝑧
𝑏
,
𝑖
𝐴
𝑧
𝑏
,
𝑗
𝐵
∑
𝑏
(
𝑧
𝑏
,
𝑖
𝐴
)
2
∑
𝑏
(
𝑧
𝑏
,
𝑗
𝐵
)
2

Barlow Twins is competitive with SOTA methods for self-supervised learning. It naturally avoids trivial constants (i.e. collapsed representations), and is robust to different training batch sizes.

Algorithm of Barlow Twins in Pytorch style pseudo code. (Image source: Zbontar et al. 2021).
BYOL

Different from the above approaches, interestingly, BYOL (Bootstrap Your Own Latent; Grill, et al 2020) claims to achieve a new state-of-the-art results without using negative samples. It relies on two neural networks, referred to as online and target networks that interact and learn from each other. The target network (parameterized by 
𝜉
) has the same architecture as the online one (parameterized by 
𝜃
), but with polyak averaged weights, 
𝜉
←
𝜏
𝜉
+
(
1
−
𝜏
)
𝜃
.

The goal is to learn a presentation 
𝑦
 that can be used in downstream tasks. The online network parameterized by 
𝜃
 contains:

An encoder 
𝑓
𝜃
;
A projector 
𝑔
𝜃
;
A predictor 
𝑞
𝜃
.

The target network has the same network architecture, but with different parameter 
𝜉
, updated by polyak averaging 
𝜃
: 
𝜉
←
𝜏
𝜉
+
(
1
−
𝜏
)
𝜃
.

The model architecture of BYOL. After training, we only care about 
𝑓
_
𝜃
 for producing representation, 
𝑦
=
𝑓
_
𝜃
(
𝑥
)
, and everything else is discarded. 
sg
 means stop gradient. (Image source: Grill, et al 2020)

Given an image 
𝑥
, the BYOL loss is constructed as follows:

Create two augmented views: 
𝑣
=
𝑡
(
𝑥
)
;
𝑣
′
=
𝑡
′
(
𝑥
)
 with augmentations sampled 
𝑡
∼
𝑇
,
𝑡
′
∼
𝑇
′
;
Then they are encoded into representations, 
𝑦
𝜃
=
𝑓
𝜃
(
𝑣
)
,
𝑦
′
=
𝑓
𝜉
(
𝑣
′
)
;
Then they are projected into latent variables, 
𝑧
𝜃
=
𝑔
𝜃
(
𝑦
𝜃
)
,
𝑧
′
=
𝑔
𝜉
(
𝑦
′
)
;
The online network outputs a prediction 
𝑞
𝜃
(
𝑧
𝜃
)
;
Both 
𝑞
𝜃
(
𝑧
𝜃
)
 and 
𝑧
′
 are L2-normalized, giving us 
𝑞
¯
𝜃
(
𝑧
𝜃
)
=
𝑞
𝜃
(
𝑧
𝜃
)
/
|
𝑞
𝜃
(
𝑧
𝜃
)
|
 and 
𝑧
′
¯
=
𝑧
′
/
|
𝑧
′
|
;
The loss 
𝐿
𝜃
BYOL
 is MSE between L2-normalized prediction 
𝑞
¯
𝜃
(
𝑧
)
 and 
𝑧
′
¯
;
The other symmetric loss 
𝐿
~
𝜃
BYOL
 can be generated by switching 
𝑣
′
 and 
𝑣
; that is, feeding 
𝑣
′
 to online network and 
𝑣
 to target network.
The final loss is 
𝐿
𝜃
BYOL
+
𝐿
~
𝜃
BYOL
 and only parameters 
𝜃
 are optimized.

Unlike most popular contrastive learning based approaches, BYOL does not use negative pairs. Most bootstrapping approaches rely on pseudo-labels or cluster indices, but BYOL directly boostrapps the latent representation.

It is quite interesting and surprising that without negative samples, BYOL still works well. Later I ran into this post by Abe Fetterman & Josh Albrecht, they highlighted two surprising findings while they were trying to reproduce BYOL:

BYOL generally performs no better than random when batch normalization is removed.
The presence of batch normalization implicitly causes a form of contrastive learning. They believe that using negative samples is important for avoiding model collapse (i.e. what if you use all-zeros representation for every data point?). Batch normalization injects dependency on negative samples inexplicitly because no matter how similar a batch of inputs are, the values are re-distributed (spread out 
∼
𝑁
(
0
,
1
) and therefore batch normalization prevents model collapse. Strongly recommend you to read the full article if you are working in this area.
Memory Bank

Computing embeddings for a large number of negative samples in every batch is extremely expensive. One common approach is to store the representation in memory to trade off data staleness for cheaper compute.

Instance Discrimination with Memoy Bank

Instance contrastive learning (Wu et al, 2018) pushes the class-wise supervision to the extreme by considering each instance as a distinct class of its own. It implies that the number of “classes” will be the same as the number of samples in the training dataset. Hence, it is unfeasible to train a softmax layer with these many heads, but instead it can be approximated by NCE.

The training pipeline of instance-level contrastive learning. The learned embedding is L2-normalized. (Image source: Wu et al, 2018)

Let 
𝑣
=
𝑓
𝜃
(
𝑥
)
 be an embedding function to learn and the vector is normalized to have 
|
𝑣
|
=
1
. A non-parametric classifier predicts the probability of a sample 
𝑣
 belonging to class 
𝑖
 with a temperature parameter 
𝜏
:

𝑃
(
𝐶
=
𝑖
|
𝑣
)
=
exp
⁡
(
𝑣
𝑖
⊤
𝑣
/
𝜏
)
∑
𝑗
=
1
𝑛
exp
⁡
(
𝑣
𝑗
⊤
𝑣
/
𝜏
)

Instead of computing the representations for all the samples every time, they implement an Memory Bank for storing sample representation in the database from past iterations. Let 
𝑉
=
{
𝑣
𝑖
}
 be the memory bank and 
𝑓
𝑖
=
𝑓
𝜃
(
𝑥
𝑖
)
 be the feature generated by forwarding the network. We can use the representation from the memory bank 
𝑣
𝑖
 instead of the feature forwarded from the network 
𝑓
𝑖
 when comparing pairwise similarity.

The denominator theoretically requires access to the representations of all the samples, but that is too expensive in practice. Instead we can estimate it via Monte Carlo approximation using a random subset of 
𝑀
 indices 
{
𝑗
𝑘
}
𝑘
=
1
𝑀
.

𝑃
(
𝑖
|
𝑣
)
=
exp
⁡
(
𝑣
⊤
𝑓
𝑖
/
𝜏
)
∑
𝑗
=
1
𝑁
exp
⁡
(
𝑣
𝑗
⊤
𝑓
𝑖
/
𝜏
)
≃
exp
⁡
(
𝑣
⊤
𝑓
𝑖
/
𝜏
)
𝑁
𝑀
∑
𝑘
=
1
𝑀
exp
⁡
(
𝑣
𝑗
𝑘
⊤
𝑓
𝑖
/
𝜏
)

Because there is only one instance per class, the training is unstable and fluctuates a lot. To improve the training smoothness, they introduced an extra term for positive samples in the loss function based on the proximal optimization method. The final NCE loss objective looks like:

	

	
𝐿
instance
	
=
−
𝐸
𝑃
𝑑
[
log
⁡
ℎ
(
𝑖
,
𝑣
𝑖
(
𝑡
−
1
)
)
−
𝜆
‖
𝑣
𝑖
(
𝑡
)
−
𝑣
𝑖
(
𝑡
−
1
)
‖
2
2
]
−
𝑀
𝐸
𝑃
𝑛
[
log
⁡
(
1
−
ℎ
(
𝑖
,
𝑣
′
(
𝑡
−
1
)
)
]


ℎ
(
𝑖
,
𝑣
)
	
=
𝑃
(
𝑖
|
𝑣
)
𝑃
(
𝑖
|
𝑣
)
+
𝑀
𝑃
𝑛
(
𝑖
)
 where the noise distribution is uniform 
𝑃
𝑛
=
1
/
𝑁

where 
{
𝑣
(
𝑡
−
1
)
}
 are embeddings stored in the memory bank from the previous iteration. The difference between iterations 
|
𝑣
𝑖
(
𝑡
)
−
𝑣
𝑖
(
𝑡
−
1
)
|
2
2
 will gradually vanish as the learned embedding converges.

MoCo & MoCo-V2

Momentum Contrast (MoCo; He et al, 2019) provides a framework of unsupervised learning visual representation as a dynamic dictionary look-up. The dictionary is structured as a large FIFO queue of encoded representations of data samples.

Given a query sample 
𝑥
𝑞
, we get a query representation through an encoder 
𝑞
=
𝑓
𝑞
(
𝑥
𝑞
)
. A list of key representations 
{
𝑘
1
,
𝑘
2
,
…
}
 in the dictionary are encoded by a momentum encoder 
𝑘
𝑖
=
𝑓
𝑘
(
𝑥
𝑖
𝑘
)
. Let’s assume among them there is a single positive key 
𝑘
+
 in the dictionary that matches 
𝑞
. In the paper, they create 
𝑘
+
 using a noise copy of 
𝑥
𝑞
 with different augmentation. Then the InfoNCE contrastive loss with temperature 
𝜏
 is used over one positive and 
𝑁
−
1
 negative samples:

𝐿
MoCo
=
−
log
⁡
exp
⁡
(
𝑞
⋅
𝑘
+
/
𝜏
)
∑
𝑖
=
1
𝑁
exp
⁡
(
𝑞
⋅
𝑘
𝑖
/
𝜏
)

Compared to the memory bank, a queue-based dictionary in MoCo enables us to reuse representations of immediately preceding mini-batches of data.

The MoCo dictionary is not differentiable as a queue, so we cannot rely on back-propagation to update the key encoder 
𝑓
𝑘
. One naive way might be to use the same encoder for both 
𝑓
𝑞
 and 
𝑓
𝑘
. Differently, MoCo proposed to use a momentum-based update with a momentum coefficient 
𝑚
∈
[
0
,
1
)
. Say, the parameters of 
𝑓
𝑞
 and 
𝑓
𝑘
 are labeled as 
𝜃
𝑞
 and 
𝜃
𝑘
, respectively.

𝜃
𝑘
←
𝑚
𝜃
𝑘
+
(
1
−
𝑚
)
𝜃
𝑞
Illustration of how Momentum Contrast (MoCo) learns visual representations. (Image source: He et al, 2019)

The advantage of MoCo compared to SimCLR is that MoCo decouples the batch size from the number of negatives, but SimCLR requires a large batch size in order to have enough negative samples and suffers performance drops when their batch size is reduced.

Two designs in SimCLR, namely, (1) an MLP projection head and (2) stronger data augmentation, are proved to be very efficient. MoCo V2 (Chen et al, 2020) combined these two designs, achieving even better transfer performance with no dependency on a very large batch size.

CURL

CURL (Srinivas, et al. 2020) applies the above ideas in Reinforcement Learning. It learns a visual representation for RL tasks by matching embeddings of two data-augmented versions, 
𝑜
𝑞
 and 
𝑜
𝑘
, of the raw observation 
𝑜
 via contrastive loss. CURL primarily relies on random crop data augmentation. The key encoder is implemented as a momentum encoder with weights as EMA of the query encoder weights, same as in MoCo.

One significant difference between RL and supervised visual tasks is that RL depends on temporal consistency between consecutive frames. Therefore, CURL applies augmentation consistently on each stack of frames to retain information about the temporal structure of the observation.

The architecture of CURL. (Image source: Srinivas, et al. 2020)
Feature Clustering
DeepCluster

DeepCluster (Caron et al. 2018) iteratively clusters features via k-means and uses cluster assignments as pseudo labels to provide supervised signals.

Illustration of DeepCluster method which iteratively clusters deep features and uses the cluster assignments as pseudo-labels. (Image source: Caron et al. 2018)

In each iteration, DeepCluster clusters data points using the prior representation and then produces the new cluster assignments as the classification targets for the new representation. However this iterative process is prone to trivial solutions. While avoiding the use of negative pairs, it requires a costly clustering phase and specific precautions to avoid collapsing to trivial solutions.

SwAV

SwAV (Swapping Assignments between multiple Views; Caron et al. 2020) is an online contrastive learning algorithm. It computes a code from an augmented version of the image and tries to predict this code using another augmented version of the same image.

Comparison of SwAV and [contrastive instance learning](#instance-discrimination-with-memoy-bank). (Image source: Caron et al. 2020)

Given features of images with two different augmentations, 
𝑧
𝑡
 and 
𝑧
𝑠
, SwAV computes corresponding codes 
𝑞
𝑡
 and 
𝑞
𝑠
 and the loss quantifies the fit by swapping two codes using 
ℓ
(
.
)
 to measure the fit between a feature and a code.

𝐿
SwAV
(
𝑧
𝑡
,
𝑧
𝑠
)
=
ℓ
(
𝑧
𝑡
,
𝑞
𝑠
)
+
ℓ
(
𝑧
𝑠
,
𝑞
𝑡
)

The swapped fit prediction depends on the cross entropy between the predicted code and a set of 
𝐾
 trainable prototype vectors 
𝐶
=
{
𝑐
1
,
…
,
𝑐
𝐾
}
. The prototype vector matrix is shared across different batches and represents anchor clusters that each instance should be clustered to.



ℓ
(
𝑧
𝑡
,
𝑞
𝑠
)
=
−
∑
𝑘
𝑞
𝑠
(
𝑘
)
log
⁡
𝑝
𝑡
(
𝑘
)
 where 
𝑝
𝑡
(
𝑘
)
=
exp
⁡
(
𝑧
𝑡
⊤
𝑐
𝑘
/
𝜏
)
∑
𝑘
′
exp
⁡
(
𝑧
𝑡
⊤
𝑐
𝑘
′
/
𝜏
)

In a mini-batch containing 
𝐵
 feature vectors 
𝑍
=
[
𝑧
1
,
…
,
𝑧
𝐵
]
, the mapping matrix between features and prototype vectors is defined as 
𝑄
=
[
𝑞
1
,
…
,
𝑞
𝐵
]
∈
𝑅
+
𝐾
×
𝐵
. We would like to maximize the similarity between the features and the prototypes:


	
	
max
𝑄
∈
𝑄
	
Tr
(
𝑄
⊤
𝐶
⊤
𝑍
)
+
𝜀
𝐻
(
𝑄
)


where 
𝑄
	
=
{
𝑄
∈
𝑅
+
𝐾
×
𝐵
∣
𝑄
1
𝐵
=
1
𝐾
1
𝐾
,
𝑄
⊤
1
𝐾
=
1
𝐵
1
𝐵
}

where 
𝐻
 is the entropy, 
𝐻
(
𝑄
)
=
−
∑
𝑖
𝑗
𝑄
𝑖
𝑗
log
⁡
𝑄
𝑖
𝑗
, controlling the smoothness of the code. The coefficient 
𝜖
 should not be too large; otherwise, all the samples will be assigned uniformly to all the clusters. The candidate set of solutions for 
𝑄
 requires every mapping matrix to have each row sum up to 
1
/
𝐾
 and each column to sum up to 
1
/
𝐵
, enforcing that each prototype gets selected at least 
𝐵
/
𝐾
 times on average.

SwAV relies on the iterative Sinkhorn-Knopp algorithm (Cuturi 2013) to find the solution for 
𝑄
.

Working with Supervised Datasets
CLIP

CLIP (Contrastive Language-Image Pre-training; Radford et al. 2021) jointly trains a text encoder and an image feature extractor over the pretraining task that predicts which caption goes with which image.

Illustration of CLIP contrastive pre-training over text-image pairs. (Image source: Radford et al. 2021)

Given a batch of 
𝑁
 (image, text) pairs, CLIP computes the dense cosine similarity matrix between all 
𝑁
×
𝑁
 possible (image, text) candidates within this batch. The text and image encoders are jointly trained to maximize the similarity between 
𝑁
 correct pairs of (image, text) associations while minimizing the similarity for 
𝑁
(
𝑁
−
1
)
 incorrect pairs via a symmetric cross entropy loss over the dense matrix.

See the numy-like pseudo code for CLIP in

CLIP algorithm in Numpy style pseudo code. (Image source: Radford et al. 2021)

Compared to other methods above for learning good visual representation, what makes CLIP really special is “the appreciation of using natural language as a training signal”. It does demand access to supervised dataset in which we know which text matches which image. It is trained on 400 million (text, image) pairs, collected from the Internet. The query list contains all the words occurring at least 100 times in the English version of Wikipedia. Interestingly, they found that Transformer-based language models are 3x slower than a bag-of-words (BoW) text encoder at zero-shot ImageNet classification. Using contrastive objective instead of trying to predict the exact words associated with images (i.e. a method commonly adopted by image caption prediction tasks) can further improve the data efficiency another 4x.

Using bag-of-words text encoding and contrastive training objectives can bring in multiple folds of data efficiency improvement. (Image source: Radford et al. 2021)

CLIP produces good visual representation that can non-trivially transfer to many CV benchmark datasets, achieving results competitive with supervised baseline. Among tested transfer tasks, CLIP struggles with very fine-grained classification, as well as abstract or systematic tasks such as counting the number of objects. The transfer performance of CLIP models is smoothly correlated with the amount of model compute.

Supervised Contrastive Learning

There are several known issues with cross entropy loss, such as the lack of robustness to noisy labels and the possibility of poor margins. Existing improvement for cross entropy loss involves the curation of better training data, such as label smoothing and data augmentation. Supervised Contrastive Loss (Khosla et al. 2021) aims to leverage label information more effectively than cross entropy, imposing that normalized embeddings from the same class are closer together than embeddings from different classes.

Supervised vs self-supervised contrastive losses. Supervised contrastive learning considers different samples from the same class as positive examples, in addition to augmented versions. (Image source: Khosla et al. 2021)

Given a set of randomly sampled 
𝑛
 (image, label) pairs, 
{
𝑥
𝑖
,
𝑦
𝑖
}
𝑖
=
1
𝑛
, 
2
𝑛
 training pairs can be created by applying two random augmentations of every sample, 
{
𝑥
~
𝑖
,
𝑦
~
𝑖
}
𝑖
=
1
2
𝑛
.

Supervised contrastive loss 
𝐿
supcon
 utilizes multiple positive and negative samples, very similar to soft nearest-neighbor loss:





𝐿
supcon
=
−
∑
𝑖
=
1
2
𝑛
1
2
|
𝑁
𝑖
|
−
1
∑
𝑗
∈
𝑁
(
𝑦
𝑖
)
,
𝑗
≠
𝑖
log
⁡
exp
⁡
(
𝑧
𝑖
⋅
𝑧
𝑗
/
𝜏
)
∑
𝑘
∈
𝐼
,
𝑘
≠
𝑖
exp
⁡
(
𝑧
𝑖
⋅
𝑧
𝑘
/
𝜏
)

where 
𝑧
𝑘
=
𝑃
(
𝐸
(
𝑥
𝑘
~
)
)
, in which 
𝐸
(
.
)
 is an encoder network (augmented image mapped to vector) 
𝑃
(
.
)
 is a projection network (one vector mapped to another). 
𝑁
𝑖
=
{
𝑗
∈
𝐼
:
𝑦
~
𝑗
=
𝑦
~
𝑖
}
 contains a set of indices of samples with label 
𝑦
𝑖
. Including more positive samples into the set 
𝑁
𝑖
 leads to improved results.

According to their experiments, supervised contrastive loss:

does outperform the base cross entropy, but only by a small amount.
outperforms the cross entropy on robustness benchmark (ImageNet-C, which applies common naturally occuring perturbations such as noise, blur and contrast changes to the ImageNet dataset).
is less sensitive to hyperparameter changes.
Language: Sentence Embedding

In this section, we focus on how to learn sentence embedding.

Text Augmentation

Most contrastive methods in vision applications depend on creating an augmented version of each image. However, it is more challenging to construct text augmentation which does not alter the semantics of a sentence. In this section we look into three approaches for augmenting text sequences, including lexical edits, back-translation and applying cutoff or dropout.

Lexical Edits

EDA (Easy Data Augmentation; Wei & Zou 2019) defines a set of simple but powerful operations for text augmentation. Given a sentence, EDA randomly chooses and applies one of four simple operations:

Synonym replacement (SR): Replace 
𝑛
 random non-stop words with their synonyms.
Random insertion (RI): Place a random synonym of a randomly selected non-stop word in the sentence at a random position.
Random swap (RS): Randomly swap two words and repeat 
𝑛
 times.
Random deletion (RD): Randomly delete each word in the sentence with probability 
𝑝
.

where 
𝑝
=
𝛼
 and 
𝑛
=
𝛼
×
sentence_length
, with the intuition that longer sentences can absorb more noise while maintaining the original label. The hyperparameter 
𝛼
 roughly indicates the percent of words in one sentence that may be changed by one augmentation.

EDA is shown to improve the classification accuracy on several classification benchmark datasets compared to baseline without EDA. The performance lift is more significant on a smaller training set. All the four operations in EDA help improve the classification accuracy, but get to optimal at different 
𝛼
’s.

EDA leads to performance improvement on several classification benchmarks. (Image source: Wei & Zou 2019)

In Contextual Augmentation (Sosuke Kobayashi, 2018), new substitutes for word 
𝑤
𝑖
 at position 
𝑖
 can be smoothly sampled from a given probability distribution, 
𝑝
(
.
∣
𝑆
∖
{
𝑤
𝑖
}
)
, which is predicted by a bidirectional LM like BERT.

Back-translation

CERT (Contrastive self-supervised Encoder Representations from Transformers; Fang et al. (2020); code) generates augmented sentences via back-translation. Various translation models for different languages can be employed for creating different versions of augmentations. Once we have a noise version of text samples, many contrastive learning frameworks introduced above, such as MoCo, can be used to learn sentence embedding.

Dropout and Cutoff

Shen et al. (2020) proposed to apply Cutoff to text augmentation, inspired by cross-view training. They proposed three cutoff augmentation strategies:

Token cutoff removes the information of a few selected tokens. To make sure there is no data leakage, corresponding tokens in the input, positional and other relevant embedding matrices should all be zeroed out.,
Feature cutoff removes a few feature columns.
Span cutoff removes a continuous chunk of texts.
Schematic illustration of token, feature and span cutoff augmentation strategies. (Image source: Shen et al. 2020)

Multiple augmented versions of one sample can be created. When training, Shen et al. (2020) applied an additional KL-divergence term to measure the consensus between predictions from different augmented samples.

SimCSE (Gao et al. 2021; code) learns from unsupervised data by predicting a sentence from itself with only dropout noise. In other words, they treat dropout as data augmentation for text sequences. A sample is simply fed into the encoder twice with different dropout masks and these two versions are the positive pair where the other in-batch samples are considered as negative pairs. It feels quite similar to the cutoff augmentation, but dropout is more flexible with less well-defined semantic meaning of what content can be masked off.

SimCSE creates augmented samples by applying different dropout masks. The supervised version leverages NLI datasets to predict positive (entailment) or negative (contradiction) given a pair of sentences. (Image source: Gao et al. 2021)

They ran experiments on 7 STS (Semantic Text Similarity) datasets and computed cosine similarity between sentence embeddings. They also tried out an optional MLM auxiliary objective loss to help avoid catastrophic forgetting of token-level knowledge. This aux loss was found to help improve performance on transfer tasks, but a consistent drop on the main STS tasks.

Experiment numbers on a collection of STS benchmarks with SimCES. (Image source: Gao et al. 2021)
Supervision from NLI

The pre-trained BERT sentence embedding without any fine-tuning has been found to have poor performance for semantic similarity tasks. Instead of using the raw embeddings directly, we need to refine the embedding with further fine-tuning.

Natural Language Inference (NLI) tasks are the main data sources to provide supervised signals for learning sentence embedding; such as SNLI, MNLI, and QQP.

Sentence-BERT

SBERT (Sentence-BERT) (Reimers & Gurevych, 2019) relies on siamese and triplet network architectures to learn sentence embeddings such that the sentence similarity can be estimated by cosine similarity between pairs of embeddings. Note that learning SBERT depends on supervised data, as it is fine-tuned on several NLI datasets.

They experimented with a few different prediction heads on top of BERT model:

Softmax classification objective: The classification head of the siamese network is built on the concatenation of two embeddings 
𝑓
(
𝑥
)
,
𝑓
(
𝑥
′
)
 and 
|
𝑓
(
𝑥
)
−
𝑓
(
𝑥
′
)
|
. The predicted output is 
𝑦
^
=
softmax
(
𝑊
𝑡
[
𝑓
(
𝑥
)
;
𝑓
(
𝑥
′
)
;
|
𝑓
(
𝑥
)
−
𝑓
(
𝑥
′
)
|
]
)
. They showed that the most important component is the element-wise difference 
|
𝑓
(
𝑥
)
−
𝑓
(
𝑥
′
)
|
.
Regression objective: This is the regression loss on 
cos
⁡
(
𝑓
(
𝑥
)
,
𝑓
(
𝑥
′
)
)
, in which the pooling strategy has a big impact. In the experiments, they observed that max performs much worse than mean and CLS-token.
Triplet objective: 
max
(
0
,
|
𝑓
(
𝑥
)
−
𝑓
(
𝑥
+
)
|
−
|
𝑓
(
𝑥
)
−
𝑓
(
𝑥
−
)
|
+
𝜖
)
, where 
𝑥
,
𝑥
+
,
𝑥
−
 are embeddings of the anchor, positive and negative sentences.

In the experiments, which objective function works the best depends on the datasets, so there is no universal winner.

Illustration of Sentence-BERT training framework with softmax classification head and regression head. (Image source: Reimers & Gurevych, 2019)

The SentEval library (Conneau and Kiela, 2018) is commonly used for evaluating the quality of learned sentence embedding. SBERT outperformed other baselines at that time (Aug 2019) on 5 out of 7 tasks.

The performance of Sentence-BERT on the SentEval benchmark. (Image source: Reimers & Gurevych, 2019)
BERT-flow

The embedding representation space is deemed isotropic if embeddings are uniformly distributed on each dimension; otherwise, it is anisotropic. Li et al, (2020) showed that a pre-trained BERT learns a non-smooth anisotropic semantic space of sentence embeddings and thus leads to poor performance for text similarity tasks without fine-tuning. Empirically, they observed two issues with BERT sentence embedding: Word frequency biases the embedding space. High-frequency words are close to the origin, but low-frequency ones are far away from the origin. Low-frequency words scatter sparsely. The embeddings of low-frequency words tend to be farther to their 
𝑘
-NN neighbors, while the embeddings of high-frequency words concentrate more densely.

BERT-flow (Li et al, 2020; code) was proposed to transform the embedding to a smooth and isotropic Gaussian distribution via normalizing flows.

Illustration of the flow-based calibration over the original sentence embedding space in BERT-flow. (Image source: Li et al, 2020)

Let 
𝑈
 be the observed BERT sentence embedding space and 
𝑍
 be the desired latent space which is a standard Gaussian. Thus, 
𝑝
𝑍
 is a Gaussian density function and 
𝑓
𝜙
:
𝑍
→
𝑈
 is an invertible transformation:

𝑧
∼
𝑝
𝑍
(
𝑧
)
𝑢
=
𝑓
𝜙
(
𝑧
)
𝑧
=
𝑓
𝜙
−
1
(
𝑢
)

A flow-based generative model learns the invertible mapping function by maximizing the likelihood of 
𝑈
’s marginal:



max
𝜙
𝐸
𝑢
=
BERT
(
𝑠
)
,
𝑠
∼
𝐷
[
log
⁡
𝑝
𝑍
(
𝑓
𝜙
−
1
(
𝑢
)
)
+
log
⁡
|
det
𝜕
𝑓
𝜙
−
1
(
𝑢
)
𝜕
𝑢
|
]

where 
𝑠
 is a sentence sampled from the text corpus 
𝐷
. Only the flow parameters 
𝜙
 are optimized while parameters in the pretrained BERT stay unchanged.

BERT-flow was shown to improve the performance on most STS tasks either with or without supervision from NLI datasets. Because learning normalizing flows for calibration does not require labels, it can utilize the entire dataset including validation and test sets.

Whitening Operation

Su et al. (2021) applied whitening operation to improve the isotropy of the learned representation and also to reduce the dimensionality of sentence embedding.

They transform the mean value of the sentence vectors to 0 and the covariance matrix to the identity matrix. Given a set of samples 
{
𝑥
𝑖
}
𝑖
=
1
𝑁
, let 
𝑥
~
𝑖
 and 
Σ
~
 be the transformed samples and corresponding covariance matrix:

	






	
𝜇
	
=
1
𝑁
∑
𝑖
=
1
𝑁
𝑥
𝑖
Σ
=
1
𝑁
∑
𝑖
=
1
𝑁
(
𝑥
𝑖
−
𝜇
)
⊤
(
𝑥
𝑖
−
𝜇
)


𝑥
~
𝑖
	
=
(
𝑥
𝑖
−
𝜇
)
𝑊
Σ
~
=
𝑊
⊤
Σ
𝑊
=
𝐼
 thus 
Σ
=
(
𝑊
−
1
)
⊤
𝑊
−
1

If we get SVD decomposition of 
Σ
=
𝑈
Λ
𝑈
⊤
, we will have 
𝑊
−
1
=
Λ
𝑈
⊤
 and 
𝑊
=
𝑈
Λ
−
1
. Note that within SVD, 
𝑈
 is an orthogonal matrix with column vectors as eigenvectors and 
Λ
 is a diagonal matrix with all positive elements as sorted eigenvalues.

A dimensionality reduction strategy can be applied by only taking the first 
𝑘
 columns of 
𝑊
, named Whitening-
𝑘
.

Pseudo code of the whitening-
𝑘
 operation. (Image source: Su et al. 2021)

Whitening operations were shown to outperform BERT-flow and achieve SOTA with 256 sentence dimensionality on many STS benchmarks, either with or without NLI supervision.

Unsupervised Sentence Embedding Learning
Context Prediction

Quick-Thought (QT) vectors (Logeswaran & Lee, 2018) formulate sentence representation learning as a classification problem: Given a sentence and its context, a classifier distinguishes context sentences from other contrastive sentences based on their vector representations (“cloze test”). Such a formulation removes the softmax output layer which causes training slowdown.

Illustration of how Quick-Thought sentence embedding vectors are learned. (Image source: Logeswaran & Lee, 2018)

Let 
𝑓
(
.
)
 and 
𝑔
(
.
)
 be two functions that encode a sentence 
𝑠
 into a fixed-length vector. Let 
𝐶
(
𝑠
)
 be the set of sentences in the context of 
𝑠
 and 
𝑆
(
𝑠
)
 be the set of candidate sentences including only one sentence 
𝑠
𝑐
∈
𝐶
(
𝑠
)
 and many other non-context negative sentences. Quick Thoughts model learns to optimize the probability of predicting the only true context sentence 
𝑠
𝑐
∈
𝑆
(
𝑠
)
. It is essentially NCE loss when considering the sentence 
(
𝑠
,
𝑠
𝑐
)
 as the positive pairs while other pairs 
(
𝑠
,
𝑠
′
)
 where 
𝑠
′
∈
𝑆
(
𝑠
)
,
𝑠
′
≠
𝑠
𝑐
 as negatives.






𝐿
QT
=
−
∑
𝑠
∈
𝐷
∑
𝑠
𝑐
∈
𝐶
(
𝑠
)
log
⁡
𝑝
(
𝑠
𝑐
|
𝑠
,
𝑆
(
𝑠
)
)
=
−
∑
𝑠
∈
𝐷
∑
𝑠
𝑐
∈
𝐶
(
𝑠
)
exp
⁡
(
𝑓
(
𝑠
)
⊤
𝑔
(
𝑠
𝑐
)
)
∑
𝑠
′
∈
𝑆
(
𝑠
)
exp
⁡
(
𝑓
(
𝑠
)
⊤
𝑔
(
𝑠
′
)
)
Mutual Information Maximization

IS-BERT (Info-Sentence BERT) (Zhang et al. 2020; code) adopts a self-supervised learning objective based on mutual information maximization to learn good sentence embeddings in the unsupervised manners.

Illustration of Info-Sentence BERT. (Image source: Zhang et al. 2020)

IS-BERT works as follows:

Use BERT to encode an input sentence 
𝑠
 to a token embedding of length 
𝑙
, 
ℎ
1
:
𝑙
.

Then apply 1-D conv net with different kernel sizes (e.g. 1, 3, 5) to process the token embedding sequence to capture the n-gram local contextual dependencies: 
𝑐
𝑖
=
ReLU
(
𝑤
⋅
ℎ
𝑖
:
𝑖
+
𝑘
−
1
+
𝑏
)
. The output sequences are padded to stay the same sizes of the inputs.

The final local representation of the 
𝑖
-th token 
𝐹
𝜃
(
𝑖
)
(
𝑥
)
 is the concatenation of representations of different kernel sizes.

The global sentence representation 
𝐸
𝜃
(
𝑥
)
 is computed by applying a mean-over-time pooling layer on the token representations 
𝐹
𝜃
(
𝑥
)
=
{
𝐹
𝜃
(
𝑖
)
(
𝑥
)
∈
𝑅
𝑑
}
𝑖
=
1
𝑙
.

Since the mutual information estimation is generally intractable for continuous and high-dimensional random variables, IS-BERT relies on the Jensen-Shannon estimator (Nowozin et al., 2016, Hjelm et al., 2019) to maximize the mutual information between 
𝐸
𝜃
(
𝑥
)
 and 
𝐹
𝜃
(
𝑖
)
(
𝑥
)
.

𝐼
𝜔
JSD
(
𝐹
𝜃
(
𝑖
)
(
𝑥
)
;
𝐸
𝜃
(
𝑥
)
)
=
𝐸
𝑥
∼
𝑃
[
−
sp
(
−
𝑇
𝜔
(
𝐹
𝜃
(
𝑖
)
(
𝑥
)
;
𝐸
𝜃
(
𝑥
)
)
)
]
−
𝐸
𝑥
∼
𝑃
,
𝑥
′
∼
𝑃
~
[
sp
(
𝑇
𝜔
(
𝐹
𝜃
(
𝑖
)
(
𝑥
′
)
;
𝐸
𝜃
(
𝑥
)
)
)
]

where 
𝑇
𝜔
:
𝐹
×
𝐸
→
𝑅
 is a learnable network with parameters 
𝜔
, generating discriminator scores. The negative sample 
𝑥
′
 is sampled from the distribution 
𝑃
~
=
𝑃
. And 
sp
(
𝑥
)
=
log
⁡
(
1
+
𝑒
𝑥
)
 is the softplus activation function.

The unsupervised numbers on SentEval with IS-BERT outperforms most of the unsupervised baselines (Sep 2020), but unsurprisingly weaker than supervised runs. When using labelled NLI datasets, IS-BERT produces results comparable with SBERT (See Fig. 25 & 30).

The performance of IS-BERT on the SentEval benchmark. (Image source: Zhang et al. 2020)
Citation

Cited as:

Weng, Lilian. (May 2021). Contrastive representation learning. Lil’Log. https://lilianweng.github.io/posts/2021-05-31-contrastive/.

Or

@article{weng2021contrastive,
  title   = "Contrastive Representation Learning",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2021",
  month   = "May",
  url     = "https://lilianweng.github.io/posts/2021-05-31-contrastive/"
}

References

[1] Sumit Chopra, Raia Hadsell and Yann LeCun. “Learning a similarity metric discriminatively, with application to face verification.” CVPR 2005.

[2] Florian Schroff, Dmitry Kalenichenko and James Philbin. “FaceNet: A Unified Embedding for Face Recognition and Clustering.” CVPR 2015.

[3] Hyun Oh Song et al. “Deep Metric Learning via Lifted Structured Feature Embedding.” CVPR 2016. [code]

[4] Ruslan Salakhutdinov and Geoff Hinton. “Learning a Nonlinear Embedding by Preserving Class Neighbourhood Structure” AISTATS 2007.

[5] Michael Gutmann and Aapo Hyvärinen. “Noise-contrastive estimation: A new estimation principle for unnormalized statistical models.” AISTATS 2010.

[6] Kihyuk Sohn et al. “Improved Deep Metric Learning with Multi-class N-pair Loss Objective” NIPS 2016.

[7] Nicholas Frosst, Nicolas Papernot and Geoffrey Hinton. “Analyzing and Improving Representations with the Soft Nearest Neighbor Loss.” ICML 2019

[8] Tongzhou Wang and Phillip Isola. “Understanding Contrastive Representation Learning through Alignment and Uniformity on the Hypersphere.” ICML 2020. [code]

[9] Zhirong Wu et al. “Unsupervised feature learning via non-parametric instance-level discrimination.” CVPR 2018.

[10] Ekin D. Cubuk et al. “AutoAugment: Learning augmentation policies from data.” arXiv preprint arXiv:1805.09501 (2018).

[11] Daniel Ho et al. “Population Based Augmentation: Efficient Learning of Augmentation Policy Schedules.” ICML 2019.

[12] Ekin D. Cubuk & Barret Zoph et al. “RandAugment: Practical automated data augmentation with a reduced search space.” arXiv preprint arXiv:1909.13719 (2019).

[13] Hongyi Zhang et al. “mixup: Beyond Empirical Risk Minimization.” ICLR 2017.

[14] Sangdoo Yun et al. “CutMix: Regularization Strategy to Train Strong Classifiers with Localizable Features.” ICCV 2019.

[15] Yannis Kalantidis et al. “Mixing of Contrastive Hard Negatives” NeuriPS 2020.

[16] Ashish Jaiswal et al. “A Survey on Contrastive Self-Supervised Learning.” arXiv preprint arXiv:2011.00362 (2021)

[17] Jure Zbontar et al. “Barlow Twins: Self-Supervised Learning via Redundancy Reduction.” arXiv preprint arXiv:2103.03230 (2021) [code]

[18] Alec Radford, et al. “Learning Transferable Visual Models From Natural Language Supervision” arXiv preprint arXiv:2103.00020 (2021)

[19] Mathilde Caron et al. “Unsupervised Learning of Visual Features by Contrasting Cluster Assignments (SwAV).” NeuriPS 2020.

[20] Mathilde Caron et al. “Deep Clustering for Unsupervised Learning of Visual Features.” ECCV 2018.

[21] Prannay Khosla et al. “Supervised Contrastive Learning.” NeurIPS 2020.

[22] Aaron van den Oord, Yazhe Li & Oriol Vinyals. “Representation Learning with Contrastive Predictive Coding” arXiv preprint arXiv:1807.03748 (2018).

[23] Jason Wei and Kai Zou. “EDA: Easy data augmentation techniques for boosting performance on text classification tasks.” EMNLP-IJCNLP 2019.

[24] Sosuke Kobayashi. “Contextual Augmentation: Data Augmentation by Words with Paradigmatic Relations.” NAACL 2018

[25] Hongchao Fang et al. “CERT: Contrastive self-supervised learning for language understanding.” arXiv preprint arXiv:2005.12766 (2020).

[26] Dinghan Shen et al. “A Simple but Tough-to-Beat Data Augmentation Approach for Natural Language Understanding and Generation.” arXiv preprint arXiv:2009.13818 (2020) [code]

[27] Tianyu Gao et al. “SimCSE: Simple Contrastive Learning of Sentence Embeddings.” arXiv preprint arXiv:2104.08821 (2020). [code]

[28] Nils Reimers and Iryna Gurevych. “Sentence-BERT: Sentence embeddings using Siamese BERT-networks.” EMNLP 2019.

[29] Jianlin Su et al. “Whitening sentence representations for better semantics and faster retrieval.” arXiv preprint arXiv:2103.15316 (2021). [code]

[30] Yan Zhang et al. “An unsupervised sentence embedding method by mutual information maximization.” EMNLP 2020. [code]

[31] Bohan Li et al. “On the sentence embeddings from pre-trained language models.” EMNLP 2020.

[32] Lajanugen Logeswaran and Honglak Lee. “An efficient framework for learning sentence representations.” ICLR 2018.

[33] Joshua Robinson, et al. “Contrastive Learning with Hard Negative Samples.” ICLR 2021.

[34] Ching-Yao Chuang et al. “Debiased Contrastive Learning.” NeuriPS 2020.

Representation-Learning
 
Long-Read
 
Language-Model
 
Unsupervised-Learning
 
Data-Augmentation
«
What are Diffusion Models?
»
Reducing Toxicity in Language Models
© 2026 Lil'Log Powered by Hugo & PaperMod