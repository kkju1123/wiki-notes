---
title: Self-Supervised Representation Learning
url: https://lilianweng.github.io/posts/2019-11-10-self-supervised/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:46:24.307858+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
Self-Supervised Representation Learning
Date: November 10, 2019 | Estimated Reading Time: 38 min | Author: Lilian Weng
Table of Contents

[Updated on 2020-01-09: add a new section on Contrastive Predictive Coding].
[Updated on 2020-04-13: add a “Momentum Contrast” section on MoCo, SimCLR and CURL.]
[Updated on 2020-07-08: add a “Bisimulation” section on DeepMDP and DBC.]
[Updated on 2020-09-12: add MoCo V2 and BYOL in the “Momentum Contrast” section.]
[Updated on 2021-05-31: remove section on “Momentum Contrast” and add a pointer to a full post on “Contrastive Representation Learning”]

Given a task and enough labels, supervised learning can solve it really well. Good performance usually requires a decent amount of labels, but collecting manual labels is expensive (i.e. ImageNet) and hard to be scaled up. Considering the amount of unlabelled data (e.g. free text, all the images on the Internet) is substantially more than a limited number of human curated labelled datasets, it is kinda wasteful not to use them. However, unsupervised learning is not easy and usually works much less efficiently than supervised learning.

What if we can get labels for free for unlabelled data and train unsupervised dataset in a supervised manner? We can achieve this by framing a supervised learning task in a special form to predict only a subset of information using the rest. In this way, all the information needed, both inputs and labels, has been provided. This is known as self-supervised learning.

This idea has been widely used in language modeling. The default task for a language model is to predict the next word given the past sequence. BERT adds two other auxiliary tasks and both rely on self-generated labels.

A great summary of how self-supervised learning tasks can be constructed (Image source: LeCun’s talk)

Here is a nicely curated list of papers in self-supervised learning. Please check it out if you are interested in reading more in depth.

Note that this post does not focus on either NLP / language modeling or generative modeling.

Why Self-Supervised Learning?

Self-supervised learning empowers us to exploit a variety of labels that come with the data for free. The motivation is quite straightforward. Producing a dataset with clean labels is expensive but unlabeled data is being generated all the time. To make use of this much larger amount of unlabeled data, one way is to set the learning objectives properly so as to get supervision from the data itself.

The self-supervised task, also known as pretext task, guides us to a supervised loss function. However, we usually don’t care about the final performance of this invented task. Rather we are interested in the learned intermediate representation with the expectation that this representation can carry good semantic or structural meanings and can be beneficial to a variety of practical downstream tasks.

For example, we might rotate images at random and train a model to predict how each input image is rotated. The rotation prediction task is made-up, so the actual accuracy is unimportant, like how we treat auxiliary tasks. But we expect the model to learn high-quality latent variables for real-world tasks, such as constructing an object recognition classifier with very few labeled samples.

Broadly speaking, all the generative models can be considered as self-supervised, but with different goals: Generative models focus on creating diverse and realistic images, while self-supervised representation learning care about producing good features generally helpful for many tasks. Generative modeling is not the focus of this post, but feel free to check my previous posts.

Images-Based

Many ideas have been proposed for self-supervised representation learning on images. A common workflow is to train a model on one or multiple pretext tasks with unlabelled images and then use one intermediate feature layer of this model to feed a multinomial logistic regression classifier on ImageNet classification. The final classification accuracy quantifies how good the learned representation is.

Recently, some researchers proposed to train supervised learning on labelled data and self-supervised pretext tasks on unlabelled data simultaneously with shared weights, like in Zhai et al, 2019 and Sun et al, 2019.

Distortion

We expect small distortion on an image does not modify its original semantic meaning or geometric forms. Slightly distorted images are considered the same as original and thus the learned features are expected to be invariant to distortion.

Exemplar-CNN (Dosovitskiy et al., 2015) create surrogate training datasets with unlabeled image patches:

Sample 
𝑁
 patches of size 32 × 32 pixels from different images at varying positions and scales, only from regions containing considerable gradients as those areas cover edges and tend to contain objects or parts of objects. They are “exemplary” patches.
Each patch is distorted by applying a variety of random transformations (i.e., translation, rotation, scaling, etc.). All the resulting distorted patches are considered to belong to the same surrogate class.
The pretext task is to discriminate between a set of surrogate classes. We can arbitrarily create as many surrogate classes as we want.
The original patch of a cute deer is in the top left corner. Random transformations are applied, resulting in a variety of distorted patches. All of them should be classified into the same class in the pretext task. (Image source: Dosovitskiy et al., 2015)

