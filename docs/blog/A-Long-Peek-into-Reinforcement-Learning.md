---
title: A (Long) Peek into Reinforcement Learning
url: https://lilianweng.github.io/posts/2018-02-19-rl-overview/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:47:59.324931+00:00'
---

Lil'Log
|
Posts
Archive
Search
Tags
FAQ
A (Long) Peek into Reinforcement Learning
Date: February 19, 2018 | Estimated Reading Time: 31 min | Author: Lilian Weng
Table of Contents

[Updated on 2020-09-03: Updated the algorithm of SARSA and Q-learning so that the difference is more pronounced.
[Updated on 2021-09-19: Thanks to 爱吃猫的鱼, we have this post in Chinese].

A couple of exciting news in Artificial Intelligence (AI) has just happened in recent years. AlphaGo defeated the best professional human player in the game of Go. Very soon the extended algorithm AlphaGo Zero beat AlphaGo by 100-0 without supervised learning on human knowledge. Top professional game players lost to the bot developed by OpenAI on DOTA2 1v1 competition. After knowing these, it is pretty hard not to be curious about the magic behind these algorithms — Reinforcement Learning (RL). I’m writing this post to briefly go over the field. We will first introduce several fundamental concepts and then dive into classic approaches to solving RL problems. Hopefully, this post could be a good starting point for newbies, bridging the future study on the cutting-edge research.

What is Reinforcement Learning?

Say, we have an agent in an unknown environment and this agent can obtain some rewards by interacting with the environment. The agent ought to take actions so as to maximize cumulative rewards. In reality, the scenario could be a bot playing a game to achieve high scores, or a robot trying to complete physical tasks with physical items; and not just limited to these.

An agent interacts with the environment, trying to take smart actions to maximize cumulative rewards.

The goal of Reinforcement Learning (RL) is to learn a good strategy for the agent from experimental trials and relative simple feedback received. With the optimal strategy, the agent is capable to actively adapt to the environment to maximize future rewards.

Key Concepts

Now Let’s formally define a set of key concepts in RL.

The agent is acting in an environment. How the environment reacts to certain actions is defined by a model which we may or may not know. The agent can stay in one of many states (
𝑠
∈
𝑆
) of the environment, and choose to take one of many actions (
𝑎
∈
𝐴
) to switch from one state to another. Which state the agent will arrive in is decided by transition probabilities between states (
𝑃
). Once an action is taken, the environment delivers a reward (
𝑟
∈
𝑅
) as feedback.

The model defines the reward function and transition probabilities. We may or may not know how the model works and this differentiate two circumstances:

Know the model: planning with perfect information; do model-based RL. When we fully know the environment, we can find the optimal solution by Dynamic Programming (DP). Do you still remember “longest increasing subsequence” or “traveling salesmen problem” from your Algorithms 101 class? LOL. This is not the focus of this post though.
Does not know the model: learning with incomplete information; do model-free RL or try to learn the model explicitly as part of the algorithm. Most of the following content serves the scenarios when the model is unknown.

The agent’s policy 
𝜋
(
𝑠
)
 provides the guideline on what is the optimal action to take in a certain state with the goal to maximize the total rewards. Each state is associated with a value function 
𝑉
(
𝑠
)
 predicting the expected amount of future rewards we are able to receive in this state by acting the corresponding policy. In other words, the value function quantifies how good a state is. Both policy and value functions are what we try to learn in reinforcement learning.

Summary of approaches in RL based on whether we want to model the value, policy, or the environment. (Image source: reproduced from David Silver's RL course lecture 1.)

The interaction between the agent and the environment involves a sequence of actions and observed rewards in time, 
𝑡
=
1
,
2
,
…
,
𝑇
. During the process, the agent accumulates the knowledge about the environment, learns the optimal policy, and makes decisions on which action to take next so as to efficiently learn the best policy. Let’s label the state, action, and reward at time step t as 
𝑆
𝑡
, 
𝐴
𝑡
, and 
𝑅
𝑡
, respectively. Thus the interaction sequence is fully described by one episode (also known as “trial” or “trajectory”) and the sequence ends at the terminal state 
𝑆
𝑇
:

𝑆
1
,
𝐴
1
,
𝑅
2
,
𝑆
2
,
𝐴
2
,
…
,
𝑆
𝑇

Terms you will encounter a lot when diving into different categories of RL algorithms:

Model-based: Rely on the model of the environment; either the model is known or the algorithm learns it explicitly.
Model-free: No dependency on the model during learning.
On-policy: Use the deterministic outcomes or samples from the target policy to train the algorithm.
Off-policy: Training on a distribution of transitions or episodes produced by a different behavior policy rather than that produced by the target policy.
Model: Transition and Reward

The model is a descriptor of the environment. With the model, we can learn or infer how the environment would interact with and provide feedback to the agent. The model has two major parts, transition probability function 
𝑃
 and reward function 
𝑅
.

Let’s say when we are in state s, we decide to take action a to arrive in the next state s’ and obtain reward r. This is known as one transition step, represented by a tuple (s, a, s’, r).

The transition function P records the probability of transitioning from state s to s’ after taking action a while obtaining reward r. We use 
𝑃
 as a symbol of “probability”.

𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)
=
𝑃
[
𝑆
𝑡
+
1
=
𝑠
′
,
𝑅
𝑡
+
1
=
𝑟
|
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]

Thus the state-transition function can be defined as a function of 
𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)
:



𝑃
𝑠
𝑠
′
𝑎
=
𝑃
(
𝑠
′
|
𝑠
,
𝑎
)
=
𝑃
[
𝑆
𝑡
+
1
=
𝑠
′
|
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
=
∑
𝑟
∈
𝑅
𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)

The reward function R predicts the next reward triggered by one action:




𝑅
(
𝑠
,
𝑎
)
=
𝐸
[
𝑅
𝑡
+
1
|
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
=
∑
𝑟
∈
𝑅
𝑟
∑
𝑠
′
∈
𝑆
𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)
Policy

Policy, as the agent’s behavior function 
𝜋
, tells us which action to take in state s. It is a mapping from state s to action a and can be either deterministic or stochastic:

Deterministic: 
𝜋
(
𝑠
)
=
𝑎
.
Stochastic: 
𝜋
(
𝑎
|
𝑠
)
=
𝑃
𝜋
[
𝐴
=
𝑎
|
𝑆
=
𝑠
]
.
Value Function

