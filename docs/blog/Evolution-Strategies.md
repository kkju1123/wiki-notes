---
title: Evolution Strategies
url: https://lilianweng.github.io/posts/2019-09-05-evolution-strategies/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:46:33.231119+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
Evolution Strategies
Date: September 5, 2019 | Estimated Reading Time: 22 min | Author: Lilian Weng
Table of Contents

Stochastic gradient descent is a universal choice for optimizing deep learning models. However, it is not the only option. With black-box optimization algorithms, you can evaluate a target function 
𝑓
(
𝑥
)
:
𝑅
𝑛
→
𝑅
, even when you don’t know the precise analytic form of 
𝑓
(
𝑥
)
 and thus cannot compute gradients or the Hessian matrix. Examples of black-box optimization methods include Simulated Annealing, Hill Climbing and Nelder-Mead method.

Evolution Strategies (ES) is one type of black-box optimization algorithms, born in the family of Evolutionary Algorithms (EA). In this post, I would dive into a couple of classic ES methods and introduce a few applications of how ES can play a role in deep reinforcement learning.

What are Evolution Strategies?

Evolution strategies (ES) belong to the big family of evolutionary algorithms. The optimization targets of ES are vectors of real numbers, 
𝑥
∈
𝑅
𝑛
.

Evolutionary algorithms refer to a division of population-based optimization algorithms inspired by natural selection. Natural selection believes that individuals with traits beneficial to their survival can live through generations and pass down the good characteristics to the next generation. Evolution happens by the selection process gradually and the population grows better adapted to the environment.

How natural selection works. (Image source: Khan Academy: Darwin, evolution, & natural selection)

Evolutionary algorithms can be summarized in the following format as a general optimization solution:

Let’s say we want to optimize a function 
𝑓
(
𝑥
)
 and we are not able to compute gradients directly. But we still can evaluate 
𝑓
(
𝑥
)
 given any 
𝑥
 and the result is deterministic. Our belief in the probability distribution over 
𝑥
 as a good solution to 
𝑓
(
𝑥
)
 optimization is 
𝑝
𝜃
(
𝑥
)
, parameterized by 
𝜃
. The goal is to find an optimal configuration of 
𝜃
.

Here given a fixed format of distribution (i.e. Gaussian), the parameter 
𝜃
 carries the knowledge about the best solutions and is being iteratively updated across generations.

Starting with an initial value of 
𝜃
, we can continuously update 
𝜃
 by looping three steps as follows:

Generate a population of samples 
𝐷
=
{
(
𝑥
𝑖
,
𝑓
(
𝑥
𝑖
)
}
 where 
𝑥
𝑖
∼
𝑝
𝜃
(
𝑥
)
.
Evaluate the “fitness” of samples in 
𝐷
.
Select the best subset of individuals and use them to update 
𝜃
, generally based on fitness or rank.

In Genetic Algorithms (GA), another popular subcategory of EA, 
𝑥
 is a sequence of binary codes, 
𝑥
∈
{
0
,
1
}
𝑛
. While in ES, 
𝑥
 is just a vector of real numbers, 
𝑥
∈
𝑅
𝑛
.

Simple Gaussian Evolution Strategies

This is the most basic and canonical version of evolution strategies. It models 
𝑝
𝜃
(
𝑥
)
 as a 
𝑛
-dimensional isotropic Gaussian distribution, in which 
𝜃
 only tracks the mean 
𝜇
 and standard deviation 
𝜎
.

𝜃
=
(
𝜇
,
𝜎
)
,
𝑝
𝜃
(
𝑥
)
∼
𝑁
(
𝜇
,
𝜎
2
𝐼
)
=
𝜇
+
𝜎
𝑁
(
0
,
𝐼
)

The process of Simple-Gaussian-ES, given 
𝑥
∈
𝑅
𝑛
:

Initialize 
𝜃
=
𝜃
(
0
)
 and the generation counter 
𝑡
=
0
Generate the offspring population of size 
Λ
 by sampling from the Gaussian distribution:


𝐷
(
𝑡
+
1
)
=
{
𝑥
𝑖
(
𝑡
+
1
)
∣
𝑥
𝑖
(
𝑡
+
1
)
=
𝜇
(
𝑡
)
+
𝜎
(
𝑡
)
𝑦
𝑖
(
𝑡
+
1
)
 where 
𝑦
𝑖
(
𝑡
+
1
)
∼
𝑁
(
𝑥
|
0
,
𝐼
)
,
;
𝑖
=
1
,
…
,
Λ
}

.
Select a top subset of 
𝜆
 samples with optimal 
𝑓
(
𝑥
𝑖
)
 and this subset is called elite set. Without loss of generality, we may consider the first 
𝑘
 samples in 
𝐷
(
𝑡
+
1
)
 to belong to the elite group — Let’s label them as
𝐷
(
𝑡
+
1
)
_
elite
=
𝑥
(
𝑡
+
1
)
_
𝑖
∣
𝑥
(
𝑡
+
1
)
_
𝑖
∈
𝐷
(
𝑡
+
1
)
,
𝑖
=
1
,
…
,
𝜆
,
𝜆
≤
Λ
Then we estimate the new mean and std for the next generation using the elite set:


	



	


𝜇
(
𝑡
+
1
)
	
=
avg
(
𝐷
elite
(
𝑡
+
1
)
)
=
1
𝜆
∑
𝑖
=
1
𝜆
𝑥
𝑖
(
𝑡
+
1
)


𝜎
(
𝑡
+
1
)
2
	
=
var
(
𝐷
elite
(
𝑡
+
1
)
)
=
1
𝜆
∑
𝑖
=
1
𝜆
(
𝑥
𝑖
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
)
2
Repeat steps (2)-(4) until the result is good enough ✌️
Covariance Matrix Adaptation Evolution Strategies (CMA-ES)

The standard deviation 
𝜎
 accounts for the level of exploration: the larger 
𝜎
 the bigger search space we can sample our offspring population. In vanilla ES, 
𝜎
(
𝑡
+
1
)
 is highly correlated with 
𝜎
(
𝑡
)
, so the algorithm is not able to rapidly adjust the exploration space when needed (i.e. when the confidence level changes).

CMA-ES, short for “Covariance Matrix Adaptation Evolution Strategy”, fixes the problem by tracking pairwise dependencies between the samples in the distribution with a covariance matrix 
𝐶
. The new distribution parameter becomes:

𝜃
=
(
𝜇
,
𝜎
,
𝐶
)
,
𝑝
𝜃
(
𝑥
)
∼
𝑁
(
𝜇
,
𝜎
2
𝐶
)
∼
𝜇
+
𝜎
𝑁
(
0
,
𝐶
)

where 
𝜎
 controls for the overall scale of the distribution, often known as step size.

Before we dig into how the parameters are updated in CMA-ES, it is better to review how the covariance matrix works in the multivariate Gaussian distribution first. As a real symmetric matrix, the covariance matrix 
𝐶
 has the following nice features (See proof & proof):

It is always diagonalizable.
Always positive semi-definite.
All of its eigenvalues are real non-negative numbers.
All of its eigenvectors are orthogonal.
There is an orthonormal basis of 
𝑅
𝑛
 consisting of its eigenvectors.

Let the matrix 
𝐶
 have an orthonormal basis of eigenvectors 
𝐵
=
[
𝑏
1
,
…
,
𝑏
𝑛
]
, with corresponding eigenvalues 
𝜆
1
2
,
…
,
𝜆
𝑛
2
. Let 
𝐷
=
diag
(
𝜆
1
,
…
,
𝜆
𝑛
)
.

			
			
			
			
	
		
			
			
		
		
		
		
𝐶
=
𝐵
⊤
𝐷
2
𝐵
=
[
∣
	
∣
		
∣


𝑏
1
	
𝑏
2
	
…
	
𝑏
𝑛


∣
	
∣
		
∣
]
[
𝜆
1
2
	
0
	
…
	
0


0
	
𝜆
2
2
	
…
	
0


⋮
	
…
	
⋱
	
⋮


0
	
…
	
0
	
𝜆
𝑛
2
]
[
−
	
𝑏
1
	
−


−
	
𝑏
2
	
−

	
…
	

−
	
𝑏
𝑛
	
−
]

The square root of 
𝐶
 is:

𝐶
1
2
=
𝐵
⊤
𝐷
𝐵
Symbol	Meaning

𝑥
𝑖
(
𝑡
)
∈
𝑅
𝑛
	the 
𝑖
-th samples at the generation (t)

𝑦
𝑖
(
𝑡
)
∈
𝑅
𝑛
	
𝑥
𝑖
(
𝑡
)
=
𝜇
(
𝑡
−
1
)
+
𝜎
(
𝑡
−
1
)
𝑦
𝑖
(
𝑡
)


𝜇
(
𝑡
)
	mean of the generation (t)

𝜎
(
𝑡
)
	step size

𝐶
(
𝑡
)
	covariance matrix

𝐵
(
𝑡
)
	a matrix of 
𝐶
’s eigenvectors as row vectors

𝐷
(
𝑡
)
	a diagonal matrix with 
𝐶
’s eigenvalues on the diagnose.

𝑝
𝜎
(
𝑡
)
	evaluation path for 
𝜎
 at the generation (t)

𝑝
𝑐
(
𝑡
)
	evaluation path for 
𝐶
 at the generation (t)

𝛼
𝜇
	learning rate for 
𝜇
’s update

𝛼
𝜎
	learning rate for 
𝑝
𝜎


𝑑
𝜎
	damping factor for 
𝜎
’s update

𝛼
𝑐
𝑝
	learning rate for 
𝑝
𝑐


𝛼
𝑐
𝜆
	learning rate for 
𝐶
’s rank-min(λ, n) update

𝛼
𝑐
1
	learning rate for 
𝐶
’s rank-1 update
Updating the Mean


𝜇
(
𝑡
+
1
)
=
𝜇
(
𝑡
)
+
𝛼
𝜇
1
𝜆
∑
𝑖
=
1
𝜆
(
𝑥
𝑖
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
)

CMA-ES has a learning rate 
𝛼
𝜇
≤
1
 to control how fast the mean 
𝜇
 should be updated. Usually it is set to 1 and thus the equation becomes the same as in vanilla ES, 
𝜇
(
𝑡
+
1
)
=
1
𝜆
∑
𝑖
=
1
𝜆
(
𝑥
𝑖
(
𝑡
+
1
)
.

Controlling the Step Size

The sampling process can be decoupled from the mean and standard deviation:

𝑥
𝑖
(
𝑡
+
1
)
=
𝜇
(
𝑡
)
+
𝜎
(
𝑡
)
𝑦
𝑖
(
𝑡
+
1
)
, where 
𝑦
𝑖
(
𝑡
+
1
)
=
𝑥
𝑖
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)
∼
𝑁
(
0
,
𝐶
)

The parameter 
𝜎
 controls the overall scale of the distribution. It is separated from the covariance matrix so that we can change steps faster than the full covariance. A larger step size leads to faster parameter update. In order to evaluate whether the current step size is proper, CMA-ES constructs an evolution path 
𝑝
𝜎
 by summing up a consecutive sequence of moving steps, 
1
𝜆
∑
𝑖
𝜆
𝑦
𝑖
(
𝑗
)
,
𝑗
=
1
,
…
,
𝑡
. By comparing this path length with its expected length under random selection (meaning single steps are uncorrelated), we are able to adjust 
𝜎
 accordingly (See Fig. 2).

Three scenarios of how single steps are correlated in different ways and their impacts on step size update. (Image source: additional annotations on Fig 5 in CMA-ES tutorial paper)

Each time the evolution path is updated with the average of moving step 
𝑦
𝑖
 in the same generation.

	



	



	
	
1
𝜆
∑
𝑖
=
1
𝜆
𝑦
𝑖
(
𝑡
+
1
)
=
1
𝜆
∑
𝑖
=
1
𝜆
𝑥
𝑖
(
𝑡
+
1
)
−
𝜆
𝜇
(
𝑡
)
𝜎
(
𝑡
)
=
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)

	
1
𝜆
∑
𝑖
=
1
𝜆
𝑦
𝑖
(
𝑡
+
1
)
∼
1
𝜆
𝑁
(
0
,
𝜆
𝐶
(
𝑡
)
)
∼
1
𝜆
𝐶
(
𝑡
)
1
2
𝑁
(
0
,
𝐼
)

	
Thus 
𝜆
𝐶
(
𝑡
)
−
1
2
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)
∼
𝑁
(
0
,
𝐼
)