Rotation of an entire image (Gidaris et al. 2018 is another interesting and cheap way to modify an input image while the semantic content stays unchanged. Each input image is first rotated by a multiple of 
90
∘
 at random, corresponding to 
[
0
∘
,
90
∘
,
180
∘
,
270
∘
]
. The model is trained to predict which rotation has been applied, thus a 4-class classification problem.

In order to identify the same image with different rotations, the model has to learn to recognize high level object parts, such as heads, noses, and eyes, and the relative positions of these parts, rather than local patterns. This pretext task drives the model to learn semantic concepts of objects in this way.

Illustration of self-supervised learning by rotating the entire input images. The model learns to predict which rotation is applied. (Image source: Gidaris et al. 2018)
Patches

The second category of self-supervised learning tasks extract multiple patches from one image and ask the model to predict the relationship between these patches.

Doersch et al. (2015) formulates the pretext task as predicting the relative position between two random patches from one image. A model needs to understand the spatial context of objects in order to tell the relative position between parts.

The training patches are sampled in the following way:

Randomly sample the first patch without any reference to image content.
Considering that the first patch is placed in the middle of a 3x3 grid, and the second patch is sampled from its 8 neighboring locations around it.
To avoid the model only catching low-level trivial signals, such as connecting a straight line across boundary or matching local patterns, additional noise is introduced by:
Add gaps between patches
Small jitters
Randomly downsample some patches to as little as 100 total pixels, and then upsampling it, to build robustness to pixelation.
Shift green and magenta toward gray or randomly drop 2 of 3 color channels (See “chromatic aberration” below)
The model is trained to predict which one of 8 neighboring locations the second patch is selected from, a classification problem over 8 classes.
Illustration of self-supervised learning by predicting the relative position of two random patches. (Image source: Doersch et al., 2015)

Other than trivial signals like boundary patterns or textures continuing, another interesting and a bit surprising trivial solution was found, called “chromatic aberration”. It is triggered by different focal lengths of lights at different wavelengths passing through the lens. In the process, there might exist small offsets between color channels. Hence, the model can learn to tell the relative position by simply comparing how green and magenta are separated differently in two patches. This is a trivial solution and has nothing to do with the image content. Pre-processing images by shifting green and magenta toward gray or randomly dropping 2 of 3 color channels can avoid this trivial solution.

Illustration of how chromatic aberration happens. (Image source: wikipedia)

Since we have already set up a 3x3 grid in each image in the above task, why not use all of 9 patches rather than only 2 to make the task more difficult? Following this idea, Noroozi & Favaro (2016) designed a jigsaw puzzle game as pretext task: The model is trained to place 9 shuffled patches back to the original locations.

A convolutional network processes each patch independently with shared weights and outputs a probability vector per patch index out of a predefined set of permutations. To control the difficulty of jigsaw puzzles, the paper proposed to shuffle patches according to a predefined permutation set and configured the model to predict a probability vector over all the indices in the set.

Because how the input patches are shuffled does not alter the correct order to predict. A potential improvement to speed up training is to use permutation-invariant graph convolutional network (GCN) so that we don’t have to shuffle the same set of patches multiple times, same idea as in this paper.

Illustration of self-supervised learning by solving jigsaw puzzle. (Image source: Noroozi & Favaro, 2016)

Another idea is to consider “feature” or “visual primitives” as a scalar-value attribute that can be summed up over multiple patches and compared across different patches. Then the relationship between patches can be defined by counting features and simple arithmetic (Noroozi, et al, 2017).

The paper considers two transformations:

Scaling: If an image is scaled up by 2x, the number of visual primitives should stay the same.
Tiling: If an image is tiled into a 2x2 grid, the number of visual primitives is expected to be the sum, 4 times the original feature counts.

The model learns a feature encoder 
𝜙
(
.
)
 using the above feature counting relationship. Given an input image 
𝑥
∈
𝑅
𝑚
×
𝑛
×
3
, considering two types of transformation operators:

Downsampling operator, 
𝐷
:
𝑅
𝑚
×
𝑛
×
3
↦
𝑅
𝑚
2
×
𝑛
2
×
3
: downsample by a factor of 2
Tiling operator 
𝑇
𝑖
:
𝑅
𝑚
×
𝑛
×
3
↦
𝑅
𝑚
2
×
𝑛
2
×
3
: extract the 
𝑖
-th tile from a 2x2 grid of the image.

We expect to learn:



𝜙
(
𝑥
)
=
𝜙
(
𝐷
∘
𝑥
)
=
∑
𝑖
=
1
4
𝜙
(
𝑇
𝑖
∘
𝑥
)

Thus the MSE loss is: 
𝐿
feat
=
|
𝜙
(
𝐷
∘
𝑥
)
−
∑
𝑖
=
1
4
𝜙
(
𝑇
𝑖
∘
𝑥
)
|
2
2
. To avoid trivial solution 
𝜙
(
𝑥
)
=
0
,
∀
𝑥
, another loss term is added to encourage the difference between features of two different images: 
𝐿
diff
=
max
(
0
,
𝑐
−
|
𝜙
(
𝐷
∘
𝑦
)
−
∑
𝑖
=
1
4
𝜙
(
𝑇
𝑖
∘
𝑥
)
|
2
2
)
, where 
𝑦
 is another input image different from 
𝑥
 and 
𝑐
 is a scalar constant. The final loss is:





𝐿
=
𝐿
feat
+
𝐿
diff
=
‖
𝜙
(
𝐷
∘
𝑥
)
−
∑
𝑖
=
1
4
𝜙
(
𝑇
𝑖
∘
𝑥
)
‖
2
2
+
max
(
0
,
𝑀
−
‖
𝜙
(
𝐷
∘
𝑦
)
−
∑
𝑖
=
1
4
𝜙
(
𝑇
𝑖
∘
𝑥
)
‖
2
2
)
Self-supervised representation learning by counting features. (Image source: Noroozi, et al, 2017)
Colorization

Colorization can be used as a powerful self-supervised task: a model is trained to color a grayscale input image; precisely the task is to map this image to a distribution over quantized color value outputs (Zhang et al. 2016).

The model outputs colors in the the CIE Lab* color space. The Lab* color is designed to approximate human vision, while, in contrast, RGB or CMYK models the color output of physical devices.

L* component matches human perception of lightness; L* = 0 is black and L* = 100 indicates white.
a* component represents green (negative) / magenta (positive) value.
b* component models blue (negative) /yellow (positive) value.

Due to the multimodal nature of the colorization problem, cross-entropy loss of predicted probability distribution over binned color values works better than L2 loss of the raw color values. The ab color space is quantized with bucket size 10.

To balance between common colors (usually low ab values, of common backgrounds like clouds, walls, and dirt) and rare colors (which are likely associated with key objects in the image), the loss function is rebalanced with a weighting term that boosts the loss of infrequent color buckets. This is just like why we need both tf and idf for scoring words in information retrieval model. The weighting term is constructed as: (1-λ) * Gaussian-kernel-smoothed empirical probability distribution + λ * a uniform distribution, where both distributions are over the quantized ab color space.

Generative Modeling

The pretext task in generative modeling is to reconstruct the original input while learning meaningful latent representation.

The denoising autoencoder (Vincent, et al, 2008) learns to recover an image from a version that is partially corrupted or has random noise. The design is inspired by the fact that humans can easily recognize objects in pictures even with noise, indicating that key visual features can be extracted and separated from noise. See my old post.

The context encoder (Pathak, et al., 2016) is trained to fill in a missing piece in the image. Let 
𝑀
^
 be a binary mask, 0 for dropped pixels and 1 for remaining input pixels. The model is trained with a combination of the reconstruction (L2) loss and the adversarial loss. The removed regions defined by the mask could be of any shape.

	
	

	

𝐿
(
𝑥
)
	
=
𝐿
recon
(
𝑥
)
+
𝐿
adv
(
𝑥
)


𝐿
recon
(
𝑥
)
	
=
‖
(
1
−
𝑀
^
)
⊙
(
𝑥
−
𝐹
(
𝑀
^
⊙
𝑥
)
)
‖
2
2


𝐿
adv
(
𝑥
)
	
=
max
𝐷
𝐸
𝑥
[
log
⁡
𝐷
(
𝑥
)
+
log
⁡
(
1
−
𝐷
(
𝐹
(
𝑀
^
⊙
𝑥
)
)
)
]

where 
𝐹
(
.
)
 is the full pipeline of reconstructing the input image with missing regions via impainting, including both encoder and decoder portions in 
𝐷
(
.
)
 is the discriminator model jointly trained, like in GAN.

Illustration of context encoder. (Image source: Pathak, et al., 2016)

When applying a mask on an image, the context encoder removes information of all the color channels in partial regions. How about only hiding a subset of channels? The split-brain autoencoder (Zhang et al., 2017) does this by predicting a subset of color channels from the rest of channels. Let the data tensor 
𝑥
∈
𝑅
ℎ
×
𝑤
×
|
𝐶
|
 with 
𝐶
 color channels be the input for the 
𝑙
-th layer of the network. It is split into two disjoint parts, 
𝑥
1
∈
𝑅
ℎ
×
𝑤
×
|
𝐶
1
|
 and 
𝑥
2
∈
𝑅
ℎ
×
𝑤
×
|
𝐶
2
|
, where 
𝐶
1
,
𝐶
2
⊆
𝐶
. Then two sub-networks are trained to do two complementary predictions: one network 
𝑓
1
 predicts 
𝑥
2
 from 
𝑥
1
 and the other network 
𝑓
1
 predicts 
𝑥
1
 from 
𝑥
2
. The loss is either L1 loss or cross entropy if color values are quantized.

The split can happen once on the RGB-D or Lab* colorspace, or happen even in every layer of a CNN network in which the number of channels can be arbitrary.

Illustration of split-brain autoencoder. (Image source: Zhang et al., 2017)

The generative adversarial networks (GANs) are able to learn to map from simple latent variables to arbitrarily complex data distributions. Studies have shown that the latent space of such generative models captures semantic variation in the data; e.g. when training GAN models on human faces, some latent variables are associated with facial expression, glasses, gender, etc (Radford et al., 2016).

Bidirectional GANs (Donahue, et al, 2017) introduces an additional encoder 
𝐸
(
.
)
 to learn the mappings from the input to the latent variable 
𝑧
. The discriminator 
𝐷
(
.
)
 predicts in the joint space of the input data and latent representation, 
(
𝑥
,
𝑧
)
, to tell apart the generated pair 
(
𝑥
,
𝐸
(
𝑥
)
)
 from the real one 
(
𝐺
(
𝑧
)
,
𝑧
)
. The model is trained to optimize the objective: 
min
𝐺
,
𝐸
max
𝐷
𝑉
(
𝐷
,
𝐸
,
𝐺
)
, where the generator 
𝐺
 and the encoder 
𝐸
 learn to generate data and latent variables that are realistic enough to confuse the discriminator and at the same time the discriminator 
𝐷
 tries to differentiate real and generated data.


				

				

𝑉
(
𝐷
,
𝐸
,
𝐺
)
=
𝐸
𝑥
∼
𝑝
𝑥
[
𝐸
𝑧
∼
𝑝
𝐸
(
.
|
𝑥
)
[
log
⁡
𝐷
(
𝑥
,
𝑧
)
]
⏟
log
⁡
𝐷
(
real
)
]
+
𝐸
𝑧
∼
𝑝
𝑧
[
𝐸
𝑥
∼
𝑝
𝐺
(
.
|
𝑧
)
[
log
⁡
1
−
𝐷
(
𝑥
,
𝑧
)
]
⏟
log
⁡
(
1
−
𝐷
(
fake
)
)
)
]
Illustration of how Bidirectional GAN works. (Image source: Donahue, et al, 2017)
Contrastive Learning