Value function measures the goodness of a state or how rewarding a state or an action is by a prediction of future reward. The future reward, also known as return, is a total sum of discounted rewards going forward. Let’s compute the return 
𝐺
𝑡
 starting from time t:



𝐺
𝑡
=
𝑅
𝑡
+
1
+
𝛾
𝑅
𝑡
+
2
+
⋯
=
∑
𝑘
=
0
∞
𝛾
𝑘
𝑅
𝑡
+
𝑘
+
1

The discounting factor 
𝛾
∈
[
0
,
1
]
 penalize the rewards in the future, because:

The future rewards may have higher uncertainty; i.e. stock market.
The future rewards do not provide immediate benefits; i.e. As human beings, we might prefer to have fun today rather than 5 years later ;).
Discounting provides mathematical convenience; i.e., we don’t need to track future steps forever to compute return.
We don’t need to worry about the infinite loops in the state transition graph.

The state-value of a state s is the expected return if we are in this state at time t, 
𝑆
𝑡
=
𝑠
:

𝑉
𝜋
(
𝑠
)
=
𝐸
𝜋
[
𝐺
𝑡
|
𝑆
𝑡
=
𝑠
]

Similarly, we define the action-value (“Q-value”; Q as “Quality” I believe?) of a state-action pair as:

𝑄
𝜋
(
𝑠
,
𝑎
)
=
𝐸
𝜋
[
𝐺
𝑡
|
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]

Additionally, since we follow the target policy 
𝜋
, we can make use of the probility distribution over possible actions and the Q-values to recover the state-value:



𝑉
𝜋
(
𝑠
)
=
∑
𝑎
∈
𝐴
𝑄
𝜋
(
𝑠
,
𝑎
)
𝜋
(
𝑎
|
𝑠
)

The difference between action-value and state-value is the action advantage function (“A-value”):

𝐴
𝜋
(
𝑠
,
𝑎
)
=
𝑄
𝜋
(
𝑠
,
𝑎
)
−
𝑉
𝜋
(
𝑠
)
Optimal Value and Policy

The optimal value function produces the maximum return:




𝑉
∗
(
𝑠
)
=
max
𝜋
𝑉
𝜋
(
𝑠
)
,
𝑄
∗
(
𝑠
,
𝑎
)
=
max
𝜋
𝑄
𝜋
(
𝑠
,
𝑎
)

The optimal policy achieves optimal value functions:




𝜋
∗
=
arg
⁡
max
𝜋
𝑉
𝜋
(
𝑠
)
,
𝜋
∗
=
arg
⁡
max
𝜋
𝑄
𝜋
(
𝑠
,
𝑎
)

And of course, we have 
𝑉
𝜋
∗
(
𝑠
)
=
𝑉
∗
(
𝑠
)
 and 
𝑄
𝜋
∗
(
𝑠
,
𝑎
)
=
𝑄
∗
(
𝑠
,
𝑎
)
.

Markov Decision Processes

In more formal terms, almost all the RL problems can be framed as Markov Decision Processes (MDPs). All states in MDP has “Markov” property, referring to the fact that the future only depends on the current state, not the history:

𝑃
[
𝑆
𝑡
+
1
|
𝑆
𝑡
]
=
𝑃
[
𝑆
𝑡
+
1
|
𝑆
1
,
…
,
𝑆
𝑡
]

Or in other words, the future and the past are conditionally independent given the present, as the current state encapsulates all the statistics we need to decide the future.

The agent-environment interaction in a Markov decision process. (Image source: Sec. 3.1 Sutton & Barto (2017).)

A Markov deicison process consists of five elements 
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
, where the symbols carry the same meanings as key concepts in the previous section, well aligned with RL problem settings:

𝑆
 - a set of states;
𝐴
 - a set of actions;
𝑃
 - transition probability function;
𝑅
 - reward function;
𝛾
 - discounting factor for future rewards. In an unknown environment, we do not have perfect knowledge about 
𝑃
 and 
𝑅
.
A fun example of Markov decision process: a typical work day. (Image source: randomant.net/reinforcement-learning-concepts)
Bellman Equations

Bellman equations refer to a set of equations that decompose the value function into the immediate reward plus the discounted future values.

	
	
	
	
	
𝑉
(
𝑠
)
	
=
𝐸
[
𝐺
𝑡
|
𝑆
𝑡
=
𝑠
]

	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝑅
𝑡
+
2
+
𝛾
2
𝑅
𝑡
+
3
+
…
|
𝑆
𝑡
=
𝑠
]

	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
(
𝑅
𝑡
+
2
+
𝛾
𝑅
𝑡
+
3
+
…
)
|
𝑆
𝑡
=
𝑠
]

	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝐺
𝑡
+
1
|
𝑆
𝑡
=
𝑠
]

	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝑉
(
𝑆
𝑡
+
1
)
|
𝑆
𝑡
=
𝑠
]

Similarly for Q-value,

	
	
𝑄
(
𝑠
,
𝑎
)
	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝑉
(
𝑆
𝑡
+
1
)
∣
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]

	
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝐸
𝑎
∼
𝜋
𝑄
(
𝑆
𝑡
+
1
,
𝑎
)
∣
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
Bellman Expectation Equations

The recursive update process can be further decomposed to be equations built on both state-value and action-value functions. As we go further in future action steps, we extend V and Q alternatively by following the policy 
𝜋
.

Illustration of how Bellman expection equations update state-value and action-value functions.
	

	


	



	



𝑉
𝜋
(
𝑠
)
	
=
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
)
𝑄
𝜋
(
𝑠
,
𝑎
)


𝑄
𝜋
(
𝑠
,
𝑎
)
	
=
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
𝑉
𝜋
(
𝑠
′
)


𝑉
𝜋
(
𝑠
)
	
=
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
)
(
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
𝑉
𝜋
(
𝑠
′
)
)


𝑄
𝜋
(
𝑠
,
𝑎
)
	
=
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
∑
𝑎
′
∈
𝐴
𝜋
(
𝑎
′
|
𝑠
′
)
𝑄
𝜋
(
𝑠
′
,
𝑎
′
)
Bellman Optimality Equations

