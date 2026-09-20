---
title: Anatomize Deep Learning with Information Theory
url: https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:48:42.911751+00:00'
---

Professor Naftali Tishby passed away in 2021. Hope the post can introduce his cool idea of information bottleneck to more people.

Recently I watched the talk [“Information Theory in Deep Learning”](https://youtu.be/bLqJHjXihK8) by Prof Naftali Tishby and found it very interesting. He presented how to apply the information theory to study the growth and transformation of deep neural networks during training. Using the [Information Bottleneck (IB)](https://arxiv.org/pdf/physics/0004057.pdf) method, he proposed a new learning bound for deep neural networks (DNN), as the traditional learning theory fails due to the exponentially large number of parameters. Another keen observation is that DNN training involves two distinct phases: First, the network is trained to fully represent the input data and minimize the generalization error; then, it learns to forget the irrelevant details by compressing the representation of the input.

Most of the materials in this post are from Prof Tishby’s talk and [related papers](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/#references).

## Basic Concepts

[**Markov Chain**](https://en.wikipedia.org/wiki/Markov_chain)

A Markov process is a [“memoryless”](http://mathworld.wolfram.com/Memoryless.html) (also called “Markov Property”) stochastic process. A Markov chain is a type of Markov process containing multiple discrete states. That is being said, the conditional probability of future states of the process is only determined by the current state and does not depend on the past states.

[**Kullback–Leibler (KL) Divergence**](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence)

KL divergence measures how one probability distribution diverges from a second expected probability distribution . It is asymmetric.

achieves the minimum zero when == everywhere.

[**Mutual Information**](https://en.wikipedia.org/wiki/Mutual_information)

Mutual information measures the mutual dependence between two variables. It quantifies the “amount of information” obtained about one random variable through the other random variable. Mutual information is symmetric.

[**Data Processing Inequality (DPI)**](https://en.wikipedia.org/wiki/Data_processing_inequality)

For any markov chain: , we would have .

A deep neural network can be viewed as a Markov chain, and thus when we are moving down the layers of a DNN, the mutual information between the layer and the input can only decrease.

[**Reparametrization invariance**](https://en.wikipedia.org/wiki/Parametrization#Parametrization_invariance)

For two invertible functions , , the mutual information still holds: .

For example, if we shuffle the weights in one layer of DNN, it would not affect the mutual information between this layer and another.

## Deep Neural Networks as Markov Chains

The training data contains sampled observations from the joint distribution of and . The input variable and weights of hidden layers are all high-dimensional random variable. The ground truth target and the predicted value are random variables of smaller dimensions in the classification settings.

![Image 1](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-dnn-structure.png)

The structure of a deep neural network, which consists of the target label , input layer , hidden layers and the final prediction . (Image source: [Tishby and Zaslavsky, 2015](https://arxiv.org/pdf/1503.02406.pdf))

If we label the hidden layers of a DNN as as in Fig. 1, we can view each layer as one state of a Markov Chain: . According to DPI, we would have:

A DNN is designed to learn how to describe to predict and eventually, to compress to only hold the information related to . Tishby describes this processing as _“successive refinement of relevant information”_.

## Information Plane Theorem

A DNN has successive internal representations of , a set of hidden layers . The _information plane_ theorem characterizes each layer by its encoder and decoder information. The encoder is a representation of the input data , while the decoder translates the information in the current layer to the target ouput .

Precisely, in an information plane plot:

*   **X-axis**: The sample complexity of is determined by the encoder mutual information . Sample complexity refers to how many samples you need to achieve certain accuracy and generalization.
*   **Y-axis**: The accuracy (generalization error) is determined by the decoder mutual information .

![Image 2](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-information-plane.png)

The encoder vs decoder mutual information of DNN hidden layers of 50 experiments. Different layers are color-coders, with green being the layer right next to the input and the orange being the furthest. There are three snapshots, at the initial epoch, 400 epochs and 9000 epochs respectively. (Image source: [Shwartz-Ziv and Tishby, 2017](https://arxiv.org/pdf/1703.00810.pdf))

Each dot in marks the encoder/ decoder mutual information of one hidden layer of one network simulation (no regularization is applied; no weights decay, no dropout, etc.). They move up as expected because the knowledge about the true labels is increasing (accuracy increases). At the early stage, the hidden layers learn a lot about the input , but later they start to compress to forget some information about the input. Tishby believes that _“the most important part of learning is actually forgetting”_. Check out this [nice video](https://youtu.be/P1A1yNsxMjc) that demonstrates how the mutual information measures of layers are changing in epoch time.

![Image 3](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-information-plane-merged.png)

Here is an aggregated view of Fig 2. The compression happens after the generalization error becomes very small. (Image source: [Tishby’ talk 15:15](https://youtu.be/bLqJHjXihK8?t=15m15s))

## Two Optimization Phases

Tracking the normalized mean and standard deviation of each layer’s weights in time also reveals two optimization phases of the training process.

![Image 4](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-mean-variation.png)

The norm of mean and standard deviation of each layer's weight gradients for each layer as a function of training epochs. Different layers are color-coded. (Image source: [Shwartz-Ziv and Tishby, 2017](https://arxiv.org/pdf/1703.00810.pdf))

Among early epochs, the mean values are three magnitudes larger than the standard deviations. After a sufficient number of epochs, the error saturates and the standard deviations become much noisier afterward. The further a layer is away from the output, the noisier it gets, because the noises can get amplified and accumulated through the back-prop process (not due to the width of the layer).

## “Old” Generalization Bounds

The generalization bounds defined by the classic learning theory is:

*   : The difference between the training error and the generalization error. The generalization error measures how accurate the prediction of an algorithm is for previously unseen data.
*   : -cover of the hypothesis class. Typically we assume the size .
*   : Confidence.
*   : The number of training samples.
*   : The VC dimension of the hypothesis.

This definition states that the difference between the training error and the generalization error is bounded by a function of the hypothesis space size and the dataset size. The bigger the hypothesis space gets, the bigger the generalization error becomes. I recommend this tutorial on ML theory, [part1](https://mostafa-samir.github.io/ml-theory-pt1/) and [part2](https://mostafa-samir.github.io/ml-theory-pt2/), if you are interested in reading more on generalization bounds.

However, it does not work for deep learning. The larger a network is, the more parameters it needs to learn. With this generalization bounds, larger networks (larger ) would have worse bounds. This is contrary to the intuition that larger networks are able to achieve better performance with higher expressivity.

## “New” Input compression bound

To solve this counterintuitive observation, Tishby et al. proposed a new input compression bound for DNN.

First let us have as an -partition of the input variable . This partition compresses the input with respect to the homogeneity to the labels into small cells. The cells in total can cover the whole input space. If the prediction outputs binary values, we can replace the cardinality of the hypothesis, , with .

When is large, the size of is approximately . Each cell in the -partition is of size . Therefore we have . Then the input compression bound becomes:

![Image 5](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-bound.png)

The black line is the optimal achievable information bottleneck (IB) limit. The red line corresponds to the upper bound on the out-of-sample IB distortion, when trained on a finite sample set. is the complexity gap and is the generalization gap. (Recreated based on [Tishby’ talk 24:50](https://youtu.be/bLqJHjXihK8?t=24m56s))

## Network Size and Training Data Size

Having more layers give us computational benefits and speed up the training process for good generalization.

![Image 6](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-layers.png)

The optimization time is much shorter (fewer epochs) with more hidden layers. (Image source: [Shwartz-Ziv and Tishby, 2017](https://arxiv.org/pdf/1703.00810.pdf))

**Compression through stochastic relaxation**: According to the [diffusion equation](https://en.wikipedia.org/wiki/Fokker%E2%80%93Planck_equation), the relaxation time of layer is proportional to the exponential of this layer’s compression amount : . We can compute the layer compression as . Because , we would expect an exponential decrease in training epochs with more hidden layers (larger ).

## The Benefit of More Training Samples

Fitting more training data requires more information captured by the hidden layers. With increased training data size, the decoder mutual information (recall that this is directly related to the generalization error), , is pushed up and gets closer to the theoretical information bottleneck bound. Tishby emphasized that It is the mutual information, not the layer size or the VC dimension, that determines generalization, different from standard theories.

![Image 7](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/ib-training-size.png)

The training data of different sizes is color-coded. The information plane of multiple converged networks are plotted. More training data leads to better generalization. (Image source: [Shwartz-Ziv and Tishby, 2017](https://arxiv.org/pdf/1703.00810.pdf))

* * *

Cited as:

```
@article{weng2017infotheory,
  title   = "Anatomize Deep Learning with Information Theory",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2017",
  url     = "https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/"
}
```

## References

[1] Naftali Tishby. [Information Theory of Deep Learning](https://youtu.be/bLqJHjXihK8)

[2] [Machine Learning Theory - Part 1: Introduction](https://mostafa-samir.github.io/ml-theory-pt1/)

[3] [Machine Learning Theory - Part 2: Generalization Bounds](https://mostafa-samir.github.io/ml-theory-pt2/)

[4] [New Theory Cracks Open the Black Box of Deep Learning](https://www.quantamagazine.org/new-theory-cracks-open-the-black-box-of-deep-learning-20170921/) by Quanta Magazine.

[5] Naftali Tishby and Noga Zaslavsky. [“Deep learning and the information bottleneck principle.”](https://arxiv.org/pdf/1503.02406.pdf) IEEE Information Theory Workshop (ITW), 2015.

[6] Ravid Shwartz-Ziv and Naftali Tishby. [“Opening the Black Box of Deep Neural Networks via Information.”](https://arxiv.org/pdf/1703.00810.pdf) arXiv preprint arXiv:1703.00810, 2017.