The Contrastive Predictive Coding (CPC) (van den Oord, et al. 2018) is an approach for unsupervised learning from high-dimensional data by translating a generative modeling problem to a classification problem. The contrastive loss or InfoNCE loss in CPC, inspired by Noise Contrastive Estimation (NCE), uses cross-entropy loss to measure how well the model can classify the “future” representation amongst a set of unrelated “negative” samples. Such design is partially motivated by the fact that the unimodal loss like MSE has no enough capacity but learning a full generative model could be too expensive.

Illustration of applying Contrastive Predictive Coding on the audio input. (Image source: van den Oord, et al. 2018)

CPC uses an encoder to compress the input data 
𝑧
𝑡
=
𝑔
enc
(
𝑥
𝑡
)
 and an autoregressive decoder to learn the high-level context that is potentially shared across future predictions, 
𝑐
𝑡
=
𝑔
ar
(
𝑧
≤
𝑡
)
. The end-to-end training relies on the NCE-inspired contrastive loss.

While predicting future information, CPC is optimized to maximize the the mutual information between input 
𝑥
 and context vector 
𝑐
:





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
⁡
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

Rather than modeling the future observations 
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
𝑓
𝑘
 can be unnormalized and a linear transformation 
𝑊
𝑘
⊤
𝑐
𝑡
 is used for the prediction with a different 
𝑊
𝑘
 matrix for every step 
𝑘
.

Given a set of 
𝑁
 random samples 
𝑋
=
{
𝑥
1
,
…
,
𝑥
𝑁
}
 containing only one positive sample 
𝑥
𝑡
∼
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
 and 
𝑁
−
1
 negative samples 
𝑥
𝑖
≠
𝑡
∼
𝑝
(
𝑥
𝑡
+
𝑘
)
, the cross-entropy loss for classifying the positive sample (where 
𝑓
𝑘
∑
𝑓
𝑘
 is the prediction) correctly is:

𝐿
𝑁
=
−
𝐸
𝑋
[
log
⁡
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
∑
𝑖
=
1
𝑁
𝑓
𝑘
(
𝑥
𝑖
,
𝑐
𝑡
)
]
Illustration of applying Contrastive Predictive Coding on images. (Image source: van den Oord, et al. 2018)

When using CPC on images (Henaff, et al. 2019), the predictor network should only access a masked feature set to avoid a trivial prediction. Precisely:

Each input image is divided into a set of overlapped patches and each patch is encoded by a resnet encoder, resulting in compressed feature vector 
𝑧
𝑖
,
𝑗
.
A masked conv net makes prediction with a mask such that the receptive field of a given output neuron can only see things above it in the image. Otherwise, the prediction problem would be trivial. The prediction can be made in both directions (top-down and bottom-up).
The prediction is made for 
𝑧
𝑖
+
𝑘
,
𝑗
 from context 
𝑐
𝑖
,
𝑗
: 
𝑧
^
𝑖
+
𝑘
,
𝑗
=
𝑊
𝑘
𝑐
𝑖
,
𝑗
.

A contrastive loss quantifies this prediction with a goal to correctly identify the target among a set of negative representation 
{
𝑧
𝑙
}
 sampled from other patches in the same image and other images in the same batch:





𝐿
CPC
=
−
∑
𝑖
,
𝑗
,
𝑘
log
⁡
𝑝
(
𝑧
𝑖
+
𝑘
,
𝑗
|
𝑧
^
𝑖
+
𝑘
,
𝑗
,
{
𝑧
𝑙
}
)
=
−
∑
𝑖
,
𝑗
,
𝑘
log
⁡
exp
⁡
(
𝑧
^
𝑖
+
𝑘
,
𝑗
⊤
𝑧
𝑖
+
𝑘
,
𝑗
)
exp
⁡
(
𝑧
^
𝑖
+
𝑘
,
𝑗
⊤
𝑧
𝑖
+
𝑘
,
𝑗
)
+
∑
𝑙
exp
⁡
(
𝑧
^
𝑖
+
𝑘
,
𝑗
⊤
𝑧
𝑙
)

For more content on contrastive learning, check out the post on “Contrastive Representation Learning”.

Video-Based

A video contains a sequence of semantically related frames. Nearby frames are close in time and more correlated than frames further away. The order of frames describes certain rules of reasonings and physical logics; such as that object motion should be smooth and gravity is pointing down.

A common workflow is to train a model on one or multiple pretext tasks with unlabelled videos and then feed one intermediate feature layer of this model to fine-tune a simple model on downstream tasks of action classification, segmentation or object tracking.

Tracking