If we are only interested in the optimal values, rather than computing the expectation following a policy, we could jump right into the maximum returns during the alternative updates without using a policy. RECAP: the optimal values 
𝑉
∗
 and 
𝑄
∗
 are the best returns we can obtain, defined here.

	

	


	



	



𝑉
∗
(
𝑠
)
	
=
max
𝑎
∈
𝐴
𝑄
∗
(
𝑠
,
𝑎
)


𝑄
∗
(
𝑠
,
𝑎
)
	
=
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
𝑉
∗
(
𝑠
′
)


𝑉
∗
(
𝑠
)
	
=
max
𝑎
∈
𝐴
(
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
𝑉
∗
(
𝑠
′
)
)


𝑄
∗
(
𝑠
,
𝑎
)
	
=
𝑅
(
𝑠
,
𝑎
)
+
𝛾
∑
𝑠
′
∈
𝑆
𝑃
𝑠
𝑠
′
𝑎
max
𝑎
′
∈
𝐴
𝑄
∗
(
𝑠
′
,
𝑎
′
)

Unsurprisingly they look very similar to Bellman expectation equations.

If we have complete information of the environment, this turns into a planning problem, solvable by DP. Unfortunately, in most scenarios, we do not know 
𝑃
𝑠
𝑠
′
𝑎
 or 
𝑅
(
𝑠
,
𝑎
)
, so we cannot solve MDPs by directly applying Bellmen equations, but it lays the theoretical foundation for many RL algorithms.

Common Approaches

Now it is the time to go through the major approaches and classic algorithms for solving RL problems. In future posts, I plan to dive into each approach further.

Dynamic Programming

When the model is fully known, following Bellman equations, we can use Dynamic Programming (DP) to iteratively evaluate value functions and improve policy.

Policy Evaluation

Policy Evaluation is to compute the state-value 
𝑉
𝜋
 for a given policy 
𝜋
:




𝑉
𝑡
+
1
(
𝑠
)
=
𝐸
𝜋
[
𝑟
+
𝛾
𝑉
𝑡
(
𝑠
′
)
|
𝑆
𝑡
=
𝑠
]
=
∑
𝑎
𝜋
(
𝑎
|
𝑠
)
∑
𝑠
′
,
𝑟
𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)
(
𝑟
+
𝛾
𝑉
𝑡
(
𝑠
′
)
)
Policy Improvement

Based on the value functions, Policy Improvement generates a better policy 
𝜋
′
≥
𝜋
 by acting greedily.