By multiplying with 
𝐶
−
1
2
, the evolution path is transformed to be independent of its direction. The term 
𝐶
(
𝑡
)
−
1
2
=
𝐵
(
𝑡
)
⊤
𝐷
(
𝑡
)
−
1
2
𝐵
(
𝑡
)
 transformation works as follows:

𝐵
(
𝑡
)
 contains row vectors of 
𝐶
’s eigenvectors. It projects the original space onto the perpendicular principal axes.
Then 
𝐷
(
𝑡
)
−
1
2
=
diag
(
1
𝜆
1
,
…
,
1
𝜆
𝑛
)
 scales the length of principal axes to be equal.
𝐵
(
𝑡
)
⊤
 transforms the space back to the original coordinate system.

In order to assign higher weights to recent generations, we use polyak averaging to update the evolution path with learning rate 
𝛼
𝜎
. Meanwhile, the weights are balanced so that 
𝑝
𝜎
 is conjugate, 
∼
𝑁
(
0
,
𝐼
)
 both before and after one update.

	

	
𝑝
𝜎
(
𝑡
+
1
)
	
=
(
1
−
𝛼
𝜎
)
𝑝
𝜎
(
𝑡
)
+
1
−
(
1
−
𝛼
𝜎
)
2
𝜆
𝐶
(
𝑡
)
−
1
2
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)

	
=
(
1
−
𝛼
𝜎
)
𝑝
𝜎
(
𝑡
)
+
𝑐
𝜎
(
2
−
𝛼
𝜎
)
𝜆
𝐶
(
𝑡
)
−
1
2
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)

The expected length of 
𝑝
𝜎
 under random selection is 
𝐸
|
𝑁
(
0
,
𝐼
)
|
, that is the expectation of the L2-norm of a 
𝑁
(
0
,
𝐼
)
 random variable. Following the idea in Fig. 2, we adjust the step size according to the ratio of 
|
𝑝
𝜎
(
𝑡
+
1
)
|
/
𝐸
|
𝑁
(
0
,
𝐼
)
|
:

	

	
ln
⁡
𝜎
(
𝑡
+
1
)
	
=
ln
⁡
𝜎
(
𝑡
)
+
𝛼
𝜎
𝑑
𝜎
(
‖
𝑝
𝜎
(
𝑡
+
1
)
‖
𝐸
‖
𝑁
(
0
,
𝐼
)
‖
−
1
)


𝜎
(
𝑡
+
1
)
	
=
𝜎
(
𝑡
)
exp
⁡
(
𝛼
𝜎
𝑑
𝜎
(
‖
𝑝
𝜎
(
𝑡
+
1
)
‖
𝐸
‖
𝑁
(
0
,
𝐼
)
‖
−
1
)
)

where 
𝑑
𝜎
≈
1
 is a damping parameter, scaling how fast 
ln
⁡
𝜎
 should be changed.

Adapting the Covariance Matrix

For the covariance matrix, it can be estimated from scratch using 
𝑦
𝑖
 of elite samples (recall that 
𝑦
𝑖
∼
𝑁
(
0
,
𝐶
)
):





𝐶
𝜆
(
𝑡
+
1
)
=
1
𝜆
∑
𝑖
=
1
𝜆
𝑦
𝑖
(
𝑡
+
1
)
𝑦
𝑖
(
𝑡
+
1
)
⊤
=
1
𝜆
𝜎
(
𝑡
)
2
∑
𝑖
=
1
𝜆
(
𝑥
𝑖
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
)
(
𝑥
𝑖
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
)
⊤

The above estimation is only reliable when the selected population is large enough. However, we do want to run fast iteration with a small population of samples in each generation. That’s why CMA-ES invented a more reliable but also more complicated way to update 
𝐶
. It involves two independent routes,

Rank-min(λ, n) update: uses the history of 
{
𝐶
𝜆
}
, each estimated from scratch in one generation.
Rank-one update: estimates the moving steps 
𝑦
𝑖
 and the sign information from the history.