The movement of an object is traced by a sequence of video frames. The difference between how the same object is captured on the screen in close frames is usually not big, commonly triggered by small motion of the object or the camera. Therefore any visual representation learned for the same object across close frames should be close in the latent feature space. Motivated by this idea, Wang & Gupta, 2015 proposed a way of unsupervised learning of visual representation by tracking moving objects in videos.

Precisely patches with motion are tracked over a small time window (e.g. 30 frames). The first patch 
𝑥
 and the last patch 
𝑥
+
 are selected and used as training data points. If we train the model directly to minimize the difference between feature vectors of two patches, the model may only learn to map everything to the same value. To avoid such a trivial solution, same as above, a random third patch 
𝑥
−
 is added. The model learns the representation by enforcing the distance between two tracked patches to be closer than the distance between the first patch and a random one in the feature space, 
𝐷
(
𝑥
,
𝑥
−
)
)
>
𝐷
(
𝑥
,
𝑥
+
)
, where 
𝐷
(
.
)
 is the cosine distance,

𝐷
(
𝑥
1
,
𝑥
2
)
=
1
−
𝑓
(
𝑥
1
)
𝑓
(
𝑥
2
)
‖
𝑓
(
𝑥
1
)
‖
‖
𝑓
(
𝑥
2
‖
)

The loss function is:

𝐿
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
max
(
0
,
𝐷
(
𝑥
,
𝑥
+
)
−
𝐷
(
𝑥
,
𝑥
−
)
+
𝑀
)
+
weight decay regularization term

where 
𝑀
 is a scalar constant controlling for the minimum gap between two distances; 
𝑀
=
0.5
 in the paper. The loss enforces 
𝐷
(
𝑥
,
𝑥
−
)
>=
𝐷
(
𝑥
,
𝑥
+
)
+
𝑀
 at the optimal case.

This form of loss function is also known as triplet loss in the face recognition task, in which the dataset contains images of multiple people from multiple camera angles. Let 
𝑥
𝑎
 be an anchor image of a specific person, 
𝑥
𝑝
 be a positive image of this same person from a different angle and 
𝑥
𝑛
 be a negative image of a different person. In the embedding space, 
𝑥
𝑎
 should be closer to 
𝑥
𝑝
 than 
𝑥
𝑛
:

𝐿
triplet
(
𝑥
𝑎
,
𝑥
𝑝
,
𝑥
𝑛
)
=
max
(
0
,
‖
𝜙
(
𝑥
𝑎
)
−
𝜙
(
𝑥
𝑝
)
‖
2
2
−
‖
𝜙
(
𝑥
𝑎
)
−
𝜙
(
𝑥
𝑛
)
‖
2
2
+
𝑀
)

A slightly different form of the triplet loss, named n-pair loss is also commonly used for learning observation embedding in robotics tasks. See a later section for more related content.

Overview of learning representation by tracking objects in videos. (a) Identify moving patches in short traces; (b) Feed two related patched and one random patch into a conv network with shared weights. (c) The loss function enforces the distance between related patches to be closer than the distance between random patches. (Image source: Wang & Gupta, 2015)

Relevant patches are tracked and extracted through a two-step unsupervised optical flow approach:

Obtain SURF interest points and use IDT to obtain motion of each SURF point.
Given the trajectories of SURF interest points, classify these points as moving if the flow magnitude is more than 0.5 pixels.

During training, given a pair of correlated patches 
𝑥
 and 
𝑥
+
, 
𝐾
 random patches 
{
𝑥
−
}
 are sampled in this same batch to form 
𝐾
 training triplets. After a couple of epochs, hard negative mining is applied to make the training harder and more efficient, that is, to search for random patches that maximize the loss and use them to do gradient updates.

Frame Sequence

Video frames are naturally positioned in chronological order. Researchers have proposed several self-supervised tasks, motivated by the expectation that good representation should learn the correct sequence of frames.

One idea is to validate frame order (Misra, et al 2016). The pretext task is to determine whether a sequence of frames from a video is placed in the correct temporal order (“temporal valid”). The model needs to track and reason about small motion of an object across frames to complete such a task.

The training frames are sampled from high-motion windows. Every time 5 frames are sampled 
(
𝑓
𝑎
,
𝑓
𝑏
,
𝑓
𝑐
,
𝑓
𝑑
,
𝑓
𝑒
)
 and the timestamps are in order 
𝑎
<
𝑏
<
𝑐
<
𝑑
<
𝑒
. Out of 5 frames, one positive tuple 
(
𝑓
𝑏
,
𝑓
𝑐
,
𝑓
𝑑
)
 and two negative tuples, 
(
𝑓
𝑏
,
𝑓
𝑎
,
𝑓
𝑑
)
 and 
(
𝑓
𝑏
,
𝑓
𝑒
,
𝑓
𝑑
)
 are created. The parameter 
𝜏
max
=
|
𝑏
−
𝑑
|
 controls the difficulty of positive training instances (i.e. higher → harder) and the parameter 
𝜏
min
=
min
(
|
𝑎
−
𝑏
|
,
|
𝑑
−
𝑒
|
)
 controls the difficulty of negatives (i.e. lower → harder).

The pretext task of video frame order validation is shown to improve the performance on the downstream task of action recognition when used as a pretraining step.

Overview of learning representation by validating the order of video frames. (a) the data sample process; (b) the model is a triplet siamese network, where all input frames have shared weights. (Image source: Misra, et al 2016)

The task in O3N (Odd-One-Out Network; Fernando et al. 2017) is based on video frame sequence validation too. One step further from above, the task is to pick the incorrect sequence from multiple video clips.

Given 
𝑁
+
1
 input video clips, one of them has frames shuffled, thus in the wrong order, and the rest 
𝑁
 of them remain in the correct temporal order. O3N learns to predict the location of the odd video clip. In their experiments, there are 6 input clips and each contain 6 frames.

The arrow of time in a video contains very informative messages, on both low-level physics (e.g. gravity pulls objects down to the ground; smoke rises up; water flows downward.) and high-level event reasoning (e.g. fish swim forward; you can break an egg but cannot revert it.). Thus another idea is inspired by this to learn latent representation by predicting the arrow of time (AoT) — whether video playing forwards or backwards (Wei et al., 2018).

A classifier should capture both low-level physics and high-level semantics in order to predict the arrow of time. The proposed T-CAM (Temporal Class-Activation-Map) network accepts 
𝑇
 groups, each containing a number of frames of optical flow. The conv layer outputs from each group are concatenated and fed into binary logistic regression for predicting the arrow of time.

Overview of learning representation by predicting the arrow of time. (a) Conv features of multiple groups of frame sequences are concatenated. (b) The top level contains 3 conv layers and average pooling. (Image source: Wei et al, 2018)

Interestingly, there exist a couple of artificial cues in the dataset. If not handled properly, they could lead to a trivial classifier without relying on the actual video content:

Due to the video compression, the black framing might not be completely black but instead may contain certain information on the chronological order. Hence black framing should be removed in the experiments.
Large camera motion, like vertical translation or zoom-in/out, also provides strong signals for the arrow of time but independent of content. The processing stage should stabilize the camera motion.