𝑄
𝜋
(
𝑠
,
𝑎
)
=
𝐸
[
𝑅
𝑡
+
1
+
𝛾
𝑉
𝜋
(
𝑆
𝑡
+
1
)
|
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
=
∑
𝑠
′
,
𝑟
𝑃
(
𝑠
′
,
𝑟
|
𝑠
,
𝑎
)
(
𝑟
+
𝛾
𝑉
𝜋
(
𝑠
′
)
)
Policy Iteration

The Generalized Policy Iteration (GPI) algorithm refers to an iterative procedure to improve the policy when combining policy evaluation and improvement.

	
	
	
	
	
	
	
𝜋
0
→
evaluation
𝑉
𝜋
0
→
improve
𝜋
1
→
evaluation
𝑉
𝜋
1
→
improve
𝜋
2
→
evaluation
⋯
→
improve
𝜋
∗
→
evaluation
𝑉
∗

In GPI, the value function is approximated repeatedly to be closer to the true value of the current policy and in the meantime, the policy is improved repeatedly to approach optimality. This policy iteration process works and always converges to the optimality, but why this is the case?

Say, we have a policy 
𝜋
 and then generate an improved version 
𝜋
′
 by greedily taking actions, 
𝜋
′
(
𝑠
)
=
arg
⁡
max
𝑎
∈
𝐴
𝑄
𝜋
(
𝑠
,
𝑎
)
. The value of this improved 
𝜋
′
 is guaranteed to be better because:

	

	

𝑄
𝜋
(
𝑠
,
𝜋
′
(
𝑠
)
)
	
=
𝑄
𝜋
(
𝑠
,
arg
⁡
max
𝑎
∈
𝐴
𝑄
𝜋
(
𝑠
,
𝑎
)
)

	
=
max
𝑎
∈
𝐴
𝑄
𝜋
(
𝑠
,
𝑎
)
≥
𝑄
𝜋
(
𝑠
,
𝜋
(
𝑠
)
)
=
𝑉
𝜋
(
𝑠
)
Monte-Carlo Methods

First, let’s recall that 
𝑉
(
𝑠
)
=
𝐸
[
𝐺
𝑡
|
𝑆
𝑡
=
𝑠
]
. Monte-Carlo (MC) methods uses a simple idea: It learns from episodes of raw experience without modeling the environmental dynamics and computes the observed mean return as an approximation of the expected return. To compute the empirical return 
𝐺
𝑡
, MC methods need to learn from complete episodes 
𝑆
1
,
𝐴
1
,
𝑅
2
,
…
,
𝑆
𝑇
 to compute 
𝐺
𝑡
=
∑
𝑘
=
0
𝑇
−
𝑡
−
1
𝛾
𝑘
𝑅
𝑡
+
𝑘
+
1
 and all the episodes must eventually terminate.

The empirical mean return for state s is:

𝟙
𝟙
𝑉
(
𝑠
)
=
∑
𝑡
=
1
𝑇
1
[
𝑆
𝑡
=
𝑠
]
𝐺
𝑡
∑
𝑡
=
1
𝑇
1
[
𝑆
𝑡
=
𝑠
]

where 𝟙
1
[
𝑆
𝑡
=
𝑠
]
 is a binary indicator function. We may count the visit of state s every time so that there could exist multiple visits of one state in one episode (“every-visit”), or only count it the first time we encounter a state in one episode (“first-visit”). This way of approximation can be easily extended to action-value functions by counting (s, a) pair.

𝟙
𝟙
𝑄
(
𝑠
,
𝑎
)
=
∑
𝑡
=
1
𝑇
1
[
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
𝐺
𝑡
∑
𝑡
=
1
𝑇
1
[
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]

To learn the optimal policy by MC, we iterate it by following a similar idea to GPI.

Improve the policy greedily with respect to the current value function: 
𝜋
(
𝑠
)
=
arg
⁡
max
𝑎
∈
𝐴
𝑄
(
𝑠
,
𝑎
)
.
Generate a new episode with the new policy 
𝜋
 (i.e. using algorithms like ε-greedy helps us balance between exploitation and exploration.)
Estimate Q using the new episode: 
𝟙
𝟙
𝑞
𝜋
(
𝑠
,
𝑎
)
=
∑
𝑡
=
1
𝑇
(
1
[
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
∑
𝑘
=
0
𝑇
−
𝑡
−
1
𝛾
𝑘
𝑅
𝑡
+
𝑘
+
1
)
∑
𝑡
=
1
𝑇
1
[
𝑆
𝑡
=
𝑠
,
𝐴
𝑡
=
𝑎
]
Temporal-Difference Learning

Similar to Monte-Carlo methods, Temporal-Difference (TD) Learning is model-free and learns from episodes of experience. However, TD learning can learn from incomplete episodes and hence we don’t need to track the episode up to termination. TD learning is so important that Sutton & Barto (2017) in their RL book describes it as “one idea … central and novel to reinforcement learning”.

Bootstrapping

TD learning methods update targets with regard to existing estimates rather than exclusively relying on actual rewards and complete returns as in MC methods. This approach is known as bootstrapping.

Value Estimation

The key idea in TD learning is to update the value function 
𝑉
(
𝑆
𝑡
)
 towards an estimated return 
𝑅
𝑡
+
1
+
𝛾
𝑉
(
𝑆
𝑡
+
1
)
 (known as “TD target”). To what extent we want to update the value function is controlled by the learning rate hyperparameter α:

	
	
	
𝑉
(
𝑆
𝑡
)
	
←
(
1
−
𝛼
)
𝑉
(
𝑆
𝑡
)
+
𝛼
𝐺
𝑡


𝑉
(
𝑆
𝑡
)
	
←
𝑉
(
𝑆
𝑡
)
+
𝛼
(
𝐺
𝑡
−
𝑉
(
𝑆
𝑡
)
)


𝑉
(
𝑆
𝑡
)
	
←
𝑉
(
𝑆
𝑡
)
+
𝛼
(
𝑅
𝑡
+
1
+
𝛾
𝑉
(
𝑆
𝑡
+
1
)
−
𝑉
(
𝑆
𝑡
)
)

Similarly, for action-value estimation:

𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
←
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
+
𝛼
(
𝑅
𝑡
+
1
+
𝛾
𝑄
(
𝑆
𝑡
+
1
,
𝐴
𝑡
+
1
)
−
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
)

Next, let’s dig into the fun part on how to learn optimal policy in TD learning (aka “TD control”). Be prepared, you are gonna see many famous names of classic algorithms in this section.

SARSA: On-Policy TD control

“SARSA” refers to the procedure of updaing Q-value by following a sequence of 
…
,
𝑆
𝑡
,
𝐴
𝑡
,
𝑅
𝑡
+
1
,
𝑆
𝑡
+
1
,
𝐴
𝑡
+
1
,
…
. The idea follows the same route of GPI. Within one episode, it works as follows:

Initialize 
𝑡
=
0
.
Start with 
𝑆
0
 and choose action 
𝐴
0
=
arg
⁡
max
𝑎
∈
𝐴
𝑄
(
𝑆
0
,
𝑎
)
, where 
𝜖
-greedy is commonly applied.
At time 
𝑡
, after applying action 
𝐴
𝑡
, we observe reward 
𝑅
𝑡
+
1
 and get into the next state 
𝑆
𝑡
+
1
.
Then pick the next action in the same way as in step 2: 
𝐴
𝑡
+
1
=
arg
⁡
max
𝑎
∈
𝐴
𝑄
(
𝑆
𝑡
+
1
,
𝑎
)
.
Update the Q-value function: 
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
←
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
+
𝛼
(
𝑅
𝑡
+
1
+
𝛾
𝑄
(
𝑆
𝑡
+
1
,
𝐴
𝑡
+
1
)
−
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
)
.
Set 
𝑡
=
𝑡
+
1
 and repeat from step 3.

In each step of SARSA, we need to choose the next action according to the current policy.

Q-Learning: Off-policy TD control

The development of Q-learning (Watkins & Dayan, 1992) is a big breakout in the early days of Reinforcement Learning. Within one episode, it works as follows:

Initialize 
𝑡
=
0
.
Starts with 
𝑆
0
.
At time step 
𝑡
, we pick the action according to Q values, 
𝐴
𝑡
=
arg
⁡
max
𝑎
∈
𝐴
𝑄
(
𝑆
𝑡
,
𝑎
)
 and 
𝜖
-greedy is commonly applied.
After applying action 
𝐴
𝑡
, we observe reward 
𝑅
𝑡
+
1
 and get into the next state 
𝑆
𝑡
+
1
.
Update the Q-value function: 
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
←
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
+
𝛼
(
𝑅
𝑡
+
1
+
𝛾
max
𝑎
∈
𝐴
𝑄
(
𝑆
𝑡
+
1
,
𝑎
)
−
𝑄
(
𝑆
𝑡
,
𝐴
𝑡
)
)
.
𝑡
=
𝑡
+
1
 and repeat from step 3.

The key difference from SARSA is that Q-learning does not follow the current policy to pick the second action 
𝐴
𝑡
+
1
. It estimates 
𝑄
∗
 out of the best Q values, but which action (denoted as 
𝑎
∗
) leads to this maximal Q does not matter and in the next step Q-learning may not follow 
𝑎
∗
.

The backup diagrams for Q-learning and SARSA. (Image source: Replotted based on Figure 6.5 in Sutton & Barto (2017))
Deep Q-Network

Theoretically, we can memorize 
𝑄
∗
(
.
)
 for all state-action pairs in Q-learning, like in a gigantic table. However, it quickly becomes computationally infeasible when the state and action space are large. Thus people use functions (i.e. a machine learning model) to approximate Q values and this is called function approximation. For example, if we use a function with parameter 
𝜃
 to calculate Q values, we can label Q value function as 
𝑄
(
𝑠
,
𝑎
;
𝜃
)
.

Unfortunately Q-learning may suffer from instability and divergence when combined with an nonlinear Q-value function approximation and bootstrapping (See Problems #2).

Deep Q-Network (“DQN”; Mnih et al. 2015) aims to greatly improve and stabilize the training procedure of Q-learning by two innovative mechanisms:

Experience Replay: All the episode steps 
𝑒
𝑡
=
(
𝑆
𝑡
,
𝐴
𝑡
,
𝑅
𝑡
,
𝑆
𝑡
+
1
)
 are stored in one replay memory 
𝐷
𝑡
=
{
𝑒
1
,
…
,
𝑒
𝑡
}
. 
𝐷
𝑡
 has experience tuples over many episodes. During Q-learning updates, samples are drawn at random from the replay memory and thus one sample could be used multiple times. Experience replay improves data efficiency, removes correlations in the observation sequences, and smooths over changes in the data distribution.
Periodically Updated Target: Q is optimized towards target values that are only periodically updated. The Q network is cloned and kept frozen as the optimization target every C steps (C is a hyperparameter). This modification makes the training more stable as it overcomes the short-term oscillations.

The loss function looks like this:



𝐿
(
𝜃
)
=
𝐸
(
𝑠
,
𝑎
,
𝑟
,
𝑠
′
)
∼
𝑈
(
𝐷
)
[
(
𝑟
+
𝛾
max
𝑎
′
𝑄
(
𝑠
′
,
𝑎
′
;
𝜃
−
)
−
𝑄
(
𝑠
,
𝑎
;
𝜃
)
)
2
]

where 
𝑈
(
𝐷
)
 is a uniform distribution over the replay memory D; 
𝜃
−
 is the parameters of the frozen target Q-network.

In addition, it is also found to be helpful to clip the error term to be between [-1, 1]. (I always get mixed feeling with parameter clipping, as many studies have shown that it works empirically but it makes the math much less pretty. :/)

Algorithm for DQN with experience replay and occasionally frozen optimization target. The prepossessed sequence is the output of some processes running on the input images of Atari games. Don't worry too much about it; just consider them as input feature vectors. (Image source: Mnih et al. 2015)

There are many extensions of DQN to improve the original design, such as DQN with dueling architecture (Wang et al. 2016) which estimates state-value function V(s) and advantage function A(s, a) with shared network parameters.

Combining TD and MC Learning

In the previous section on value estimation in TD learning, we only trace one step further down the action chain when calculating the TD target. One can easily extend it to take multiple steps to estimate the return.

Let’s label the estimated return following n steps as 
𝐺
𝑡
(
𝑛
)
,
𝑛
=
1
,
…
,
∞
, then:

𝑛
	
𝐺
𝑡
	Notes

𝑛
=
1
	
𝐺
𝑡
(
1
)
=
𝑅
𝑡
+
1
+
𝛾
𝑉
(
𝑆
𝑡
+
1
)
	TD learning

𝑛
=
2
	
𝐺
𝑡
(
2
)
=
𝑅
𝑡
+
1
+
𝛾
𝑅
𝑡
+
2
+
𝛾
2
𝑉
(
𝑆
𝑡
+
2
)
	
…		

𝑛
=
𝑛
	
𝐺
𝑡
(
𝑛
)
=
𝑅
𝑡
+
1
+
𝛾
𝑅
𝑡
+
2
+
⋯
+
𝛾
𝑛
−
1
𝑅
𝑡
+
𝑛
+
𝛾
𝑛
𝑉
(
𝑆
𝑡
+
𝑛
)
	
…		

𝑛
=
∞
	
𝐺
𝑡
(
∞
)
=
𝑅
𝑡
+
1
+
𝛾
𝑅
𝑡
+
2
+
⋯
+
𝛾
𝑇
−
𝑡
−
1
𝑅
𝑇
+
𝛾
𝑇
−
𝑡
𝑉
(
𝑆
𝑇
)
	MC estimation

The generalized n-step TD learning still has the same form for updating the value function:

𝑉
(
𝑆
𝑡
)
←
𝑉
(
𝑆
𝑡
)
+
𝛼
(
𝐺
𝑡
(
𝑛
)
−
𝑉
(
𝑆
𝑡
)
)

We are free to pick any 
𝑛
 in TD learning as we like. Now the question becomes what is the best 
𝑛
? Which 
𝐺
𝑡
(
𝑛
)
 gives us the best return approximation? A common yet smart solution is to apply a weighted sum of all possible n-step TD targets rather than to pick a single best n. The weights decay by a factor λ with n, 
𝜆
𝑛
−
1
; the intuition is similar to why we want to discount future rewards when computing the return: the more future we look into the less confident we would be. To make all the weight (n → ∞) sum up to 1, we multiply every weight by (1-λ), because:

	
	
	
	
let 
𝑆
	
=
1
+
𝜆
+
𝜆
2
+
…


𝑆
	
=
1
+
𝜆
(
1
+
𝜆
+
𝜆
2
+
…
)


𝑆
	
=
1
+
𝜆
𝑆


𝑆
	
=
1
/
(
1
−
𝜆
)

This weighted sum of many n-step returns is called λ-return 
𝐺
𝑡
𝜆
=
(
1
−
𝜆
)
∑
𝑛
=
1
∞
𝜆
𝑛
−
1
𝐺
𝑡
(
𝑛
)
. TD learning that adopts λ-return for value updating is labeled as TD(λ). The original version we introduced above is equivalent to TD(0).

Comparison of the backup diagrams of Monte-Carlo, Temporal-Difference learning, and Dynamic Programming for state value functions. (Image source: David Silver's RL course lecture 4: "Model-Free Prediction")
Policy Gradient

All the methods we have introduced above aim to learn the state/action value function and then to select actions accordingly. Policy Gradient methods instead learn the policy directly with a parameterized function respect to 
𝜃
, 
𝜋
(
𝑎
|
𝑠
;
𝜃
)
. Let’s define the reward function (opposite of loss function) as the expected return and train the algorithm with the goal to maximize the reward function. My next post described why the policy gradient theorem works (proof) and introduced a number of policy gradient algorithms.

In discrete space:

𝐽
(
𝜃
)
=
𝑉
𝜋
𝜃
(
𝑆
1
)
=
𝐸
𝜋
𝜃
[
𝑉
1
]

where 
𝑆
1
 is the initial starting state.

Or in continuous space:





𝐽
(
𝜃
)
=
∑
𝑠
∈
𝑆
𝑑
𝜋
𝜃
(
𝑠
)
𝑉
𝜋
𝜃
(
𝑠
)
=
∑
𝑠
∈
𝑆
(
𝑑
𝜋
𝜃
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
,
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)
)

where 
𝑑
𝜋
𝜃
(
𝑠
)
 is stationary distribution of Markov chain for 
𝜋
𝜃
. If you are unfamiliar with the definition of a “stationary distribution,” please check this reference.

Using gradient ascent we can find the best θ that produces the highest return. It is natural to expect policy-based methods are more useful in continuous space, because there is an infinite number of actions and/or states to estimate the values for in continuous space and hence value-based approaches are computationally much more expensive.

Policy Gradient Theorem

Computing the gradient numerically can be done by perturbing θ by a small amount ε in the k-th dimension. It works even when 
𝐽
(
𝜃
)
 is not differentiable (nice!), but unsurprisingly very slow.

𝜕
𝐽
(
𝜃
)
𝜕
𝜃
𝑘
≈
𝐽
(
𝜃
+
𝜖
𝑢
𝑘
)
−
𝐽
(
𝜃
)
𝜖

Or analytically,




𝐽
(
𝜃
)
=
𝐸
𝜋
𝜃
[
𝑟
]
=
∑
𝑠
∈
𝑆
𝑑
𝜋
𝜃
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑅
(
𝑠
,
𝑎
)

Actually we have nice theoretical support for (replacing 
𝑑
(
.
)
 with 
𝑑
𝜋
(
.
)
):






𝐽
(
𝜃
)
=
∑
𝑠
∈
𝑆
𝑑
𝜋
𝜃
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)
∝
∑
𝑠
∈
𝑆
𝑑
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)

Check Sec 13.1 in Sutton & Barto (2017) for why this is the case.

Then,

	


	


	



	


	
𝐽
(
𝜃
)
	
=
∑
𝑠
∈
𝑆
𝑑
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)


∇
𝐽
(
𝜃
)
	
=
∑
𝑠
∈
𝑆
𝑑
(
𝑠
)
∑
𝑎
∈
𝐴
∇
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)

	
=
∑
𝑠
∈
𝑆
𝑑
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
∇
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)

	
=
∑
𝑠
∈
𝑆
𝑑
(
𝑠
)
∑
𝑎
∈
𝐴
𝜋
(
𝑎
|
𝑠
;
𝜃
)
∇
ln
⁡
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)

	
=
𝐸
𝜋
𝜃
[
∇
ln
⁡
𝜋
(
𝑎
|
𝑠
;
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)
]