The first route considers the estimation of 
𝐶
 from the entire history of 
{
𝐶
𝜆
}
. For example, if we have experienced a large number of generations, 
𝐶
(
𝑡
+
1
)
≈
avg
(
𝐶
𝜆
(
𝑖
)
;
𝑖
=
1
,
…
,
𝑡
)
 would be a good estimator. Similar to 
𝑝
𝜎
, we also use polyak averaging with a learning rate to incorporate the history:



𝐶
(
𝑡
+
1
)
=
(
1
−
𝛼
𝑐
𝜆
)
𝐶
(
𝑡
)
+
𝛼
𝑐
𝜆
𝐶
𝜆
(
𝑡
+
1
)
=
(
1
−
𝛼
𝑐
𝜆
)
𝐶
(
𝑡
)
+
𝛼
𝑐
𝜆
1
𝜆
∑
𝑖
=
1
𝜆
𝑦
𝑖
(
𝑡
+
1
)
𝑦
𝑖
(
𝑡
+
1
)
⊤

A common choice for the learning rate is 
𝛼
𝑐
𝜆
≈
min
(
1
,
𝜆
/
𝑛
2
)
.

The second route tries to solve the issue that 
𝑦
𝑖
𝑦
𝑖
⊤
=
(
−
𝑦
𝑖
)
(
−
𝑦
𝑖
)
⊤
 loses the sign information. Similar to how we adjust the step size 
𝜎
, an evolution path 
𝑝
𝑐
 is used to track the sign information and it is constructed in a way that 
𝑝
𝑐
 is conjugate, 
∼
𝑁
(
0
,
𝐶
)
 both before and after a new generation.

We may consider 
𝑝
𝑐
 as another way to compute 
avg
𝑖
(
𝑦
𝑖
)
 (notice that both 
∼
𝑁
(
0
,
𝐶
)
) while the entire history is used and the sign information is maintained. Note that we’ve known 
𝑘
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)
∼
𝑁
(
0
,
𝐶
)
 in the last section,

	

	
𝑝
𝑐
(
𝑡
+
1
)
	
=
(
1
−
𝛼
𝑐
𝑝
)
𝑝
𝑐
(
𝑡
)
+
1
−
(
1
−
𝛼
𝑐
𝑝
)
2
𝜆
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)

	
=
(
1
−
𝛼
𝑐
𝑝
)
𝑝
𝑐
(
𝑡
)
+
𝛼
𝑐
𝑝
(
2
−
𝛼
𝑐
𝑝
)
𝜆
𝜇
(
𝑡
+
1
)
−
𝜇
(
𝑡
)
𝜎
(
𝑡
)

Then the covariance matrix is updated according to 
𝑝
𝑐
:

𝐶
(
𝑡
+
1
)
=
(
1
−
𝛼
𝑐
1
)
𝐶
(
𝑡
)
+
𝛼
𝑐
1
𝑝
𝑐
(
𝑡
+
1
)
𝑝
𝑐
(
𝑡
+
1
)
⊤

The rank-one update approach is claimed to generate a significant improvement over the rank-min(λ, n)-update when 
𝑘
 is small, because the signs of moving steps and correlations between consecutive steps are all utilized and passed down through generations.

Eventually we combine two approaches together,


				




				

𝐶
(
𝑡
+
1
)
=
(
1
−
𝛼
𝑐
𝜆
−
𝛼
𝑐
1
)
𝐶
(
𝑡
)
+
𝛼
𝑐
1
𝑝
𝑐
(
𝑡
+
1
)
𝑝
𝑐
(
𝑡
+
1
)
⊤
⏟
rank-one update
+
𝛼
𝑐
𝜆
1
𝜆
∑
𝑖
=
1
𝜆
𝑦
𝑖
(
𝑡
+
1
)
𝑦
𝑖
(
𝑡
+
1
)
⊤
⏟
rank-min(lambda, n) update

In all my examples above, each elite sample is considered to contribute an equal amount of weights, 
1
/
𝜆
. The process can be easily extended to the case where selected samples are assigned with different weights, 
𝑤
1
,
…
,
𝑤
𝜆
, according to their performances. See more detail in tutorial.

Illustration of how CMA-ES works on a 2D optimization problem (the lighter color the better). Black dots are samples in one generation. The samples are more spread out initially but when the model has higher confidence in finding a good solution in the late stage, the samples become very concentrated over the global optimum. (Image source: Wikipedia CMA-ES)
Natural Evolution Strategies

Natural Evolution Strategies (NES; Wierstra, et al, 2008) optimizes in a search distribution of parameters and moves the distribution in the direction of high fitness indicated by the natural gradient.

Natural Gradients

Given an objective function 
𝐽
(
𝜃
)
 parameterized by 
𝜃
, let’s say our goal is to find the optimal 
𝜃
 to maximize the objective function value. A plain gradient finds the steepest direction within a small Euclidean distance from the current 
𝜃
; the distance restriction is applied on the parameter space. In other words, we compute the plain gradient with respect to a small change of the absolute value of 
𝜃
. The optimal step is:



𝑑
∗
=
argmax
‖
𝑑
‖
=
𝜖
𝐽
(
𝜃
+
𝑑
)
, where 
𝜖
→
0

Differently, natural gradient works with a probability distribution space parameterized by 
𝜃
, 
𝑝
𝜃
(
𝑥
)
 (referred to as “search distribution” in NES paper). It looks for the steepest direction within a small step in the distribution space where the distance is measured by KL divergence. With this constraint we ensure that each update is moving along the distributional manifold with constant speed, without being slowed down by its curvature.