The AoT pretext task is shown to improve the performance on action classification downstream task when used as a pretraining step. Note that fine-tuning is still needed.

Video Colorization

Vondrick et al. (2018) proposed video colorization as a self-supervised learning problem, resulting in a rich representation that can be used for video segmentation and unlabelled visual region tracking, without extra fine-tuning.

Unlike the image-based colorization, here the task is to copy colors from a normal reference frame in color to another target frame in grayscale by leveraging the natural temporal coherency of colors across video frames (thus these two frames shouldn’t be too far apart in time). In order to copy colors consistently, the model is designed to learn to keep track of correlated pixels in different frames.

Video colorization by copying colors from a reference frame to target frames in grayscale. (Image source: Vondrick et al. 2018)

The idea is quite simple and smart. Let 
𝑐
𝑖
 be the true color of the 
𝑖
−
𝑡
ℎ
 pixel in the reference frame and 
𝑐
𝑗
 be the color of 
𝑗
-th pixel in the target frame. The predicted color of 
𝑗
-th color in the target 
𝑐
^
𝑗
 is a weighted sum of colors of all the pixels in reference, where the weighting term measures the similarity:



𝑐
^
𝑗
=
∑
𝑖
𝐴
𝑖
𝑗
𝑐
𝑖
 where 
𝐴
𝑖
𝑗
=
exp
⁡
(
𝑓
𝑖
𝑓
𝑗
)
∑
𝑖
′
exp
⁡
(
𝑓
𝑖
′
𝑓
𝑗
)

where 
𝑓
 are learned embeddings for corresponding pixels; 
𝑖
′
 indexes all the pixels in the reference frame. The weighting term implements an attention-based pointing mechanism, similar to matching network and pointer network. As the full similarity matrix could be really large, both frames are downsampled. The categorical cross-entropy loss between 
𝑐
𝑗
 and 
𝑐
^
𝑗
 is used with quantized colors, just like in Zhang et al. 2016.

Based on how the reference frame are marked, the model can be used to complete several color-based downstream tasks such as tracking segmentation or human pose in time. No fine-tuning is needed. See

Use video colorization to track object segmentation and human pose in time. (Image source: Vondrick et al. (2018))

A couple common observations:

Combining multiple pretext tasks improves performance;
Deeper networks improve the quality of representation;
Supervised learning baselines still beat all of them by far.
Control-Based

When running a RL policy in the real world, such as controlling a physical robot on visual inputs, it is non-trivial to properly track states, obtain reward signals or determine whether a goal is achieved for real. The visual data has a lot of noise that is irrelevant to the true state and thus the equivalence of states cannot be inferred from pixel-level comparison. Self-supervised representation learning has shown great potential in learning useful state embedding that can be used directly as input to a control policy.

All the cases discussed in this section are in robotic learning, mainly for state representation from multiple camera views and goal representation.

Multi-View Metric Learning

The concept of metric learning has been mentioned multiple times in the previous sections. A common setting is: Given a triple of samples, (anchor 
𝑠
𝑎
, positive sample 
𝑠
𝑝
, negative sample 
𝑠
𝑛
), the learned representation embedding 
𝜙
(
𝑠
)
 fulfills that 
𝑠
𝑎
 stays close to 
𝑠
𝑝
 but far away from 
𝑠
𝑛
 in the latent space.

Grasp2Vec (Jang & Devin et al., 2018) aims to learn an object-centric vision representation in the robot grasping task from free, unlabelled grasping activities. By object-centric, it means that, irrespective of how the environment or the robot looks like, if two images contain similar items, they should be mapped to similar representation; otherwise the embeddings should be far apart.

A conceptual illustration of how grasp2vec learns an object-centric state embedding. (Image source: Jang & Devin et al., 2018)

The grasping system can tell whether it moves an object but cannot tell which object it is. Cameras are set up to take images of the entire scene and the grasped object. During early training, the grasp robot is executed to grasp any object 
𝑜
 at random, producing a triple of images, 
(
𝑠
pre
,
𝑠
post
,
𝑜
)
:

𝑜
 is an image of the grasped object held up to the camera;
𝑠
pre
 is an image of the scene before grasping, with the object 
𝑜
 in the tray;
𝑠
post
 is an image of the same scene after grasping, without the object 
𝑜
 in the tray.

To learn object-centric representation, we expect the difference between embeddings of 
𝑠
pre
 and 
𝑠
post
 to capture the removed object 
𝑜
. The idea is quite interesting and similar to relationships that have been observed in word embedding, e.g. distance(“king”, “queen”) ≈ distance(“man”, “woman”).

Let 
𝜙
𝑠
 and 
𝜙
𝑜
 be the embedding functions for the scene and the object respectively. The model learns the representation by minimizing the distance between 
𝜙
𝑠
(
𝑠
pre
)
−
𝜙
𝑠
(
𝑠
post
)
 and 
𝜙
𝑜
(
𝑜
)
 using n-pair loss:

	
	

𝐿
grasp2vec
	
=
NPair
(
𝜙
𝑠
(
𝑠
pre
)
−
𝜙
𝑠
(
𝑠
post
)
,
𝜙
𝑜
(
𝑜
)
)
+
NPair
(
𝜙
𝑜
(
𝑜
)
,
𝜙
𝑠
(
𝑠
pre
)
−
𝜙
𝑠
(
𝑠
post
)
)


where 
NPair
(
𝑎
,
𝑝
)
	
=
∑
𝑖
<
𝐵
−
log
⁡
exp
⁡
(
𝑎
𝑖
⊤
𝑝
𝑗
)
∑
𝑗
<
𝐵
,
𝑖
≠
𝑗
exp
⁡
(
𝑎
𝑖
⊤
𝑝
𝑗
)
+
𝜆
(
‖
𝑎
𝑖
‖
2
2
+
‖
𝑝
𝑖
‖
2
2
)

where 
𝐵
 refers to a batch of (anchor, positive) sample pairs.

When framing representation learning as metric learning, n-pair loss is a common choice. Rather than processing explicit a triple of (anchor, positive, negative) samples, the n-pairs loss treats all other positive instances in one mini-batch across pairs as negatives.

The embedding function 
𝜙
𝑜
 works great for presenting a goal 
𝑔
 with an image. The reward function that quantifies how close the actually grasped object 
𝑜
 is close to the goal is defined as 
𝑟
=
𝜙
𝑜
(
𝑔
)
⋅
𝜙
𝑜
(
𝑜
)
. Note that computing rewards only relies on the learned latent space and doesn’t involve ground truth positions, so it can be used for training on real robots.

Localization results of grasp2vec embedding. The heatmap of localizing a goal object in a pre-grasping scene is defined as 
𝜙
_
𝑜
(
𝑜
)
⊤
𝜙
_
𝑠
,
spatial
(
𝑠
_
pre
)
, where 
𝜙
_
𝑠
,
spatial
 is the output of the last resnet block after ReLU. The fourth column is a failure case and the last three columns take real images as goals. (Image source: Jang & Devin et al., 2018)