This result is named “Policy Gradient Theorem” which lays the theoretical foundation for various policy gradient algorithms:

∇
𝐽
(
𝜃
)
=
𝐸
𝜋
𝜃
[
∇
ln
⁡
𝜋
(
𝑎
|
𝑠
,
𝜃
)
𝑄
𝜋
(
𝑠
,
𝑎
)
]
REINFORCE

REINFORCE, also known as Monte-Carlo policy gradient, relies on 
𝑄
𝜋
(
𝑠
,
𝑎
)
, an estimated return by MC methods using episode samples, to update the policy parameter 
𝜃
.

A commonly used variation of REINFORCE is to subtract a baseline value from the return 
𝐺
𝑡
 to reduce the variance of gradient estimation while keeping the bias unchanged. For example, a common baseline is state-value, and if applied, we would use 
𝐴
(
𝑠
,
𝑎
)
=
𝑄
(
𝑠
,
𝑎
)
−
𝑉
(
𝑠
)
 in the gradient ascent update.

Initialize θ at random
Generate one episode 
𝑆
1
,
𝐴
1
,
𝑅
2
,
𝑆
2
,
𝐴
2
,
…
,
𝑆
𝑇
For t=1, 2, … , T:
Estimate the the return G_t since the time step t.
𝜃
←
𝜃
+
𝛼
𝛾
𝑡
𝐺
𝑡
∇
ln
⁡
𝜋
(
𝐴
𝑡
|
𝑆
𝑡
,
𝜃
)
.
Actor-Critic

