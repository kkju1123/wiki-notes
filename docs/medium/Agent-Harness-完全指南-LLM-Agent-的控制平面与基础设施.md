---
title: Agent Harness 完全指南：LLM Agent 的控制平面与基础设施
url: https://medium.com/gitconnected/agent-harness-a-no-bs-guide-8f69e3c0a3da
source_type: medium
author: S Sankar
tags:
- AI Agent
- Agent Harness
- LLM
- Orchestration
- Observability
summary: Agent Harness 是控制、监控与评估 AI Agent 的基础设施层，用于防止长任务失控并保障安全。
fetched_at: '2026-09-17T08:49:28.193472+00:00'
---

Member-only story

[AI Agent](https://medium.com/tag/ai-agent?source=post_page---header_tags--8f69e3c0a3da-----------------------------------------)

[Artificial Intelligence](https://medium.com/tag/artificial-intelligence?source=post_page---header_tags--8f69e3c0a3da-----------------------------------------)

[AI](https://medium.com/tag/ai?source=post_page---header_tags--8f69e3c0a3da-----------------------------------------)

[LLM](https://medium.com/tag/llm?source=post_page---header_tags--8f69e3c0a3da-----------------------------------------)

[Large Language Models](https://medium.com/tag/large-language-models?source=post_page---header_tags--8f69e3c0a3da-----------------------------------------)

# Agent Harness — A complete guide

[

![S Sankar](https://miro.medium.com/v2/resize:fill:64:64/1*_EWL2lSb84Qu1dU07IPzDQ.jpeg)


](https://medium.com/@AIBites?source=post_page---byline--8f69e3c0a3da-----------------------------------------)

[S Sankar](https://medium.com/@AIBites?source=post_page---byline--8f69e3c0a3da-----------------------------------------)

9 min read3 days ago

_There isn’t much more to understanding Agent Harness than what's in here!_

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*RD27u6X0ynG5tWq4cXvSkQ.jpeg)

When we started interacting with LLMs back in 2023, life was simple. We simply asked a question and got an answer back. A typical example of a question from a programmer like me would be, “Write a function in Python to sort an array of numbers”. We get the answer in near real-time without the model going into _thinking_ mode! Because the models were bare-bones LLMs without all the bells and whistles of today’s agentic systems.

Non-members can read for free [here](https://medium.com/@AIBites/agent-harness-a-no-bs-guide-8f69e3c0a3da?sk=70748051e7a19aa3d94e4d5499e26a8d)!

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*pSapMrZeImZbNL-ez94UlQ.png)

A bare-bones example of how we got started with LLMs back in 2023.

By bells and whistles, I am referring to tools, memory, skills, and many more that have become inevitable in 2026. All these add-ons are what make a simple LLM an AI agent.

### The Agentic race on the one side

Back in 2023 and 2024, the race was about the models. OpenAI had one of the best general-purpose models, and Anthropic had one of the best coding models. Competitors constantly built models to beat other models in standard benchmarks like MMLU, SWE-bench, etc.

The model race started to saturate, and things like reasoning and context started appearing. Things like “_thinking…_” started to appear in chat UIs when we ask something. People got curious as to what's going on under the hood when “_thinking…_” appears in the UI.

The agent race begins…

### User’s ambition on the other side

As the models got more sophisticated, we as users also got ambitious. Instead of asking questions, we started assigning tasks. The tasks needed the agents to set goals, use external tools like the browser, recall past conversations and facts from memory, and much more to answer the user.

I still vividly remember how I took a screenshot of the Amazon landing page and gave it as input to a coding model, asked it to build a site to mimic the screenshot (all in one shot)! It did come up with a similar UI, but think of a user who actually wanted the UI, backend, infrastructure, and everything around it to be set up in one shot.

This is where we are in 2026!

So these days I prompt, “Solve the bug raised in issue #122 on GitHub”. A prompt like this demands much more action from the LLM. There are several steps tucked inside this simple prompt. An LLM in 2023 would have said, “Sorry, I don’t have access to your GitHub repo”.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*1--Ittdv7ZjFlXZrZ6_yAg.png)

As users get ambitious, the prompts are turning into tasks. We need sophisticated agents to tackle the user’s demands.

But an LLM with a good Agent Harness, on the other hand, enters _thinking_ mode. It goes through reasoning → planning → choose tool/action → execute tool → observe result → repeat or stop.

### I know, but what on earth is Agent Harness?

If we have so many add-ons like context, tools, memory, etc. wrapped around an LLM, there should be an efficient way to **_engineer_** how these things interact with the LLM. And Agent Harness is the term given to it. Formally,

> An agent harness is the infrastructure layer that wraps, controls, tests, monitors, and evaluates an AI agent.

In simple terms, it is the “operating system” around the LLM.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*_5Owg_5HSfXeMLx0fNv6Mg.png)

As the infrastructure layer that wraps around the LLM, the harness consists of the agent architecture, the orchestration framework to orchestrate the multiple sub-agents within the harness, the evaluation system that evaluates the agent’s output, the runtime infrastructure, security, governance, and last but not least, observability.

### Why do we need an agent harness?

As we set bigger and bigger goals for AI agents, the time it takes for the AI agents to execute these tasks is increasing. This is evident from the interesting plot below from Anthropic.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*zYm5dRPSKHzAsBnXfAN3zA.png)

Data from Anthropic reveals that the agent execution times are growing exponentially!

Just a year back in 2025, the agents used to take around an hour to execute a given task. But these days it's hovering around the 12-hour mark! These long-horizon tasks come with some problems. They may loop forever, hallucinate tools, corrupt the agent state, exceed token budgets, lose context, retry bad actions, fail silently, misuse permissions
or even forget objectives.

Now, these are expensive problems. Imagine an AI agent went into an infinite loop for 12 hours. The token usage and the compute used by the loop would be enormous. We can set a budget in the harness for any given task to overcome this problem! The budget can be a time, token, or compute budget.

Even worse are the security implications. Imagine the LLM chose to use a tool that is disruptive, like a tool that deletes all the files in the file system. Through the harness, we can have fine-grained control over what the LLM has access to!

In short, it's the harness that gives you control over the agent and hence gives you confidence that the agent is doing what you want it to do!

## Components of agent harness

Whatever we call a harness, it should fall into one of the components below.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*Fa_69lDTTu4N3t7gaB3R3g.png)

-   **Agent state.** Answers the question: _Where am I right now?_ It keeps track of the current task, step, observations, decisions, etc. For example, if there are 4 steps in the agent’s plan to accomplish a goal, the state keeps track of what is complete, the current state, and what remains to be done.
-   **Tools.** Answers the question: _What can I do?_ Gives the agent capabilities like code execution, browser, APIs, databases, shell, etc. The tools are the most obvious of all the components. For example, for a coding agent, a _read\_file_ function could be one of the fundamental tools.
-   **Planning.** Answers the question: _What should I do?_ Decides _what should happen next_ and creates/revises a plan. In a coding task, the planning component can come up with a sequence of 4–5 steps that need to be accomplished in order to complete the task.
-   **Memory.** Answers the question: _What do I know/remember?_ Stores and retrieves information across steps or sessions. Whenever we retrieve some information from a Vector DB into context, it might be worth summarizing the contents to optimize the context and hence the working memory. These things need to be managed by the memory component of the harness.
-   **Orchestration.** Answers the question: _How do I execute this process?_Controls the execution loop, tool calls, retries, sub-agents, parallelism, handoffs, etc. For instance, lets say the agent is dealing with a purchase task. A payment sub-agent can be invoked, and the execution of the rest of the operation, such as updating the inventory, can be executed in parallel by an inventory agent. Or, the inventory agent should only execute in sequence after the payment agent has completed its task successfully.
-   **Context Management.** Answers the question: _What should the LLM see right now?_ Decides what information gets placed into the LLM’s context window at each step.
-   **Safety & Governance.** Answers the question: _What am I allowed to do?_ Permissions, sandboxing, approval gates, policies, limits, authentication, etc. are all covered under this component. Even if the model has a killer tool at its disposal, to what extent can it go? For example, even if the model is allowed to edit files in a coding task, it is safe not to allow it to delete all or some of the files in the file system.
-   **Observability & Evaluation.** Answers the question: Did it actually work?Traces what happened and measures whether the agent succeeded, failed, or violated constraints. Things like what it took for the model to complete the given task get tracked in observability. For example, the model may need 3 tools out of 10 to complete a task. It's worth tracking how these tools get used. If they fail, is it because the model is missing some tools in its repertoire, or is it because some data is missing from the context?

And _harness engineering_ is the new discipline of how we design, implement, and execute the above components so that the agent is at its best.

### A practical coding agent example

Now that we know the different components, lets see how these components interact with each other for a simple coding task.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*eWs9wDTGoY_0GFNQx96bGg.png)

Lets say we have implemented the harness for a coding agent. The user sends the prompt:

> Solve the bug in issue #122 in github

Note that this is not a simple, “explain to me… “ kind of prompt. This is a genuine task. To complete this task, we need agentic capabilities like planning, tool use, memory, and much more.

Below is one logical way in which the harness can be implemented:

-   To start with, the user’s input goes into the context. The model starts to reason and comes up with a goal. The goal remains in the context till its accomplished.
-   It then decides to “plan” the execution of the task. It comes up with this plan, which is: check any related bugs, fix the right file, compile and test, run test cases, and update memory.
-   As the first step of execution, the harness retrieves some memory from the Vector DB, which is the details of past bugs relevant to the task at hand. It finds authentication-related information as the closest match. So it puts it in the context.
-   The model then uses the info in the context to decide to exit the auth.py file to fix the authentication bug. This, however, needs the _read\_file_ and _write\_file_ tools. The model uses these tools, edits the file, and saves it.
-   The model decides to test the code change through the terminal. Now the terminal tool needs to be used, which is available in the set of tools at the model’s disposal. (The harness ensures that the model has the right level of access to the terminal and files)
-   The model moves on to the next step in the plan and executes test cases. It will need access to further tools for this. Also, note that every step of the way updates the context to ensure the model keeps track of where it is and what remains to be done.
-   All along the way, the observability component ensures that everything is logged. The logging includes the tools used, the time it takes for the task, memory used, any failures in the middle, etc.
-   Note that we haven’t touched upon evaluation, which is a beast on its own. It is not just about accuracy and can cover several metrics in line with the observed behaviour of the agent.

## So where are we heading?

One of the largest challenges in agentic AI today is long-horizon reliability. As running agents for hours, if not days is becomes a reality, we need to ensure that the run is absolutely reliable. If the agent is stuck in an infinite loop or some corrupt state, we may end up wasting resources and compromising safety.

Next, evaluation gets quite challenging as we build more sophisticated harnesses around the models. Every task is going to leave traces with tons of data, like the memory used; should I have to swap the model or engineer the harness? Should I have to add more tools or fix existing ones?

These are questions that are popping up as the community actively works on building sophisticated harnesses.

Lets wait and watch where we head next. See you in my next…

Lastly, if you liked this article, why not follow us on [X](https://twitter.com/ai_bites), where we share AI news and research updates — daily.

Check out and subscribe to our [YouTube channel](https://www.youtube.com/c/aibites), where we explain AI concepts and papers visually.

Do not forget to clap, and let’s celebrate you reaching the end of this story.