Other than the embedding-similarity-based reward function, there are a few other tricks for training the RL policy in the grasp2vec framework:

Posthoc labeling: Augment the dataset by labeling a randomly grasped object as a correct goal, like HER (Hindsight Experience Replay; Andrychowicz, et al., 2017).
Auxiliary goal augmentation: Augment the replay buffer even further by relabeling transitions with unachieved goals; precisely, in each iteration, two goals are sampled 
(
𝑔
,
𝑔
′
)
 and both are used to add new transitions into replay buffer.

TCN (Time-Contrastive Networks; Sermanet, et al. 2018) learn from multi-camera view videos with the intuition that different viewpoints at the same timestep of the same scene should share the same embedding (like in FaceNet) while embedding should vary in time, even of the same camera viewpoint. Therefore embedding captures the semantic meaning of the underlying state rather than visual similarity. The TCN embedding is trained with triplet loss.

The training data is collected by taking videos of the same scene simultaneously but from different angles. All the videos are unlabelled.

An illustration of time-contrastive approach for learning state embedding. The blue frames selected from two camera views at the same timestep are anchor and positive samples, while the red frame at a different timestep is the negative sample.

TCN embedding extracts visual features that are invariant to camera configurations. It can be used to construct a reward function for imitation learning based on the euclidean distance between the demo video and the observations in the latent space.

A further improvement over TCN is to learn embedding over multiple frames jointly rather than a single frame, resulting in mfTCN (Multi-frame Time-Contrastive Networks; Dwibedi et al., 2019). Given a set of videos from several synchronized camera viewpoints, 
𝑣
1
,
𝑣
2
,
…
,
𝑣
𝑘
, the frame at time 
𝑡
 and the previous 
𝑛
−
1
 frames selected with stride 
𝑠
 in each video are aggregated and mapped into one embedding vector, resulting in a lookback window of size 
(
𝑛
−
1
)
×
𝑠
+
1
. Each frame first goes through a CNN to extract low-level features and then we use 3D temporal convolutions to aggregate frames in time. The model is trained with n-pairs loss.

The sampling process for training mfTCN. (Image source: Dwibedi et al., 2019)

The training data is sampled as follows:

First we construct two pairs of video clips. Each pair contains two clips from different camera views but with synchronized timesteps. These two sets of videos should be far apart in time.
Sample a fixed number of frames from each video clip in the same pair simultaneously with the same stride.
Frames with the same timesteps are trained as positive samples in the n-pair loss, while frames across pairs are negative samples.

mfTCN embedding can capture the position and velocity of objects in the scene (e.g. in cartpole) and can also be used as inputs for policy.

Autonomous Goal Generation

RIG (Reinforcement learning with Imagined Goals; Nair et al., 2018) described a way to train a goal-conditioned policy with unsupervised representation learning. A policy learns from self-supervised practice by first imagining “fake” goals and then trying to achieve them.

The workflow of RIG. (Image source: Nair et al., 2018)

The task is to control a robot arm to push a small puck on a table to a desired position. The desired position, or the goal, is present in an image. During training, it learns latent embedding of both state 
𝑠
 and goal 
𝑔
 through 
𝛽
-VAE encoder and the control policy operates entirely in the latent space.

Let’s say a 
𝛽
-VAE has an encoder 
𝑞
𝜙
 mapping input states to latent variable 
𝑧
 which is modeled by a Gaussian distribution and a decoder 
𝑝
𝜓
 mapping 
𝑧
 back to the states. The state encoder in RIG is set to be the mean of 
𝛽
-VAE encoder.

	

	
	
𝑧
	
∼
𝑞
𝜙
(
𝑧
|
𝑠
)
=
𝑁
(
𝑧
;
𝜇
𝜙
(
𝑠
)
,
𝜎
𝜙
2
(
𝑠
)
)


𝐿
𝛽
-VAE
	
=
−
𝐸
𝑧
∼
𝑞
𝜙
(
𝑧
|
𝑠
)
[
log
⁡
𝑝
𝜓
(
𝑠
|
𝑧
)
]
+
𝛽
𝐷
KL
(
𝑞
𝜙
(
𝑧
|
𝑠
)
‖
𝑝
𝜓
(
𝑠
)
)


𝑒
(
𝑠
)
	
≜
𝜇
𝜙
(
𝑠
)

The reward is the Euclidean distance between state and goal embedding vectors: 
𝑟
(
𝑠
,
𝑔
)
=
−
|
𝑒
(
𝑠
)
−
𝑒
(
𝑔
)
|
. Similar to grasp2vec, RIG applies data augmentation as well by latent goal relabeling: precisely half of the goals are generated from the prior at random and the other half are selected using HER. Also same as grasp2vec, rewards do not depend on any ground truth states but only the learned state encoding, so it can be used for training on real robots.

The algorithm of RIG. (Image source: Nair et al., 2018)

The problem with RIG is a lack of object variations in the imagined goal pictures. If 
𝛽
-VAE is only trained with a black puck, it would not be able to create a goal with other objects like blocks of different shapes and colors. A follow-up improvement replaces 
𝛽
-VAE with a CC-VAE (Context-Conditioned VAE; Nair, et al., 2019), inspired by CVAE (Conditional VAE; Sohn, Lee & Yan, 2015), for goal generation.

The workflow of context-conditioned RIG. (Image source: Nair, et al., 2019).

A CVAE conditions on a context variable 
𝑐
. It trains an encoder 
𝑞
𝜙
(
𝑧
|
𝑠
,
𝑐
)
 and a decoder 
𝑝
𝜓
(
𝑠
|
𝑧
,
𝑐
)
 and note that both have access to 
𝑐
. The CVAE loss penalizes information passing from the input state 
𝑠
 through an information bottleneck but allows for unrestricted information flow from 
𝑐
 to both encoder and decoder.

𝐿
CVAE
=
−
𝐸
𝑧
∼
𝑞
𝜙
(
𝑧
|
𝑠
,
𝑐
)
[
log
⁡
𝑝
𝜓
(
𝑠
|
𝑧
,
𝑐
)
]
+
𝛽
𝐷
KL
(
𝑞
𝜙
(
𝑧
|
𝑠
,
𝑐
)
‖
𝑝
𝜓
(
𝑠
)
)

To create plausible goals, CC-VAE conditions on a starting state 
𝑠
0
 so that the generated goal presents a consistent type of object as in 
𝑠
0
. This goal consistency is necessary; e.g. if the current scene contains a red puck but the goal has a blue block, it would confuse the policy.

Other than the state encoder 
𝑒
(
𝑠
)
≜
𝜇
𝜙
(
𝑠
)
, CC-VAE trains a second convolutional encoder 
𝑒
0
(
.
)
 to translate the starting state 
𝑠
0
 into a compact context representation 
𝑐
=
𝑒
0
(
𝑠
0
)
. Two encoders, 
𝑒
(
.
)
 and 
𝑒
0
(
.
)
, are intentionally different without shared weights, as they are expected to encode different factors of image variation. In addition to the loss function of CVAE, CC-VAE adds an extra term to learn to reconstruct 
𝑐
 back to 
