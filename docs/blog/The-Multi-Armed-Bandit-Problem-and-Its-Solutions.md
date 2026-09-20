---
title: The Multi-Armed Bandit Problem and Its Solutions
url: https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:48:06.527669+00:00'
---

The algorithms are implemented for Bernoulli bandit in [lilianweng/multi-armed-bandit](http://github.com/lilianweng/multi-armed-bandit).

## Exploitation vs Exploration

The exploration vs exploitation dilemma exists in many aspects of our life. Say, your favorite restaurant is right around the corner. If you go there every day, you would be confident of what you will get, but miss the chances of discovering an even better option. If you try new places all the time, very likely you are gonna have to eat unpleasant food from time to time. Similarly, online advisors try to balance between the known most attractive ads and the new ads that might be even more successful.

![Image 1](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/exploration_vs_exploitation.png)

A real-life example of the exploration vs exploitation dilemma: where to eat? (Image source: UC Berkeley AI course [slide](http://ai.berkeley.edu/lecture_slides.html), [lecture 11](http://ai.berkeley.edu/slides/Lecture%2011%20--%20Reinforcement%20Learning%20II/SP14%20CS188%20Lecture%2011%20--%20Reinforcement%20Learning%20II.pptx).)

If we have learned all the information about the environment, we are able to find the best strategy by even just simulating brute-force, let alone many other smart approaches. The dilemma comes from the _incomplete_ information: we need to gather enough information to make best overall decisions while keeping the risk under control. With exploitation, we take advantage of the best option we know. With exploration, we take some risk to collect information about unknown options. The best long-term strategy may involve short-term sacrifices. For example, one exploration trial could be a total failure, but it warns us of not taking that action too often in the future.

## What is Multi-Armed Bandit?

The [multi-armed bandit](https://en.wikipedia.org/wiki/Multi-armed_bandit) problem is a classic problem that well demonstrates the exploration vs exploitation dilemma. Imagine you are in a casino facing multiple slot machines and each is configured with an unknown probability of how likely you can get a reward at one play. The question is: _What is the best strategy to achieve highest long-term rewards?_

In this post, we will only discuss the setting of having an infinite number of trials. The restriction on a finite number of trials introduces a new type of exploration problem. For instance, if the number of trials is smaller than the number of slot machines, we cannot even try every machine to estimate the reward probability (!) and hence we have to behave smartly w.r.t. a limited set of knowledge and resources (i.e. time).

![Image 2](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/bern_bandit.png)

An illustration of how a Bernoulli multi-armed bandit works. The reward probabilities are **unknown** to the player.

A naive approach can be that you continue to playing with one machine for many many rounds so as to eventually estimate the “true” reward probability according to the [law of large numbers](https://en.wikipedia.org/wiki/Law_of_large_numbers). However, this is quite wasteful and surely does not guarantee the best long-term reward.

## Definition

Now let’s give it a scientific definition.

A Bernoulli multi-armed bandit can be described as a tuple of , where:

*   We have machines with reward probabilities, .
*   At each time step t, we take an action a on one slot machine and receive a reward r.
*    is a set of actions, each referring to the interaction with one slot machine. The value of action a is the expected reward, . If action at the time step t is on the i-th machine, then .
*    is a reward function. In the case of Bernoulli bandit, we observe a reward r in a _stochastic_ fashion. At the time step t, may return reward 1 with a probability or 0 otherwise.

It is a simplified version of [Markov decision process](https://en.wikipedia.org/wiki/Markov_decision_process), as there is no state .

The goal is to maximize the cumulative reward . If we know the optimal action with the best reward, then the goal is same as to minimize the potential [regret](https://en.wikipedia.org/wiki/Regret_(decision_theory)) or loss by not picking the optimal action.

The optimal reward probability of the optimal action is:

Our loss function is the total regret we might have by not selecting the optimal action up to the time step T:

## Bandit Strategies

Based on how we do exploration, there several ways to solve the multi-armed bandit.

*   No exploration: the most naive approach and a bad one.
*   Exploration at random
*   Exploration smartly with preference to uncertainty

## ε-Greedy Algorithm

The ε-greedy algorithm takes the best action most of the time, but does random exploration occasionally. The action value is estimated according to the past experience by averaging the rewards associated with the target action a that we have observed so far (up to the current time step t):

where is a binary indicator function and is how many times the action a has been selected so far, .

According to the ε-greedy algorithm, with a small probability we take a random action, but otherwise (which should be the most of the time, probability 1-) we pick the best action that we have learnt so far: .

Check my toy implementation [here](https://github.com/lilianweng/multi-armed-bandit/blob/master/solvers.py#L45).

## Upper Confidence Bounds

Random exploration gives us an opportunity to try out options that we have not known much about. However, due to the randomness, it is possible we end up exploring a bad action which we have confirmed in the past (bad luck!). To avoid such inefficient exploration, one approach is to decrease the parameter ε in time and the other is to be optimistic about options with _high uncertainty_ and thus to prefer actions for which we haven’t had a confident value estimation yet. Or in other words, we favor exploration of actions with a strong potential to have a optimal value.

The Upper Confidence Bounds (UCB) algorithm measures this potential by an upper confidence bound of the reward value, , so that the true value is below with bound with high probability. The upper bound is a function of ; a larger number of trials should give us a smaller bound .

In UCB algorithm, we always select the greediest action to maximize the upper confidence bound:

Now, the question is _how to estimate the upper confidence bound_.

## Hoeffding’s Inequality

If we do not want to assign any prior knowledge on how the distribution looks like, we can get help from [“Hoeffding’s Inequality”](http://cs229.stanford.edu/extra-notes/hoeffding.pdf) — a theorem applicable to any bounded distribution.

Let be i.i.d. (independent and identically distributed) random variables and they are all bounded by the interval [0, 1]. The sample mean is . Then for , we have:

Given one target action , let us consider:

*    as the random variables,
*    as the true mean,
*    as the sample mean,
*   And as the upper confidence bound, 

Then we have,

We want to pick a bound so that with high chances the true mean is blow the sample mean + the upper confidence bound. Thus should be a small probability. Let’s say we are ok with a tiny threshold p:

## UCB1

One heuristic is to reduce the threshold p in time, as we want to make more confident bound estimation with more rewards observed. Set we get **UCB1** algorithm:

## Bayesian UCB

In UCB or UCB1 algorithm, we do not assume any prior on the reward distribution and therefore we have to rely on the Hoeffding’s Inequality for a very generalize estimation. If we are able to know the distribution upfront, we would be able to make better bound estimation.

For example, if we expect the mean reward of every slot machine to be Gaussian as in Fig 2, we can set the upper bound as 95% confidence interval by setting to be twice the standard deviation.

![Image 3](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/bern_UCB.png)

When the expected reward has a Gaussian distribution. is the standard deviation and is the upper confidence bound. The constant is a adjustable hyperparameter. (Image source: [UCL RL course lecture 9's slides](http://www0.cs.ucl.ac.uk/staff/d.silver/web/Teaching_files/XX.pdf))

Check my toy implementation of [UCB1](https://github.com/lilianweng/multi-armed-bandit/blob/master/solvers.py#L76) and [Bayesian UCB](https://github.com/lilianweng/multi-armed-bandit/blob/master/solvers.py#L99) with Beta prior on θ.

## Thompson Sampling

Thompson sampling has a simple idea but it works great for solving the multi-armed bandit problem.

![Image 4](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/klay-thompson.jpg)

Oops, I guess not this Thompson? (Credit goes to [Ben Taborsky](https://www.linkedin.com/in/benjamin-taborsky); he has a full theorem of how Thompson invented while pondering over who to pass the ball. Yes I stole his joke.)

At each time step, we want to select action a according to the probability that a is **optimal**:

where is the probability of taking action a given the history .

For the Bernoulli bandit, it is natural to assume that follows a [Beta](https://en.wikipedia.org/wiki/Beta_distribution) distribution, as is essentially the success probability θ in [Bernoulli](https://en.wikipedia.org/wiki/Bernoulli_distribution) distribution. The value of is within the interval [0, 1]; α and β correspond to the counts when we **succeeded** or **failed** to get a reward respectively.

First, let us initialize the Beta parameters α and β based on some prior knowledge or belief for every action. For example,

*   α = 1 and β = 1; we expect the reward probability to be 50% but we are not very confident.
*   α = 1000 and β = 9000; we strongly believe that the reward probability is 10%.

At each time t, we sample an expected reward, , from the prior distribution for every action. The best action is selected among samples: . After the true reward is observed, we can update the Beta distribution accordingly, which is essentially doing Bayesian inference to compute the posterior with the known prior and the likelihood of getting the sampled data.

Thompson sampling implements the idea of [probability matching](https://en.wikipedia.org/wiki/Probability_matching). Because its reward estimations are sampled from posterior distributions, each of these probabilities is equivalent to the probability that the corresponding action is optimal, conditioned on observed history.

However, for many practical and complex problems, it can be computationally intractable to estimate the posterior distributions with observed true rewards using Bayesian inference. Thompson sampling still can work out if we are able to approximate the posterior distributions using methods like Gibbs sampling, Laplace approximate, and the bootstraps. This [tutorial](https://arxiv.org/pdf/1707.02038.pdf) presents a comprehensive review; strongly recommend it if you want to learn more about Thompson sampling.

## Case Study

I implemented the above algorithms in [lilianweng/multi-armed-bandit](https://github.com/lilianweng/multi-armed-bandit). A [BernoulliBandit](https://github.com/lilianweng/multi-armed-bandit/blob/master/bandits.py#L13) object can be constructed with a list of random or predefined reward probabilities. The bandit algorithms are implemented as subclasses of [Solver](https://github.com/lilianweng/multi-armed-bandit/blob/master/solvers.py#L9), taking a Bandit object as the target problem. The cumulative regrets are tracked in time.

![Image 5](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/bandit_experiment.png)

The result of a small experiment on solving a Bernoulli bandit with K = 10 slot machines with reward probabilities, {0.0, 0.1, 0.2, ..., 0.9}. Each solver runs 10000 steps.

(Left) The plot of time step vs the cumulative regrets. (Middle) The plot of true reward probability vs estimated probability. (Right) The fraction of each action is picked during the 10000-step run.*

## Summary

We need exploration because information is valuable. In terms of the exploration strategies, we can do no exploration at all, focusing on the short-term returns. Or we occasionally explore at random. Or even further, we explore and we are picky about which options to explore — actions with higher uncertainty are favored because they can provide higher information gain.

![Image 6](https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/bandit_solution_summary.png)

* * *

Cited as:

```
@article{weng2018bandit,
  title   = "The Multi-Armed Bandit Problem and Its Solutions",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2018",
  url     = "https://lilianweng.github.io/posts/2018-01-23-multi-armed-bandit/"
}
```

## References

[1] CS229 Supplemental Lecture notes: [Hoeffding’s inequality](http://cs229.stanford.edu/extra-notes/hoeffding.pdf).

[2] RL Course by David Silver - Lecture 9: [Exploration and Exploitation](https://youtu.be/sGuiWX07sKw)

[3] Olivier Chapelle and Lihong Li. [“An empirical evaluation of thompson sampling.”](http://papers.nips.cc/paper/4321-an-empirical-evaluation-of-thompson-sampling.pdf) NIPS. 2011.

[4] Russo, Daniel, et al. [“A Tutorial on Thompson Sampling.”](https://arxiv.org/pdf/1707.02038.pdf) arXiv:1707.02038 (2017).