If the value function is learned in addition to the policy, we would get Actor-Critic algorithm.

Critic: updates value function parameters w and depending on the algorithm it could be action-value 
𝑄
(
𝑎
|
𝑠
;
𝑤
)
 or state-value 
𝑉
(
𝑠
;
𝑤
)
.
Actor: updates policy parameters θ, in the direction suggested by the critic, 
𝜋
(
𝑎
|
𝑠
;
𝜃
)
.

Let’s see how it works in an action-value actor-critic algorithm.

Initialize s, θ, w at random; sample 
𝑎
∼
𝜋
(
𝑎
|
𝑠
;
𝜃
)
.
For t = 1… T:
Sample reward 
𝑟
𝑡
∼
𝑅
(
𝑠
,
𝑎
)
 and next state 
𝑠
′
∼
𝑃
(
𝑠
′
|
𝑠
,
𝑎
)
.
Then sample the next action 
𝑎
′
∼
𝜋
(
𝑠
′
,
𝑎
′
;
𝜃
)
.
Update policy parameters: 
𝜃
←
𝜃
+
𝛼
𝜃
𝑄
(
𝑠
,
𝑎
;
𝑤
)
∇
𝜃
ln
⁡
𝜋
(
𝑎
|
𝑠
;
𝜃
)
.
Compute the correction for action-value at time t:

𝐺
𝑡
:
𝑡
+
1
=
𝑟
𝑡
+
𝛾
𝑄
(
𝑠
′
,
𝑎
′
;
𝑤
)
−
𝑄
(
𝑠
,
𝑎
;
𝑤
)

and use it to update value function parameters:

𝑤
←
𝑤
+
𝛼
𝑤
𝐺
𝑡
:
𝑡
+
1
∇
𝑤
𝑄
(
𝑠
,
𝑎
;
𝑤
)
.
Update 
𝑎
←
𝑎
′
 and 
𝑠
←
𝑠
′
.

𝛼
𝜃
 and 
𝛼
𝑤
 are two learning rates for policy and value function parameter updates, respectively.

A3C

Asynchronous Advantage Actor-Critic (Mnih et al., 2016), short for A3C, is a classic policy gradient method with the special focus on parallel training.

In A3C, the critics learn the state-value function, 
𝑉
(
𝑠
;
𝑤
)
, while multiple actors are trained in parallel and get synced with global parameters from time to time. Hence, A3C is good for parallel training by default, i.e. on one machine with multi-core CPU.