𝑠
0
, 
𝑠
^
0
=
𝑑
0
(
𝑐
)
.

𝐿
CC-VAE
=
𝐿
CVAE
+
log
⁡
𝑝
(
𝑠
0
|
𝑐
)
Examples of imagined goals generated by CVAE that conditions on the context image (the first row), while VAE fails to capture the object consistency. (Image source: Nair, et al., 2019).
Bisimulation

Task-agnostic representation (e.g. a model that intends to represent all the dynamics in the system) may distract the RL algorithms as irrelevant information is also presented. For example, if we just train an auto-encoder to reconstruct the input image, there is no guarantee that the entire learned representation will be useful for RL. Therefore, we need to move away from reconstruction-based representation learning if we only want to learn information relevant to control, as irrelevant details are still important for reconstruction.

Representation learning for control based on bisimulation does not depend on reconstruction, but aims to group states based on their behavioral similarity in MDP.

Bisimulation (Givan et al. 2003) refers to an equivalence relation between two states with similar long-term behavior. Bisimulation metrics quantify such relation so that we can aggregate states to compress a high-dimensional state space into a smaller one for more efficient computation. The bisimulation distance between two states corresponds to how behaviorally different these two states are.

Given a MDP 
𝑀
=
⟨
𝑆
,
𝐴
,
𝑃
,
𝑅
,
𝛾
⟩
 and a bisimulation relation 
𝐵
, two states that are equal under relation 
𝐵
 (i.e. 
𝑠
𝑖
𝐵
𝑠
𝑗
) should have the same immediate reward for all actions and the same transition probabilities over the next bisimilar states:

	
	
𝑅
(
𝑠
𝑖
,
𝑎
)
	
=
𝑅
(
𝑠
𝑗
,
𝑎
)
∀
𝑎
∈
𝐴


𝑃
(
𝐺
|
𝑠
𝑖
,
𝑎
)
	
=
𝑃
(
𝐺
|
𝑠
𝑗
,
𝑎
)
∀
𝑎
∈
𝐴
∀
𝐺
∈
𝑆
𝐵

where 
𝑆
𝐵
 is a partition of the state space under the relation 
𝐵
.

Note that 
=
 is always a bisimulation relation. The most interesting one is the maximal bisimulation relation 
∼
, which defines a partition 
𝑆
∼
 with fewest groups of states.

DeepMDP learns a latent space model by minimizing two losses on a reward model and a dynamics model. (Image source: Gelada, et al. 2019)

With a goal similar to bisimulation metric, DeepMDP (Gelada, et al. 2019) simplifies high-dimensional observations in RL tasks and learns a latent space model via minimizing two losses:

prediction of rewards and
prediction of the distribution over next latent states.


𝐿
𝑅
¯
(
𝑠
,
𝑎
)
=
|
𝑅
(
𝑠
,
𝑎
)
−
𝑅
¯
(
𝜙
(
𝑠
)
,
𝑎
)
|


𝐿
𝑃
¯
(
𝑠
,
𝑎
)
=
𝐷
(
𝜙
𝑃
(
𝑠
,
𝑎
)
,
𝑃
¯
(
.
|
𝜙
(
𝑠
)
,
𝑎
)
)

where 
𝜙
(
𝑠
)
 is the embedding of state 
𝑠
; symbols with bar are functions (reward function 
𝑅
 and transition function 
𝑃
) in the same MDP but running in the latent low-dimensional observation space. Here the embedding representation 
𝜙
 can be connected to bisimulation metrics, as the bisimulation distance is proved to be upper-bounded by the L2 distance in the latent space.

The function 
𝐷
 quantifies the distance between two probability distributions and should be chosen carefully. DeepMDP focuses on Wasserstein-1 metric (also known as “earth-mover distance”). The Wasserstein-1 distance between distributions 
𝑃
 and 
𝑄
 on a metric space 
(
𝑀
,
𝑑
)
 (i.e., 
𝑑
:
𝑀
×
𝑀
→
𝑅
) is:



𝑊
𝑑
(
𝑃
,
𝑄
)
=
inf
𝜆
∈
Π
(
𝑃
,
𝑄
)
∫
𝑀
×
𝑀
𝑑
(
𝑥
,
𝑦
)
𝜆
(
𝑥
,
𝑦
)
d
𝑥
d
𝑦

where 
Π
(
𝑃
,
𝑄
)
 is the set of all couplings of 
𝑃
 and 
𝑄
. 
𝑑
(
𝑥
,
𝑦
)
 defines the cost of moving a particle from point 
𝑥
 to point 
𝑦
.

The Wasserstein metric has a dual form according to the Monge-Kantorovich duality:



𝑊
𝑑
(
𝑃
,
𝑄
)
=
sup
𝑓
∈
𝐹
𝑑
|
𝐸
𝑥
∼
𝑃
𝑓
(
𝑥
)
−
𝐸
𝑦
∼
𝑄
𝑓
(
𝑦
)
|

where 
𝐹
𝑑
 is the set of 1-Lipschitz functions under the metric 
𝑑
 - 
𝐹
𝑑
=
{
𝑓
:
|
𝑓
(
𝑥
)
−
𝑓
(
𝑦
)
|
≤
𝑑
(
𝑥
,
𝑦
)
}
.

DeepMDP generalizes the model to the Norm Maximum Mean Discrepancy (Norm-MMD) metrics to improve the tightness of the bounds of its deep value function and, at the same time, to save computation (Wasserstein is expensive computationally). In their experiments, they found the model architecture of the transition prediction model can have a big impact on the performance. Adding these DeepMDP losses as auxiliary losses when training model-free RL agents leads to good improvement on most of the Atari games.

Deep Bisimulatioin for Control (short for DBC; Zhang et al. 2020) learns the latent representation of observations that are good for control in RL tasks, without domain knowledge or pixel-level reconstruction.

The Deep Bisimulation for Control algorithm learns a bisimulation metric representation via learning a reward model and a dynamics model. The model architecture is a siamese network. (Image source: Zhang et al. 2020)

Similar to DeepMDP, DBC models the dynamics by learning a reward model and a transition model. Both models operate in the latent space, 
𝜙
(
𝑠
)
. The optimization of embedding 
𝜙
 depends on one important conclusion from Ferns, et al. 2004 (Theorem 4.5) and Ferns, et al 2011 (Theorem 2.6):

Given 
𝑐
∈
(
0
,
1
)
 a discounting factor, 
𝜋
 a policy that is being improved continuously, and 
𝑀
 the space of bounded pseudometric on the state space 
𝑆
, we can define 
𝐹
:
𝑀
↦
𝑀
:

𝐹
(
𝑑
;
𝜋
)
(
𝑠
𝑖
,
𝑠
𝑗
)
=
(
1
−
𝑐
)
|
𝑅
𝑠
𝑖
𝜋
−
𝑅
𝑠
𝑗
𝜋
|
+
𝑐
𝑊
𝑑
(
𝑃
𝑠
𝑖
𝜋
,
𝑃
𝑠
𝑗
𝜋
)