𝑑
N
∗
=
argmax
KL
[
𝑝
𝜃
‖
𝑝
𝜃
+
𝑑
]
=
𝜖
𝐽
(
𝜃
+
𝑑
)
Estimation using Fisher Information Matrix

But, how to compute 
KL
[
𝑝
𝜃
|
𝑝
𝜃
+
Δ
𝜃
]
 precisely? By running Taylor expansion of 
log
⁡
𝑝
𝜃
+
𝑑
 at 
𝜃
, we get:

		
		
	
	
	
	
	
KL
[
𝑝
𝜃
‖
𝑝
𝜃
+
𝑑
]

	
=
𝐸
𝑥
∼
𝑝
𝜃
[
log
⁡
𝑝
𝜃
(
𝑥
)
−
log
⁡
𝑝
𝜃
+
𝑑
(
𝑥
)
]
	
	
≈
𝐸
𝑥
∼
𝑝
𝜃
[
log
⁡
𝑝
𝜃
(
𝑥
)
−
(
log
⁡
𝑝
𝜃
(
𝑥
)
+
∇
𝜃
log
⁡
𝑝
𝜃
(
𝑥
)
𝑑
+
1
2
𝑑
⊤
∇
𝜃
2
log
⁡
𝑝
𝜃
(
𝑥
)
𝑑
)
]
	
; Taylor expand 
log
⁡
𝑝
𝜃
+
𝑑

	
≈
−
𝐸
𝑥
[
∇
𝜃
log
⁡
𝑝
𝜃
(
𝑥
)
]
𝑑
−
1
2
𝑑
⊤
𝐸
𝑥
[
∇
𝜃
2
log
⁡
𝑝
𝜃
(
𝑥
)
]
𝑑
	

where

		
	
	
		
		
𝐸
𝑥
[
∇
𝜃
log
⁡
𝑝
𝜃
]
𝑑
	
=
∫
𝑥
∼
𝑝
𝜃
𝑝
𝜃
(
𝑥
)
∇
𝜃
log
⁡
𝑝
𝜃
(
𝑥
)
	
	
=
∫
𝑥
∼
𝑝
𝜃
𝑝
𝜃
(
𝑥
)
1
𝑝
𝜃
(
𝑥
)
∇
𝜃
𝑝
𝜃
(
𝑥
)
	
	
=
∇
𝜃
(
∫
𝑥
𝑝
𝜃
(
𝑥
)
)
	
; note that 
𝑝
𝜃
(
𝑥
)
 is probability distribution.

	
=
∇
𝜃
(
1
)
=
0

Finally we have,

KL
[
𝑝
𝜃
‖
𝑝
𝜃
+
𝑑
]
=
−
1
2
𝑑
⊤
𝐹
𝜃
𝑑
, where 
𝐹
𝜃
=
𝐸
𝑥
[
(
∇
𝜃
log
⁡
𝑝
𝜃
)
(
∇
𝜃
log
⁡
𝑝
𝜃
)
⊤
]

where 
𝐹
𝜃
 is called the Fisher Information Matrix and it is the covariance matrix of 
∇
𝜃
log
⁡
𝑝
𝜃
 since 
𝐸
[
∇
𝜃
log
⁡
𝑝
𝜃
]
=
0
.

The solution to the following optimization problem:

max
𝐽
(
𝜃
+
𝑑
)
≈
max
(
𝐽
(
𝜃
)
+
∇
𝜃
𝐽
(
𝜃
)
⊤
𝑑
)
 s.t. 
KL
[
𝑝
𝜃
‖
𝑝
𝜃
+
𝑑
]
−
𝜖
=
0

can be found using a Lagrangian multiplier,

	

	

	
𝐿
(
𝜃
,
𝑑
,
𝛽
)
	
=
𝐽
(
𝜃
)
+
∇
𝜃
𝐽
(
𝜃
)
⊤
𝑑
−
𝛽
(
1
2
𝑑
⊤
𝐹
𝜃
𝑑
+
𝜖
)
=
0
 s.t. 
𝛽
>
0


∇
𝑑
𝐿
(
𝜃
,
𝑑
,
𝛽
)
	
=
∇
𝜃
𝐽
(
𝜃
)
−
𝛽
𝐹
𝜃
𝑑
=
0


Thus 
𝑑
N
∗
	
=
∇
𝜃
N
𝐽
(
𝜃
)
=
𝐹
𝜃
−
1
∇
𝜃
𝐽
(
𝜃
)

where 
𝑑
N
∗
 only extracts the direction of the optimal moving step on 
𝜃
, ignoring the scalar 
𝛽
−
1
.

The natural gradient samples (black solid arrows) in the right are the plain gradient samples (black solid arrows) in the left multiplied by the inverse of their covariance. In this way, a gradient direction with high uncertainty (indicated by high covariance with other samples) are penalized with a small weight. The aggregated natural gradient (red dash arrow) is therefore more trustworthy than the natural gradient (green solid arrow). (Image source: additional annotations on Fig 2 in NES paper)
NES Algorithm

The fitness associated with one sample is labeled as 
𝑓
(
𝑥
)
 and the search distribution over 
𝑥
 is parameterized by 
𝜃
. NES is expected to optimize the parameter 
𝜃
 to achieve maximum expected fitness:

𝐽
(
𝜃
)
=
𝐸
𝑥
∼
𝑝
𝜃
(
𝑥
)
[
𝑓
(
𝑥
)
]
=
∫
𝑥
𝑓
(
𝑥
)
𝑝
𝜃
(
𝑥
)
𝑑
𝑥

Using the same log-likelihood trick in REINFORCE:

	
	

	
	
∇
𝜃
𝐽
(
𝜃
)
	
