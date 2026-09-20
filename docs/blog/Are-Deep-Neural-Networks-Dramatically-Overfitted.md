---
title: Are Deep Neural Networks Dramatically Overfitted?
url: https://lilianweng.github.io/posts/2019-03-14-overfit/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:46:52.258208+00:00'
---

[Updated on 2019-05-27: add the [section](https://lilianweng.github.io/posts/2019-03-14-overfit/#the-lottery-ticket-hypothesis) on Lottery Ticket Hypothesis.]

If you are like me, entering into the field of deep learning with experience in traditional machine learning, you may often ponder over this question: Since a typical deep neural network has so many parameters and training error can easily be perfect, it should surely suffer from substantial overfitting. How could it be ever generalized to out-of-sample data points?

The effort in understanding why deep neural networks can generalize somehow reminds me of this interesting paper on System Biology — [“Can a biologist fix a radio?”](https://www.cell.com/cancer-cell/pdf/S1535-6108(02)00133-2.pdf) (Lazebnik, 2002). If a biologist intends to fix a radio machine like how she works on a biological system, life could be hard. Because the full mechanism of the radio system is not revealed, poking small local functionalities might give some hints but it can hardly present all the interactions within the system, let alone the entire working flow. No matter whether you think it is relevant to DL, it is a very fun read.

I would like to discuss a couple of papers on generalizability and complexity measurement of deep learning models in the post. Hopefully, it could shed light on your thinking path towards the understanding of why DNN can generalize.

## Classic Theorems on Compression and Model Selection

Let’s say we have a classification problem and a dataset, we can develop many models to solve it, from fitting a simple linear regression to memorizing the full dataset in disk space. Which one is better? If we only care about the accuracy over training data (especially given that testing data is likely unknown), the memorization approach seems to be the best — well, it doesn’t sound right.

There are many classic theorems to guide us when deciding what types of properties a good model should possess in such scenarios.

## Occam’s Razor

[Occam’s Razor](http://pespmc1.vub.ac.be/OCCAMRAZ.html) is an informal principle for problem-solving, proposed by [William of Ockham](https://en.wikipedia.org/wiki/William_of_Ockham) in the 14th century:

> “Simpler solutions are more likely to be correct than complex ones.”

The statement is extremely powerful when we are facing multiple candidates of underlying theories to explain the world and have to pick one. Too many unnecessary assumptions might seem to be plausible for one problem, but harder to be generalized to other complications or to eventually lead to the basic principles of the universe.

Think of this, it took people hundreds of years to figure out that the sky is blue in the daytime but reddish at sunset are because of the same reason ([Rayleigh scattering](https://en.wikipedia.org/wiki/Rayleigh_scattering)), although two phenomena look very different. People must have proposed many other explanations for them separately but the unified and simple version won eventually.

## Minimum Description Length principle

The principle of Occam’s Razor can be similarly applied to machine learning models. A formalized version of such concept is called the _Minimum Description Length (MDL)_ principle, used for comparing competing models / explanations given data observed.

> “Comprehension is compression.”

The fundamental idea in MDL is to _view learning as data compression_. By compressing the data, we need to discover regularity or patterns in the data with the high potentiality to generalize to unseen samples. [Information bottleneck](https://lilianweng.github.io/posts/2017-09-28-information-bottleneck/) theory believes that a deep neural network is trained first to represent the data by minimizing the generalization error and then learn to compress this representation by trimming noise.

Meanwhile, MDL considers the model description as part of the compression delivery, so the model cannot be arbitrarily large.

A _two-part version_ of MDL principle states that: Let be a list of models that can explain the dataset . The best hypothesis among them should be the one that minimizes the sum:

*    is the length of the description of model in bits.
*    is the length of the description of the data in bits when encoded with .

In simple words, the _best_ model is the _smallest_ model containing the encoded data and the model itself. Following this criterion, the memorization approach I proposed at the beginning of the section sounds horrible no matter how good accuracy it can achieve on the training data.

People might argue Occam’s Razor is wrong, as given the real world can be arbitrarily complicated, why do we have to find simple models? One interesting view by MDL is to consider models as **“languages”** instead of fundamental generative theorems. We would like to find good compression strategies to describe regularity in a small set of samples, and they **do not have to be the “real” generative model** for explaining the phenomenon. Models can be wrong but still useful (i.e., think of any Bayesian prior).

## Kolmogorov Complexity

Kolmogorov Complexity relies on the concept of modern computers to define the algorithmic (descriptive) complexity of an object: It is _the length of the shortest binary computer program that describes the object_. Following MDL, a computer is essentially the most general form of data decompressor.

The formal definition of Kolmogorov Complexity states that: Given a universal computer and a program , let’s denote as the output of the computer processing the program and as the descriptive length of the program. Then Kolmogorov Complexity of a string with respect to a universal computer is:

Note that a universal computer is one that can mimic the actions of any other computers. All modern computers are universal as they can all be reduced to Turing machines. The definition is universal no matter which computers we are using, because another universal computer can always be programmed to clone the behavior of , while encoding this clone program is just a constant.

There are a lot of connections between Kolmogorov Complexity and Shannon Information Theory, as both are tied to universal coding. It is an amazing fact that the expected Kolmogorov Complexity of a random variable is approximately equal to its Shannon entropy (see Sec 2.3 of [the report](https://homepages.cwi.nl/~paulv/papers/info.pdf)). More on this topic is out of the scope here, but there are many interesting readings online. Help yourself :)

## Solomonoff’s Inference Theory

Another mathematical formalization of Occam’s Razor is Solomonoff’s theory of universal inductive inference ([Solomonoff](https://www.sciencedirect.com/science/article/pii/S0019995864902232), [1964](https://www.sciencedirect.com/science/article/pii/S0019995864901317)). The principle is to favor models that correspond to the “shortest program” to produce the training data, based on its Kolmogorov complexity

## Expressive Power of DL Models

Deep neural networks have an extremely large number of parameters compared to the traditional statistical models. If we use MDL to measure the complexity of a deep neural network and consider the number of parameters as the model description length, it would look awful. The model description can easily grow out of control.

However, having numerous parameters is _necessary_ for a neural network to obtain high expressivity power. Because of its great capability to capture any flexible data representation, deep neural networks have achieved great success in many applications.

## Universal Approximation Theorem

The _Universal Approximation Theorem_ states that a feedforward network with: 1) a linear output layer, 2) at least one hidden layer containing a finite number of neurons and 3) some activation function can approximate **any** continuous functions on a compact subset of to arbitrary accuracy. The theorem was first proved for sigmoid activation function ([Cybenko, 1989](https://pdfs.semanticscholar.org/05ce/b32839c26c8d2cb38d5529cf7720a68c3fab.pdf)). Later it was shown that the universal approximation property is not specific to the choice of activation ([Hornik, 1991](http://zmjones.com/static/statistical-learning/hornik-nn-1991.pdf)) but the multilayer feedforward architecture.

Although a feedforward network with a single layer is sufficient to represent any function, the width has to be exponentially large. The universal approximation theorem does not guarantee whether the model can be learned or generalized properly. Often, adding more layers helps to reduce the number of hidden neurons needed in a shallow network.

To take advantage of the universal approximation theorem, we can always find a neural network to represent the target function with error under any desired threshold, but we need to pay the price — the network might grow super large.

## Proof: Finite Sample Expressivity of Two-layer NN

The Universal Approximation Theorem we have discussed so far does not consider a finite sample set. [Zhang, et al. (2017)](https://arxiv.org/abs/1611.03530) provided a neat proof on the finite-sample expressivity of two-layer neural networks.

A neural network can represent any function given a sample size in dimensions if: For every finite sample set with and every function defined on this sample set: , we can find a set of weight configuration for so that .

The paper proposed a theorem:

> There exists a two-layer neural network with ReLU activations and weights that can represent any function on a sample of size in dimensions.

_Proof._ First we would like to construct a two-layer neural network . The input is a -dimensional vector, . The hidden layer has hidden units, associated with a weight matrix , a bias vector and ReLU activation function. The second layer outputs a scalar value with weight vector and zero biases.

The output of network for a input vector can be represented as follows:

where is the -th column in the matrix.

Given a sample set and target values , we would like to find proper weights , so that .

Let’s combine all sample points into one batch as one input matrix . If set , would be a square matrix of size .

We can simplify to have the same column vectors across all the columns:

![Image 1](https://lilianweng.github.io/posts/2019-03-14-overfit/nn-expressivity-proof.png)
Let , we would like to find a suitable and such that . This is always achievable because we try to solve unknown variables with constraints and are independent (i.e. pick a random , sort and then set ’s as values in between). Then becomes a lower triangular matrix:

It is a nonsingular square matrix as , so we can always find suitable to solve (In other words, the column space of is all of and we can find a linear combination of column vectors to obtain any ).

## Deep NN can Learn Random Noise

As we know two-layer neural networks are universal approximators, it is less surprising to see that they are able to learn unstructured random noise perfectly, as shown in [Zhang, et al. (2017)](https://arxiv.org/abs/1611.03530). If labels of image classification dataset are randomly shuffled, the high expressivity power of deep neural networks can still empower them to achieve near-zero training loss. These results do not change with regularization terms added.

![Image 2](https://lilianweng.github.io/posts/2019-03-14-overfit/fit-random-labels.png)

Fit models on CIFAR10 with random labels or random pixels: (a) learning curves; (b-c) label corruption ratio is the percentage of randomly shuffled labels. (Image source: [Zhang et al. 2017](https://arxiv.org/abs/1611.03530))

## Are Deep Learning Models Dramatically Overfitted?

Deep learning models are heavily over-parameterized and can often get to perfect results on training data. In the traditional view, like bias-variance trade-offs, this could be a disaster that nothing may generalize to the unseen test data. However, as is often the case, such “overfitted” (training error = 0) deep learning models still present a decent performance on out-of-sample test data. Hmm … interesting and why?

## Modern Risk Curve for Deep Learning

The traditional machine learning uses the following U-shape risk curve to measure the bias-variance trade-offs and quantify how generalizable a model is. If I get asked how to tell whether a model is overfitted, this would be the first thing popping into my mind.

As the model turns larger (more parameters added), the training error decreases to close to zero, but the test error (generalization error) starts to increase once the model complexity grows to pass the threshold between “underfitting” and “overfitting”. In a way, this is well aligned with Occam’s Razor.

![Image 3](https://lilianweng.github.io/posts/2019-03-14-overfit/bias-variance-risk-curve.png)

U-shaped bias-variance risk curve. (Image source: (left) [paper](https://arxiv.org/abs/1812.11118) (right) [fig. 6 of this post](http://scott.fortmann-roe.com/docs/BiasVariance.html))

Unfortunately this does not apply to deep learning models. [Belkin et al. (2018)](https://arxiv.org/abs/1812.11118) reconciled the traditional bias-variance trade-offs and proposed a new double-U-shaped risk curve for deep neural networks. Once the number of network parameters is high enough, the risk curve enters another regime.

![Image 4](https://lilianweng.github.io/posts/2019-03-14-overfit/new-bias-variance-risk-curve.png)

A new double-U-shaped bias-variance risk curve for deep neural networks. (Image source: [original paper](https://arxiv.org/abs/1812.11118))

The paper claimed that it is likely due to two reasons:

*   The number of parameters is not a good measure of _inductive bias_, defined as the set of assumptions of a learning algorithm used to predict for unknown samples. See more discussion on DL model complexity in [later](https://lilianweng.github.io/posts/2019-03-14-overfit/#intrinsic-dimension)[sections](https://lilianweng.github.io/posts/2019-03-14-overfit/#heterogeneous-layer-robustness).
*   Equipped with a larger model, we might be able to discover larger function classes and further find interpolating functions that have smaller norm and are thus “simpler”.

The double-U-shaped risk curve was observed empirically, as shown in the paper. However I was struggling quite a bit to reproduce the results. There are some signs of life, but in order to generate a pretty smooth curve similar to the theorem, [many details](https://lilianweng.github.io/posts/2019-03-14-overfit/#experiments) in the experiment have to be taken care of.

![Image 5](https://lilianweng.github.io/posts/2019-03-14-overfit/new-risk-curve-mnist.png)

Training and evaluation errors of a one hidden layer fc network of different numbers of hidden units, trained on 4000 data points sampled from MNIST. (Image source: [original paper](https://arxiv.org/abs/1812.11118))

## Regularization is not the Key to Generalization

Regularization is a common way to control overfitting and improve model generalization performance. Interestingly some research ([Zhang, et al. 2017](https://arxiv.org/abs/1611.03530)) has shown that explicit regularization (i.e. data augmentation, weight decay and dropout) is neither necessary or sufficient for reducing generalization error.

Taking the Inception model trained on CIFAR10 as an example (see Fig. 5), regularization techniques help with out-of-sample generalization but not much. No single regularization seems to be critical independent of other terms. Thus, it is unlikely that regularizers are the _fundamental reason_ for generalization.

![Image 6](https://lilianweng.github.io/posts/2019-03-14-overfit/regularization-generalization-test.png)

The accuracy of Inception model trained on CIFAR10 with different combinations of taking on or off data augmentation and weight decay. (Image source: Table 1 in the [original paper](https://arxiv.org/abs/1611.03530))

## Intrinsic Dimension

The number of parameters is not correlated with model overfitting in the field of deep learning, suggesting that parameter counting cannot indicate the true complexity of deep neural networks.

Apart from parameter counting, researchers have proposed many ways to quantify the complexity of these models, such as the number of degrees of freedom of models ([Gao & Jojic, 2016](https://arxiv.org/abs/1603.09260)), or prequential code ([Blier & Ollivier, 2018](https://arxiv.org/abs/1802.07044)).

I would like to discuss a recent method on this matter, named **intrinsic dimension** ([Li et al, 2018](https://arxiv.org/abs/1804.08838)). Intrinsic dimension is intuitive, easy to measure, while still revealing many interesting properties of models of different sizes.

Considering a neural network with a great number of parameters, forming a high-dimensional parameter space, the learning happens on this high-dimensional _objective landscape_. The shape of the parameter space manifold is critical. For example, a smoother manifold is beneficial for optimization by providing more predictive gradients and allowing for larger learning rates—this was claimed to be the reason why batch normalization has succeeded in stabilizing training ([Santurkar, et al, 2019](https://arxiv.org/abs/1805.11604)).

Even though the parameter space is huge, fortunately we don’t have to worry too much about the optimization process getting stuck in local optima, as it has been [shown](https://arxiv.org/abs/1406.2572) that local optimal points in the objective landscape almost always lay in saddle-points rather than valleys. In other words, there is always a subset of dimensions containing paths to leave local optima and keep on exploring.

![Image 7](https://lilianweng.github.io/posts/2019-03-14-overfit/optimization-landscape-shape.png)

Illustrations of various types of critical points on the parameter optimization landscape. (Image source: [here](https://www.offconvex.org/2016/03/22/saddlepoints/))

One intuition behind the measurement of intrinsic dimension is that, since the parameter space has such high dimensionality, it is probably not necessary to exploit all the dimensions to learn efficiently. If we only travel through a slice of objective landscape and still can learn a good solution, the complexity of the resulting model is likely lower than what it appears to be by parameter-counting. This is essentially what intrinsic dimension tries to assess.

Say a model has dimensions and its parameters are denoted as . For learning, a smaller -dimensional subspace is randomly sampled, , where . During one optimization update, rather than taking a gradient step according to all dimensions, only the smaller subspace is used and remapped to update model parameters.

![Image 8](https://lilianweng.github.io/posts/2019-03-14-overfit/intrinsic-dimension-illustration.png)

Illustration of parameter vectors for direct optimization when . (Image source: [original paper](https://arxiv.org/abs/1804.08838))

The gradient update formula looks like the follows:

where are the initialization values and is a projection matrix that is randomly sampled before training. Both and are not trainable and fixed during training. is initialized as all zeros.

By searching through the value of , the corresponding when the solution emerges is defined as the _intrinsic dimension_.

It turns out many problems have much smaller intrinsic dimensions than the number of parameters. For example, on CIFAR10 image classification, a fully-connected network with 650k+ parameters has only 9k intrinsic dimension and a convolutional network containing 62k parameters has an even lower intrinsic dimension of 2.9k.

![Image 9](https://lilianweng.github.io/posts/2019-03-14-overfit/intrinsic-dimension.png)

The measured intrinsic dimensions for various models achieving 90% of the best performance. (Image source: [original paper](https://arxiv.org/abs/1804.08838))

The measurement of intrinsic dimensions suggests that deep learning models are significantly simpler than what they might appear to be.

## Heterogeneous Layer Robustness

[Zhang et al. (2019)](https://arxiv.org/abs/1902.01996) investigated the role of parameters in different layers. The fundamental question raised by the paper is: _“are all layers created equal?”_ The short answer is: No. The model is more sensitive to changes in some layers but not others.

The paper proposed two types of operations that can be applied to parameters of the -th layer, , at time , to test their impacts on model robustness:

*   **Re-initialization**: Reset the parameters to the initial values, . The performance of a network in which layer was re-initialized is referred to as the _re-initialization robustness_ of layer .

*   **Re-randomization**: Re-sampling the layer’s parameters at random, . The corresponding network performance is called the _re-randomization robustness_ of layer .

Layers can be categorized into two categories with the help of these two operations:

*   **Robust Layers**: The network has no or only negligible performance degradation after re-initializing or re-randomizing the layer.
*   **Critical Layers**: Otherwise.

Similar patterns are observed on fully-connected and convolutional networks. Re-randomizing any of the layers _completely destroys_ the model performance, as the prediction drops to random guessing immediately. More interestingly and surprisingly, when applying re-initialization, only the first or the first few layers (those closest to the input layer) are critical, while re-initializing higher levels causes _only negligible decrease_ in performance.

![Image 10](https://lilianweng.github.io/posts/2019-03-14-overfit/layer-robustness-results.png)

(a) A fc network trained on MNIST. Each row corresponds to one layer in the network. The first column is re-randomization robustness of each layer and the rest of the columns indicate re-initialization robustness at different training time. (b) VGG11 model (conv net) trained on CIFAR 10. Similar representation as in (a) but rows and columns are transposed. (Image source: [original paper](https://arxiv.org/abs/1902.01996))

ResNet is able to use shortcuts between non-adjacent layers to re-distribute the sensitive layers across the networks rather than just at the bottom. With the help of residual block architecture, the network can _evenly be robust to re-randomization_. Only the first layer of each residual block is still sensitive to both re-initialization and re-randomization. If we consider each residual block as a local sub-network, the robustness pattern resembles the fc and conv nets above.

![Image 11](https://lilianweng.github.io/posts/2019-03-14-overfit/layer-robustness-resnet.png)

Re-randomization (first row) and re-initialization (the reset rows) robustness of layers in ResNet-50 model trained on CIFAR10. (Image source: [original paper](https://arxiv.org/abs/1902.01996))

Based on the fact that many top layers in deep neural networks are not critical to the model performance after re-initialization, the paper loosely concluded that:

> “Over-capacitated deep networks trained with stochastic gradient have low-complexity due to self-restricting the number of critical layers.”

We can consider re-initialization as a way to reduce the effective number of parameters, and thus the observation is aligned with what intrinsic dimension has demonstrated.

## The Lottery Ticket Hypothesis

The lottery ticket hypothesis ([Frankle & Carbin, 2019](https://arxiv.org/abs/1803.03635)) is another intriguing and inspiring discovery, supporting that only a subset of network parameters have impact on the model performance and thus the network is not overfitted. The lottery ticket hypothesis states that a randomly initialized, dense, feed-forward network contains a pool of subnetworks and among them only a subset are _“winning tickets”_ which can achieve the optimal performance when _trained in isolation_.

The idea is motivated by network pruning techniques — removing unnecessary weights (i.e. tiny weights that are almost negligible) without harming the model performance. Although the final network size can be reduced dramatically, it is hard to train such a pruned network architecture successfully from scratch. It feels like in order to successfully train a neural network, we need a large number of parameters, but we don’t need that many parameters to keep the accuracy high once the model is trained. Why is that?

The lottery ticket hypothesis did the following experiments:

1.   Randomly initialize a dense feed-forward network with initialization values ;
2.   Train the network for multiple iterations to achieve a good performance with parameter config ;
3.   Run pruning on and creating a mask .
4.   The “winning ticket” initialization config is .

Only training the small “winning ticket” subset of parameters with the initial values as found in step 1, the model is able to achieve the same level of accuracy as in step 2. It turns out a large parameter space is not needed in the final solution representation, but needed for training as it provides a big pool of initialization configs of many much smaller subnetworks.

The lottery ticket hypothesis opens a new perspective about interpreting and dissecting deep neural network results. Many interesting following-up works are on the way.

## Experiments

After seeing all the interesting findings above, it should be pretty fun to reproduce them. Some results are easily to reproduce than others. Details are described below. My code is available on github [lilianweng/generalization-experiment](https://github.com/lilianweng/generalization-experiment).

**New Risk Curve for DL Models**

This is the trickiest one to reproduce. The authors did give me a lot of good advice and I appreciate it a lot. Here are a couple of noticeable settings in their experiments:

*   There are no regularization terms like weight decay, dropout.
*   In Fig 3, the training set contains 4k samples. It is only sampled once and fixed for all the models. The evaluation uses the full MNIST test set.
*   Each network is trained for a long time to achieve near-zero training risk. The learning rate is adjusted differently for models of different sizes.
*   To make the model less sensitive to the initialization in the under-parameterization region, their experiments adopted a _“weight reuse”_ scheme: the parameters obtained from training a smaller neural network are used as initialization for training larger networks.

I did not train or tune each model long enough to get perfect training performance, but evaluation error indeed shows a special twist around the interpolation threshold, different from training error. For example, for MNIST, the threshold is the number of training samples times the number of classes (10), that is 40000.

The x-axis is the number of model parameters: (28 * 28 + 1) * num. units + num. units * 10, in logarithm.

![Image 12](https://lilianweng.github.io/posts/2019-03-14-overfit/risk_curve_loss-mse_sample-4000_epoch-500.png)
**Layers are not Created Equal**

This one is fairly easy to reproduce. See my implementation [here](https://github.com/lilianweng/generalization-experiment/blob/master/layer_equality.py).

In the first experiment, I used a three-layer fc networks with 256 units in each layer. Layer 0 is the input layer while layer 3 is the output. The network is trained on MNIST for 100 epochs.

![Image 13](https://lilianweng.github.io/posts/2019-03-14-overfit/layer_equality_256x3.png)
In the second experiment, I used a four-layer fc networks with 128 units in each layer. Other settings are the same as experiment 1.

![Image 14](https://lilianweng.github.io/posts/2019-03-14-overfit/layer_equality_128x4.png)
**Intrinsic Dimension Measurement**

To correctly map the -dimensional subspace to the full parameter space, the projection matrix should have orthogonal columns. Because the production is the sum of columns of scaled by corresponding scalar values in the -dim vector, , it is better to fully utilize the subspace with orthogonal columns in .

My implementation follows a naive approach by sampling a large matrix with independent entries from a standard normal distribution. The columns are expected to be independent in a high dimension space and thus to be orthogonal. This works when the dimension is not too large. When exploring with a large , there are methods for creating sparse projection matrices, which is what the intrinsic dimension paper suggested.

Here are experiment runs on two networks: (left) a two-layer fc network with 64 units in each layer and (right) a one-layer fc network with 128 hidden units, trained on 10% of MNIST. For every , the model is trained for 100 epochs. See the [code](https://github.com/lilianweng/generalization-experiment/blob/master/intrinsic_dimensions.py)[here](https://github.com/lilianweng/generalization-experiment/blob/master/intrinsic_dimensions_measurement.py).

![Image 15](https://lilianweng.github.io/posts/2019-03-14-overfit/intrinsic-dimension-net-64-64-and-128.png)

* * *

Cited as:

```
@article{weng2019overfit,
  title   = "Are Deep Neural Networks Dramatically Overfitted?",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2019",
  url     = "https://lilianweng.github.io/posts/2019-03-14-overfit/"
}
```

## References

[1] Wikipedia page on [Occam’s Razor](https://en.wikipedia.org/wiki/Occam%27s_razor).

[2] [Occam’s Razor](http://pespmc1.vub.ac.be/OCCAMRAZ.html) on Principia Cybernetica Web.

[3] Peter Grunwald. [“A Tutorial Introduction to the Minimum Description Length Principle”](https://arxiv.org/abs/math/0406077). 2004.

[4] Ian Goodfellow, et al. [Deep Learning](https://www.deeplearningbook.org/). 2016. [Sec 6.4.1](https://www.deeplearningbook.org/contents/mlp.html).

[5] Zhang, Chiyuan, et al. [“Understanding deep learning requires rethinking generalization.”](https://arxiv.org/abs/1611.03530) ICLR 2017.

[6] Shibani Santurkar, et al. [“How does batch normalization help optimization?.”](https://arxiv.org/abs/1805.11604) NIPS 2018.

[7] Mikhail Belkin, et al. [“Reconciling modern machine learning and the bias-variance trade-off.”](https://arxiv.org/abs/1812.11118) arXiv:1812.11118, 2018.

[8] Chiyuan Zhang, et al. [“Are All Layers Created Equal?”](https://arxiv.org/abs/1902.01996) arXiv:1902.01996, 2019.

[9] Chunyuan Li, et al. [“Measuring the intrinsic dimension of objective landscapes.”](https://arxiv.org/abs/1804.08838) ICLR 2018.

[10] Jonathan Frankle and Michael Carbin. [“The lottery ticket hypothesis: Finding sparse, trainable neural networks.”](https://arxiv.org/abs/1803.03635) ICLR 2019.