Then, 
𝐹
 has a unique fixed point 
𝑑
~
 which is a 
𝜋
∗
-bisimulation metric and 
𝑑
~
(
𝑠
𝑖
,
𝑠
𝑗
)
=
0
⟺
𝑠
𝑖
∼
𝑠
𝑗
.

[The proof is not trivial. I may or may not add it in the future _(:3」∠)_ …]

Given batches of observations pairs, the training loss for 
𝜙
, 
𝐽
(
𝜙
)
, minimizes the mean square error between the on-policy bisimulation metric and Euclidean distance in the latent space:

𝐽
(
𝜙
)
=
(
‖
𝜙
(
𝑠
𝑖
)
−
𝜙
(
𝑠
𝑗
)
‖
1
−
|
𝑅
^
(
𝜙
¯
(
𝑠
𝑖
)
)
−
𝑅
^
(
𝜙
¯
(
𝑠
𝑗
)
)
|
−
𝛾
𝑊
2
(
𝑃
^
(
⋅
|
𝜙
¯
(
𝑠
𝑖
)
,
𝜋
¯
(
𝜙
¯
(
𝑠
𝑖
)
)
)
,
𝑃
^
(
⋅
|
𝜙
¯
(
𝑠
𝑗
)
,
𝜋
¯
(
𝜙
¯
(
𝑠
𝑗
)
)
)
)
)
2

where 
𝜙
¯
(
𝑠
)
 denotes 
𝜙
(
𝑠
)
 with stop gradient and 
𝜋
¯
 is the mean policy output. The learned reward model 
𝑅
^
 is deterministic and the learned forward dynamics model 
𝑃
^
 outputs a Gaussian distribution.

DBC is based on SAC but operates on the latent space:

The algorithm of Deep Bisimulation for Control. (Image source: Zhang et al. 2020)

Cited as:

@article{weng2019selfsup,
  title   = "Self-Supervised Representation Learning",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2019",
  url     = "https://lilianweng.github.io/posts/2019-11-10-self-supervised/"
}

References

[1] Alexey Dosovitskiy, et al. “Discriminative unsupervised feature learning with exemplar convolutional neural networks.” IEEE transactions on pattern analysis and machine intelligence 38.9 (2015): 1734-1747.

[2] Spyros Gidaris, Praveer Singh & Nikos Komodakis. “Unsupervised Representation Learning by Predicting Image Rotations” ICLR 2018.

[3] Carl Doersch, Abhinav Gupta, and Alexei A. Efros. “Unsupervised visual representation learning by context prediction.” ICCV. 2015.

[4] Mehdi Noroozi & Paolo Favaro. “Unsupervised learning of visual representations by solving jigsaw puzzles.” ECCV, 2016.

[5] Mehdi Noroozi, Hamed Pirsiavash, and Paolo Favaro. “Representation learning by learning to count.” ICCV. 2017.

[6] Richard Zhang, Phillip Isola & Alexei A. Efros. “Colorful image colorization.” ECCV, 2016.

[7] Pascal Vincent, et al. “Extracting and composing robust features with denoising autoencoders.” ICML, 2008.

[8] Jeff Donahue, Philipp Krähenbühl, and Trevor Darrell. “Adversarial feature learning.” ICLR 2017.

[9] Deepak Pathak, et al. “Context encoders: Feature learning by inpainting.” CVPR. 2016.

[10] Richard Zhang, Phillip Isola, and Alexei A. Efros. “Split-brain autoencoders: Unsupervised learning by cross-channel prediction.” CVPR. 2017.

[11] Xiaolong Wang & Abhinav Gupta. “Unsupervised Learning of Visual Representations using Videos.” ICCV. 2015.

[12] Carl Vondrick, et al. “Tracking Emerges by Colorizing Videos” ECCV. 2018.

[13] Ishan Misra, C. Lawrence Zitnick, and Martial Hebert. “Shuffle and learn: unsupervised learning using temporal order verification.” ECCV. 2016.

[14] Basura Fernando, et al. “Self-Supervised Video Representation Learning With Odd-One-Out Networks” CVPR. 2017.

[15] Donglai Wei, et al. “Learning and Using the Arrow of Time” CVPR. 2018.

[16] Florian Schroff, Dmitry Kalenichenko and James Philbin. “FaceNet: A Unified Embedding for Face Recognition and Clustering” CVPR. 2015.

[17] Pierre Sermanet, et al. “Time-Contrastive Networks: Self-Supervised Learning from Video” CVPR. 2018.

[18] Debidatta Dwibedi, et al. “Learning actionable representations from visual observations.” IROS. 2018.

[19] Eric Jang & Coline Devin, et al. “Grasp2Vec: Learning Object Representations from Self-Supervised Grasping” CoRL. 2018.

[20] Ashvin Nair, et al. “Visual reinforcement learning with imagined goals” NeuriPS. 2018.

[21] Ashvin Nair, et al. “Contextual imagined goals for self-supervised robotic learning” CoRL. 2019.

[22] Aaron van den Oord, Yazhe Li & Oriol Vinyals. “Representation Learning with Contrastive Predictive Coding” arXiv preprint arXiv:1807.03748, 2018.

[23] Olivier J. Henaff, et al. “Data-Efficient Image Recognition with Contrastive Predictive Coding” arXiv preprint arXiv:1905.09272, 2019.

[24] Kaiming He, et al. “Momentum Contrast for Unsupervised Visual Representation Learning.” CVPR 2020.

[25] Zhirong Wu, et al. “Unsupervised Feature Learning via Non-Parametric Instance-level Discrimination.” CVPR 2018.

[26] Ting Chen, et al. “A Simple Framework for Contrastive Learning of Visual Representations.” arXiv preprint arXiv:2002.05709, 2020.

[27] Aravind Srinivas, Michael Laskin & Pieter Abbeel “CURL: Contrastive Unsupervised Representations for Reinforcement Learning.” arXiv preprint arXiv:2004.04136, 2020.

[28] Carles Gelada, et al. “DeepMDP: Learning Continuous Latent Space Models for Representation Learning” ICML 2019.

[29] Amy Zhang, et al. “Learning Invariant Representations for Reinforcement Learning without Reconstruction” arXiv preprint arXiv:2006.10742, 2020.

[30] Xinlei Chen, et al. “Improved Baselines with Momentum Contrastive Learning” arXiv preprint arXiv:2003.04297, 2020.

[31] Jean-Bastien Grill, et al. “Bootstrap Your Own Latent: A New Approach to Self-Supervised Learning” arXiv preprint arXiv:2006.07733, 2020.

[32] Abe Fetterman & Josh Albrecht. “Understanding self-supervised and contrastive learning with Bootstrap Your Own Latent (BYOL)” Untitled blog. Aug 24, 2020.

Representation-Learning
 
Long-Read
 
Generative-Model
 
Object-Recognition
 
Reinforcement-Learning
 
Unsupervised-Learning
«
Curriculum for Reinforcement Learning
»
Evolution Strategies
© 2026 Lil'Log Powered by Hugo & PaperMod