The loss function for state-value is to minimize the mean squared error, 
𝐽
𝑣
(
𝑤
)
=
(
𝐺
𝑡
−
𝑉
(
𝑠
;
𝑤
)
)
2
 and we use gradient descent to find the optimal w. This state-value function is used as the baseline in the policy gradient update.

Here is the algorithm outline:

We have global parameters, θ and w; similar thread-specific parameters, θ’ and w'.
Initialize the time step t = 1
While T <= T_MAX:
Reset gradient: dθ = 0 and dw = 0.
Synchronize thread-specific parameters with global ones: θ’ = θ and w’ = w.
𝑡
start
 = t and get 
𝑠
𝑡
.
While (
𝑠
𝑡
≠
TERMINAL
) and (
𝑡
−
𝑡
start
<=
𝑡
max
):
Pick the action 
𝑎
𝑡
∼
𝜋
(
𝑎
𝑡
|
𝑠
𝑡
;
𝜃
′
)
 and receive a new reward 
𝑟
𝑡
 and a new state 
𝑠
𝑡
+
1
.
Update t = t + 1 and T = T + 1.
Initialize the variable that holds the return estimation
		
𝑅
=
{
0
	
if 
𝑠
𝑡
 is TERMINAL
 
𝑉
(
𝑠
𝑡
;
𝑤
′
)
	
otherwise
.
For 
𝑖
=
𝑡
−
1
,
…
,
𝑡
start
:
𝑅
←
𝑟
𝑖
+
𝛾
𝑅
; here R is a MC measure of 
𝐺
𝑖
.
Accumulate gradients w.r.t. θ’: 
𝑑
𝜃
←
𝑑
𝜃
+
∇
𝜃
′
log
⁡
𝜋
(
𝑎
𝑖
|
𝑠
𝑖
;
𝜃
′
)
(
𝑅
−
𝑉
(
𝑠
𝑖
;
𝑤
′
)
)
;
Accumulate gradients w.r.t. w’: 
𝑑
𝑤
←
𝑑
𝑤
+
∇
𝑤
′
(
𝑅
−
𝑉
(
𝑠
𝑖
;
𝑤
′
)
)
2
.
Update synchronously θ using dθ, and w using dw.

A3C enables the parallelism in multiple agent training. The gradient accumulation step (6.2) can be considered as a reformation of minibatch-based stochastic gradient update: the values of w or θ get corrected by a little bit in the direction of each training thread independently.

Evolution Strategies

Evolution Strategies (ES) is a type of model-agnostic optimization approach. It learns the optimal solution by imitating Darwin’s theory of the evolution of species by natural selection. Two prerequisites for applying ES: (1) our solutions can freely interact with the environment and see whether they can solve the problem; (2) we are able to compute a fitness score of how good each solution is. We don’t have to know the environment configuration to solve the problem.

Say, we start with a population of random solutions. All of them are capable of interacting with the environment and only candidates with high fitness scores can survive (only the fittest can survive in a competition for limited resources). A new generation is then created by recombining the settings (gene mutation) of high-fitness survivors. This process is repeated until the new solutions are good enough.

Very different from the popular MDP-based approaches as what we have introduced above, ES aims to learn the policy parameter 
𝜃
 without value approximation. Let’s assume the distribution over the parameter 
𝜃
 is an isotropic multivariate Gaussian with mean 
𝜇
 and fixed covariance 
𝜎
2
𝐼
. The gradient of 
𝐹
(
𝜃
)
 is calculated:

			
			
	
		
			
			
	
		
	
		
	
		
	
∇
𝜃
𝐸
𝜃
∼
𝑁
(
𝜇
,
𝜎
2
)
𝐹
(
𝜃
)


=
	
∇
𝜃
∫
𝜃
𝐹
(
𝜃
)
Pr
(
𝜃
)
		
Pr(.) is the Gaussian density function.


=
	
∫
𝜃
𝐹
(
𝜃
)
Pr
(
𝜃
)
∇
𝜃
Pr
(
𝜃
)
Pr
(
𝜃
)


=
	
∫
𝜃
𝐹
(
𝜃
)
Pr
(
𝜃
)
∇
𝜃
log
⁡
Pr
(
𝜃
)


=
	
𝐸
𝜃
∼
𝑁
(
𝜇
,
𝜎
2
)
[
𝐹
(
𝜃
)
∇
𝜃
log
⁡
Pr
(
𝜃
)
]
		
Similar to how we do policy gradient update.


=
	
𝐸
𝜃
∼
𝑁
(
𝜇
,
𝜎
2
)
[
𝐹
(
𝜃
)
∇
𝜃
log
⁡
(
1
2
𝜋
𝜎
2
𝑒
−
(
𝜃
−
𝜇
)
2
2
𝜎
2
)
]


=
	
𝐸
𝜃
∼
𝑁
(
𝜇
,
𝜎
2
)
[
𝐹
(
𝜃
)
∇
𝜃
(
−
log
⁡
2
𝜋
𝜎
2
−
(
𝜃
−
𝜇
)
2
2
𝜎
2
)
]


=
	
𝐸
𝜃
∼
𝑁
(
𝜇
,
𝜎
2
)
[
𝐹
(
𝜃
)
𝜃
−
𝜇
𝜎
2
]

We can rewrite this formula in terms of a “mean” parameter 
𝜃
 (different from the 
𝜃
 above; this 
𝜃
 is the base gene for further mutation), 
𝜖
∼
𝑁
(
0
,
𝐼
)
 and therefore 
𝜃
+
𝜖
𝜎
∼
𝑁
(
𝜃
,
𝜎
2
)
. 
𝜖
 controls how much Gaussian noises should be added to create mutation:

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
+
𝜎
𝜖
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
𝐹
(
𝜃
+
𝜎
𝜖
)
𝜖
]
A simple parallel evolution-strategies-based RL algorithm. Parallel workers share the random seeds so that they can reconstruct the Gaussian noises with tiny communication bandwidth. (Image source: Salimans et al. 2017.)

ES, as a black-box optimization algorithm, is another approach to RL problems (In my original writing, I used the phrase “a nice alternative”; Seita pointed me to this discussion and thus I updated my wording.). It has a couple of good characteristics (Salimans et al., 2017) keeping it fast and easy to train:

ES does not need value function approximation;
ES does not perform gradient back-propagation;
ES is invariant to delayed or long-term rewards;
ES is highly parallelizable with very little data communication.
Known Problems
Exploration-Exploitation Dilemma

The problem of exploration vs exploitation dilemma has been discussed in my previous post. When the RL problem faces an unknown environment, this issue is especially a key to finding a good solution: without enough exploration, we cannot learn the environment well enough; without enough exploitation, we cannot complete our reward optimization task.

Different RL algorithms balance between exploration and exploitation in different ways. In MC methods, Q-learning or many on-policy algorithms, the exploration is commonly implemented by ε-greedy; In ES, the exploration is captured by the policy parameter perturbation. Please keep this into consideration when developing a new RL algorithm.

Deadly Triad Issue

We do seek the efficiency and flexibility of TD methods that involve bootstrapping. However, when off-policy, nonlinear function approximation, and bootstrapping are combined in one RL algorithm, the training could be unstable and hard to converge. This issue is known as the deadly triad (Sutton & Barto, 2017). Many architectures using deep learning models were proposed to resolve the problem, including DQN to stabilize the training with experience replay and occasionally frozen target network.

Case Study: AlphaGo Zero

The game of Go has been an extremely hard problem in the field of Artificial Intelligence for decades until recent years. AlphaGo and AlphaGo Zero are two programs developed by a team at DeepMind. Both involve deep Convolutional Neural Networks (CNN) and Monte Carlo Tree Search (MCTS) and both have been approved to achieve the level of professional human Go players. Different from AlphaGo that relied on supervised learning from expert human moves, AlphaGo Zero used only reinforcement learning and self-play without human knowledge beyond the basic rules.

The board of Go. Two players play black and white stones alternatively on the vacant intersections of a board with 19 x 19 lines. A group of stones must have at least one open point (an intersection, called a "liberty") to remain on the board and must have at least two or more enclosed liberties (called "eyes") to stay "alive". No stone shall repeat a previous position.

With all the knowledge of RL above, let’s take a look at how AlphaGo Zero works. The main component is a deep CNN over the game board configuration (precisely, a ResNet with batch normalization and ReLU). This network outputs two values:

(
𝑝
,
𝑣
)
=
𝑓
𝜃
(
𝑠
)
𝑠
: the game board configuration, 19 x 19 x 17 stacked feature planes; 17 features for each position, 8 past configurations (including current) for the current player + 8 past configurations for the opponent + 1 feature indicating the color (1=black, 0=white). We need to code the color specifically because the network is playing with itself and the colors of current player and opponents are switching between steps.
𝑝
: the probability of selecting a move over 19^2 + 1 candidates (19^2 positions on the board, in addition to passing).
𝑣
: the winning probability given the current setting.

During self-play, MCTS further improves the action probability distribution 
𝜋
∼
𝑝
(
.
)
 and then the action 
𝑎
𝑡
 is sampled from this improved policy. The reward 
𝑧
𝑡
 is a binary value indicating whether the current player eventually wins the game. Each move generates an episode tuple 
(
𝑠
𝑡
,
𝜋
𝑡
,
𝑧
𝑡
)
 and it is saved into the replay memory. The details on MCTS are skipped for the sake of space in this post; please read the original paper if you are interested.

AlphaGo Zero is trained by self-play while MCTS improves the output policy further in every step. (Image source: Figure 1a in Silver et al., 2017).

The network is trained with the samples in the replay memory to minimize the loss:

𝐿
=
(
𝑧
−
𝑣
)
2
−
𝜋
⊤
log
⁡
𝑝
+
𝑐
‖
𝜃
‖
2

where 
𝑐
 is a hyperparameter controlling the intensity of L2 penalty to avoid overfitting.

AlphaGo Zero simplified AlphaGo by removing supervised learning and merging separated policy and value networks into one. It turns out that AlphaGo Zero achieved largely improved performance with a much shorter training time! I strongly recommend reading these two papers side by side and compare the difference, super fun.

I know this is a long read, but hopefully worth it. If you notice mistakes and errors in this post, don’t hesitate to contact me at [lilian dot wengweng at gmail dot com]. See you in the next post! :)

Cited as:

@article{weng2018bandit,
  title   = "A (Long) Peek into Reinforcement Learning",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2018",
  url     = "https://lilianweng.github.io/posts/2018-02-19-rl-overview/"
}

References

[1] Yuxi Li. Deep reinforcement learning: An overview. arXiv preprint arXiv:1701.07274. 2017.

[2] Richard S. Sutton and Andrew G. Barto. Reinforcement Learning: An Introduction; 2nd Edition. 2017.

[3] Volodymyr Mnih, et al. Asynchronous methods for deep reinforcement learning. ICML. 2016.

[4] Tim Salimans, et al. Evolution strategies as a scalable alternative to reinforcement learning. arXiv preprint arXiv:1703.03864 (2017).

[5] David Silver, et al. Mastering the game of go without human knowledge. Nature 550.7676 (2017): 354.

[6] David Silver, et al. Mastering the game of Go with deep neural networks and tree search. Nature 529.7587 (2016): 484-489.

[7] Volodymyr Mnih, et al. Human-level control through deep reinforcement learning. Nature 518.7540 (2015): 529.

[8] Ziyu Wang, et al. Dueling network architectures for deep reinforcement learning. ICML. 2016.

[9] Reinforcement Learning lectures by David Silver on YouTube.

[10] OpenAI Blog: Evolution Strategies as a Scalable Alternative to Reinforcement Learning

[11] Frank Sehnke, et al. Parameter-exploring policy gradients. Neural Networks 23.4 (2010): 551-559.

[12] Csaba Szepesvári. Algorithms for reinforcement learning. 1st Edition. Synthesis lectures on artificial intelligence and machine learning 4.1 (2010): 1-103.

If you notice mistakes and errors in this post, please don’t hesitate to contact me at [lilian dot wengweng at gmail dot com] and I would be super happy to correct them right away!

Reinforcement-Learning
 
Long-Read
 
Math-Heavy
«
Policy Gradient Algorithms
»
The Multi-Armed Bandit Problem and Its Solutions
© 2026 Lil'Log Powered by Hugo & PaperMod