=
∇
𝜃
∫
𝑥
𝑓
(
𝑥
)
𝑝
𝜃
(
𝑥
)
𝑑
𝑥

	
=
∫
𝑥
𝑓
(
𝑥
)
𝑝
𝜃
(
𝑥
)
𝑝
𝜃
(
𝑥
)
∇
𝜃
𝑝
𝜃
(
𝑥
)
𝑑
𝑥

	
=
∫
𝑥
𝑓
(
𝑥
)
𝑝
𝜃
(
𝑥
)
∇
𝜃
log
⁡
𝑝
𝜃
(
𝑥
)
𝑑
𝑥

	
=
𝐸
𝑥
∼
𝑝
𝜃
[
𝑓
(
𝑥
)
∇
𝜃
log
⁡
𝑝
𝜃
(
𝑥
)
]

Besides natural gradients, NES adopts a couple of important heuristics to make the algorithm performance more robust.

NES applies rank-based fitness shaping, that is to use the rank under monotonically increasing fitness values instead of using 
𝑓
(
𝑥
)
 directly. Or it can be a function of the rank (“utility function”), which is considered as a free parameter of NES.
NES adopts adaptation sampling to adjust hyperparameters at run time. When changing 
𝜃
→
𝜃
′
, samples drawn from 
𝑝
𝜃
 are compared with samples from 
𝑝
𝜃
′
 using [Mann-Whitney U-test(https://en.wikipedia.org/wiki/Mann%E2%80%93Whitney_U_test)]; if there shows a positive or negative sign, the target hyperparameter decreases or increases by a multiplication constant. Note the score of a sample 
𝑥
𝑖
′
∼
𝑝
𝜃
′
(
𝑥
)
 has importance sampling weights applied 
𝑤
𝑖
′
=
𝑝
𝜃
(
𝑥
)
/
𝑝
𝜃
′
(
𝑥
)
.
Applications: ES in Deep Reinforcement Learning
OpenAI ES for RL

The concept of using evolutionary algorithms in reinforcement learning can be traced back long ago, but only constrained to tabular RL due to computational limitations.

Inspired by NES, researchers at OpenAI (Salimans, et al. 2017) proposed to use NES as a gradient-free black-box optimizer to find optimal policy parameters 
𝜃
 that maximizes the return function 
𝐹
(
𝜃
)
. The key is to add Gaussian noise 
𝜖
 on the model parameter 
𝜃
 and then use the log-likelihood trick to write it as the gradient of the Gaussian pdf. Eventually only the noise term is left as a weighting scalar for measured performance.

Let’s say the current parameter value is 
𝜃
^
 (the added hat is to distinguish the value from the random variable 
𝜃
). The search distribution of 
𝜃
 is designed to be an isotropic multivariate Gaussian with a mean 
𝜃
^
 and a fixed covariance matrix 
𝜎
2
𝐼
,

𝜃
∼
𝑁
(
𝜃
^
,
𝜎
2
𝐼
)
 equivalent to 
𝜃
=
𝜃
^
+
𝜎
𝜖
,
𝜖
∼
𝑁
(
0
,
𝐼
)

The gradient for 
𝜃
 update is:

	
	
	
	
	
	

	
	
	
	
	
	
	
	
	
∇
𝜃
𝐸
𝜃
∼
𝑁
(
𝜃
^
,
𝜎
2
𝐼
)
𝐹
(
𝜃
)

	
=
∇
𝜃
𝐸
𝜖
∼
𝑁
(
0
,
𝐼
)
𝐹
(
𝜃
^
+
𝜎
𝜖
)

	
=
∇
𝜃
∫
𝜖
𝑝
(
𝜖
)
𝐹
(
𝜃
^
+
𝜎
𝜖
)
𝑑
𝜖
	
; Gaussian 
𝑝
(
𝜖
)
=
(
2
𝜋
)
−
𝑛
2
exp
⁡
(
−
1
2
𝜖
⊤
𝜖
)

	
=
∫
𝜖
𝑝
(
𝜖
)
∇
𝜖
log
⁡
𝑝
(
𝜖
)
∇
𝜃
𝜖
𝐹
(
𝜃
^
+
𝜎
𝜖
)
𝑑
𝜖
	
; log-likelihood trick

	
=
𝐸
𝜖
∼
𝑁
(
0
,
𝐼
)
[
∇
𝜖
(
−
1
2
𝜖
⊤
𝜖
)
∇
𝜃
(
𝜃
−
𝜃
^
𝜎
)
𝐹
(
𝜃
^
+
𝜎
𝜖
)
]
	
	
=
𝐸
𝜖
∼
𝑁
(
0
,
𝐼
)
[
(
−
𝜖
)
(
1
𝜎
)
𝐹
(
𝜃
^
+
𝜎
𝜖
)
]
	
	
=
1
𝜎
𝐸
𝜖
∼
𝑁
(
0
,
𝐼
)
[
𝜖
𝐹
(
𝜃
^
+
𝜎
𝜖
)
]
	
; negative sign can be absorbed.

In one generation, we can sample many 
𝑒
𝑝
𝑠
𝑖
𝑙
𝑜
𝑛
𝑖
,
𝑖
=
1
,
…
,
𝑛
 and evaluate the fitness in parallel. One beautiful design is that no large model parameter needs to be shared. By only communicating the random seeds between workers, it is enough for the master node to do parameter update. This approach is later extended to adaptively learn a loss function; see my previous post on Evolved Policy Gradient.

The algorithm for training a RL policy using evolution strategies. (Image source: ES-for-RL paper)

To make the performance more robust, OpenAI ES adopts virtual batch normalization (BN with mini-batch used for calculating statistics fixed), mirror sampling (sampling a pair of 
(
−
𝜖
,
𝜖
)
 for evaluation), and fitness shaping.

Exploration with ES

Exploration (vs exploitation) is an important topic in RL. The optimization direction in the ES algorithm above is only extracted from the cumulative return 
𝐹
(
𝜃
)
. Without explicit exploration, the agent might get trapped in a local optimum.

Novelty-Search ES (NS-ES; Conti et al, 2018) encourages exploration by updating the parameter in the direction to maximize the novelty score. The novelty score depends on a domain-specific behavior characterization function 
𝑏
(
𝜋
𝜃
)
. The choice of 
𝑏
(
𝜋
𝜃
)
 is specific to the task and seems to be a bit arbitrary; for example, in the Humanoid locomotion task in the paper, 
𝑏
(
𝜋
𝜃
)
 is the final 
(
𝑥
,
𝑦
)
 location of the agent.

Every policy’s 
𝑏
(
𝜋
𝜃
)
 is pushed to an archive set 
𝐴
.
Novelty of a policy 
𝜋
𝜃
 is measured as the k-nearest neighbor score between 
𝑏
(
𝜋
𝜃
)
 and all other entries in 
𝐴
. (The use case of the archive set sounds quite similar to episodic memory.)


𝑁
(
𝜃
,
𝐴
)
=
1
𝜆
∑
𝑖
=
1
𝜆
‖
𝑏
(
𝜋
𝜃
)
,
𝑏
𝑖
knn
‖
2
, where 
𝑏
𝑖
knn
∈
kNN
(
𝑏
(
𝜋
𝜃
)
,
𝐴
)

The ES optimization step relies on the novelty score instead of fitness:

∇
𝜃
𝐸
𝜃
∼
𝑁
(
𝜃
^
,
𝜎
2
𝐼
)
𝑁
(
𝜃
,
𝐴
)
=
1
𝜎
𝐸
𝜖
∼
𝑁
(
0
,
𝐼
)
[
𝜖
𝑁
(
𝜃
^
+
𝜎
𝜖
,
𝐴
)
]

NS-ES maintains a group of 
𝑀
 independently trained agents (“meta-population”), 
𝑀
=
{
𝜃
1
,
…
,
𝜃
𝑀
}
 and picks one to advance proportional to the novelty score. Eventually we select the best policy. This process is equivalent to ensembling; also see the same idea in SVPG.

	


	


𝑚
	
←
pick 
𝑖
=
1
,
…
,
𝑀
 according to probability
𝑁
(
𝜃
𝑖
,
𝐴
)
∑
𝑗
=
1
𝑀
𝑁
(
𝜃
𝑗
,
𝐴
)


𝜃
𝑚
(
𝑡
+
1
)
	
←
𝜃
𝑚
(
𝑡
)
+
𝛼
1
𝜎
∑
𝑖
=
1
𝑁
𝜖
𝑖
𝑁
(
𝜃
𝑚
(
𝑡
)
+
𝜖
𝑖
,
𝐴
)
 where 
𝜖
𝑖
∼
𝑁
(
0
,
𝐼
)

where 
𝑁
 is the number of Gaussian perturbation noise vectors and 
𝛼
 is the learning rate.

NS-ES completely discards the reward function and only optimizes for novelty to avoid deceptive local optima. To incorporate the fitness back into the formula, another two variations are proposed.

NSR-ES:



𝜃
𝑚
(
𝑡
+
1
)
←
𝜃
𝑚
(
𝑡
)
+
𝛼
1
𝜎
∑
𝑖
=
1
𝑁
𝜖
𝑖
𝑁
(
𝜃
𝑚
(
𝑡
)
+
𝜖
𝑖
,
𝐴
)
+
𝐹
(
𝜃
𝑚
(
𝑡
)
+
𝜖
𝑖
)
2

NSRAdapt-ES (NSRA-ES): the adaptive weighting parameter 
𝑤
=
1.0
 initially. We start decreasing 
𝑤
 if performance stays flat for a number of generations. Then when the performance starts to increase, we stop decreasing 
𝑤
 but increase it instead. In this way, fitness is preferred when the performance stops growing but novelty is preferred otherwise.



𝜃
𝑚
(
𝑡
+
1
)
←
𝜃
𝑚
(
𝑡
)
+
𝛼
1
𝜎
∑
𝑖
=
1
𝑁
𝜖
𝑖
(
(
1
−
𝑤
)
𝑁
(
𝜃
𝑚
(
𝑡
)
+
𝜖
𝑖
,
𝐴
)
+
𝑤
𝐹
(
𝜃
𝑚
(
𝑡
)
+
𝜖
𝑖
)
)
(Left) The environment is Humanoid locomotion with a three-sided wall which plays a role as a deceptive trap to create local optimum. (Right) Experiments compare ES baseline and other variations that encourage exploration. (Image source: NS-ES paper)
CEM-RL
Architectures of the (a) CEM-RL and (b) ERL algorithms (Image source: CEM-RL paper)

The CEM-RL method (Pourchot & Sigaud, 2019) combines Cross Entropy Method (CEM) with either DDPG or TD3. CEM here works pretty much the same as the simple Gaussian ES described above and therefore the same function can be replaced using CMA-ES. CEM-RL is built on the framework of Evolutionary Reinforcement Learning (ERL; Khadka & Tumer, 2018) in which the standard EA algorithm selects and evolves a population of actors and the rollout experience generated in the process is then added into reply buffer for training both RL-actor and RL-critic networks.

Workflow:

The mean actor of the CEM population is 
𝜋
𝜇
 is initialized with a random actor network.
The critic network 
𝑄
 is initialized too, which will be updated by DDPG/TD3.
Repeat until happy:
a. Sample a population of actors 
∼
𝑁
(
𝜋
𝜇
,
Σ
)
.
b. Half of the population is evaluated. Their fitness scores are used as the cumulative reward 
𝑅
 and added into replay buffer.
c. The other half are updated together with the critic.
d. The new 
𝜋
𝑚
𝑢
 and 
Σ
 is computed using top performing elite samples. CMA-ES can be used for parameter update too.
Extension: EA in Deep Learning

(This section is not on evolution strategies, but still an interesting and relevant reading.)

The Evolutionary Algorithms have been applied on many deep learning problems. POET (Wang et al, 2019) is a framework based on EA and attempts to generate a variety of different tasks while the problems themselves are being solved. POET has been introduced in my last post on meta-RL. Evolutionary Reinforcement Learning (ERL) is another example; See Fig. 7 (b).

Below I would like to introduce two applications in more detail, Population-Based Training (PBT) and Weight-Agnostic Neural Networks (WANN).

Hyperparameter Tuning: PBT
Paradigms of comparing different ways of hyperparameter tuning. (Image source: PBT paper)

Population-Based Training (Jaderberg, et al, 2017), short for PBT applies EA on the problem of hyperparameter tuning. It jointly trains a population of models and corresponding hyperparameters for optimal performance.

PBT starts with a set of random candidates, each containing a pair of model weights initialization and hyperparameters, 
{
(
𝜃
𝑖
,
ℎ
𝑖
)
∣
𝑖
=
1
,
…
,
𝑁
}
. Every sample is trained in parallel and asynchronously evaluates its own performance periodically. Whenever a member deems ready (i.e. after taking enough gradient update steps, or when the performance is good enough), it has a chance to be updated by comparing with the whole population:

exploit(): When this model is under-performing, the weights could be replaced with a better performing model.
explore(): If the model weights are overwritten, explore step perturbs the hyperparameters with random noise.

In this process, only promising model and hyperparameter pairs can survive and keep on evolving, achieving better utilization of computational resources.

The algorithm of population-based training. (Image source: PBT paper)
Network Topology Optimization: WANN

Weight Agnostic Neural Networks (short for WANN; Gaier & Ha 2019) experiments with searching for the smallest network topologies that can achieve the optimal performance without training the network weights. By not considering the best configuration of network weights, WANN puts much more emphasis on the architecture itself, making the focus different from NAS. WANN is heavily inspired by a classic genetic algorithm to evolve network topologies, called NEAT (“Neuroevolution of Augmenting Topologies”; Stanley & Miikkulainen 2002).

The workflow of WANN looks pretty much the same as standard GA:

Initialize: Create a population of minimal networks.
Evaluation: Test with a range of shared weight values.
Rank and Selection: Rank by performance and complexity.
Mutation: Create new population by varying best networks.
mutation operations for searching for new network topologies in WANN (Image source: WANN paper)

At the “evaluation” stage, all the network weights are set to be the same. In this way, WANN is actually searching for network that can be described with a minimal description length. In the “selection” stage, both the network connection and the model performance are considered.

Performance of WANN found network topologies on different RL tasks are compared with baseline FF networks commonly used in the literature. "Tuned Shared Weight" only requires adjusting one weight value. (Image source: WANN paper)

As shown in Fig. 11, WANN results are evaluated with both random weights and shared weights (single weight). It is interesting that even when enforcing weight-sharing on all weights and tuning this single parameter, WANN can discover topologies that achieve non-trivial good performance.

Cited as:

@article{weng2019ES,
  title   = "Evolution Strategies",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2019",
  url     = "https://lilianweng.github.io/posts/2019-09-05-evolution-strategies/"
}

References

[1] Nikolaus Hansen. “The CMA Evolution Strategy: A Tutorial” arXiv preprint arXiv:1604.00772 (2016).

[2] Marc Toussaint. Slides: “Introduction to Optimization”

[3] David Ha. “A Visual Guide to Evolution Strategies” blog.otoro.net. Oct 2017.

[4] Daan Wierstra, et al. “Natural evolution strategies.” IEEE World Congress on Computational Intelligence, 2008.

[5] Agustinus Kristiadi. “Natural Gradient Descent” Mar 2018.

[6] Razvan Pascanu & Yoshua Bengio. “Revisiting Natural Gradient for Deep Networks.” arXiv preprint arXiv:1301.3584 (2013).

[7] Tim Salimans, et al. “Evolution strategies as a scalable alternative to reinforcement learning.” arXiv preprint arXiv:1703.03864 (2017).

[8] Edoardo Conti, et al. “Improving exploration in evolution strategies for deep reinforcement learning via a population of novelty-seeking agents.” NIPS. 2018.

[9] Aloïs Pourchot & Olivier Sigaud. “CEM-RL: Combining evolutionary and gradient-based methods for policy search.” ICLR 2019.

[10] Shauharda Khadka & Kagan Tumer. “Evolution-guided policy gradient in reinforcement learning.” NIPS 2018.

[11] Max Jaderberg, et al. “Population based training of neural networks.” arXiv preprint arXiv:1711.09846 (2017).

[12] Adam Gaier & David Ha. “Weight Agnostic Neural Networks.” arXiv preprint arXiv:1906.04358 (2019).

Evolution
 
Reinforcement-Learning
«
Self-Supervised Representation Learning
»
Meta Reinforcement Learning
© 2026 Lil'Log Powered by Hugo & PaperMod