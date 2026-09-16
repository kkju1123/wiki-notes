---
title: 5 Papers Every Agentic AI Engineer Should Know
url: https://medium.com/@lekha-bhan88/5-papers-every-agentic-ai-engineer-should-know-edf64a6118d4
source_type: medium
author: Lekha Priya
tags: []
summary: ''
fetched_at: '2026-09-16T11:13:32.388950+00:00'
---

Member-only story

[Reasearch Papers](https://medium.com/tag/reasearch-paper?source=post_page---header_tags--edf64a6118d4-----------------------------------------)

[Ai Engineering](https://medium.com/tag/ai-engineering?source=post_page---header_tags--edf64a6118d4-----------------------------------------)

[Generative Ai Tools](https://medium.com/tag/generative-ai-tools?source=post_page---header_tags--edf64a6118d4-----------------------------------------)

# 5 Papers Every Agentic AI Engineer Should Know

## The papers behind the reasoning, tools, memory, reflection, and multi-agent systems we build today.

[

![Lekha Priya](https://miro.medium.com/v2/resize:fill:64:64/1*kWZRIfGVrAD2fV2JAxXEmQ.jpeg)


](/?source=post_page---byline--edf64a6118d4-----------------------------------------)

[Lekha Priya](/?source=post_page---byline--edf64a6118d4-----------------------------------------)

11 min read5 days ago

**Agentic AI is moving fast.**

Every week, there is a new framework, a new agent pattern, a new orchestration library, or a new model claiming to make agents smarter.

But here’s the part that often gets lost in the noise:

**Many of today’s “new” agentic patterns aren’t actually new.**

The ideas behind reasoning-and-acting loops, tool use, agent memory, self-reflection, planning, and multi-agent collaboration have been explored in research for years.

If you’re an **AI engineer, ML engineer, GenAI developer, AI architect, researcher, or someone transitioning from LLM applications into Agentic AI**, understanding these foundations can change the way you approach building agents.

You don’t need to read hundreds of papers.

Start with these five.

They represent some of the key research directions that shaped how we think about modern AI agents — and you’ll start recognizing their influence everywhere, from **LangGraph and CrewAI to AutoGen, MCP, tool-calling systems, and production agent architectures.**

Because frameworks come and go.

**Architectural patterns evolve.**

But the research ideas underneath them are what you want to understand.

And once you see those connections, you stop learning Agentic AI as a collection of frameworks and start understanding it as a **set of engineering principles**.

> **_Don’t just learn how to build an agent. Learn why agents are built the way they are._**

## 📖 A note for free readers

If you’re enjoying this article but don’t have access to the full publication, **you can use my friend’s link to read this edition for free.**[https://lekha-bhan88.medium.com/5-papers-every-agentic-ai-engineer-should-know-edf64a6118d4?sk=bab581c0d2e23c48f1923ffaf5313085](/5-papers-every-agentic-ai-engineer-should-know-edf64a6118d4?sk=bab581c0d2e23c48f1923ffaf5313085)

Now, let’s go back to the research that started shaping the agentic systems we’re building today.

## 1\. ReAct — Reasoning + Acting

### ReAct: Synergizing Reasoning and Acting in Language Models

The first paper I would recommend to anyone getting serious about agentic AI is **ReAct**.

The core idea is beautifully simple:

Think → Act → Observe → Think → Act → Observe

Instead of asking an LLM to reason entirely inside a single response, ReAct interleaves **reasoning with actions**.

The agent can:

-   formulate a plan
-   take an action
-   observe the result
-   update its reasoning
-   change course

That last part is critical.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*iMyWeSOjVlDL-BY9D5rHSw.png)

The paper demonstrated this approach using external sources such as Wikipedia and showed improvements on both reasoning and interactive decision-making tasks.

[https://arxiv.org/pdf/2210.03629](https://arxiv.org/pdf/2210.03629)

### Why it matters today?

This pattern is essentially the ancestor of today’s:

**Agent → Tool → Observation → Agent**

loop.

When you build a LangGraph agent that decides whether to search, retrieves information, evaluates the result and then decides what to do next, you’re implementing a modern version of this fundamental idea.

The important conceptual shift is:

> **_Reasoning doesn’t have to happen before action. Reasoning and action can inform each other._**

And that idea sits underneath a huge portion of modern agent engineering.

## 2\. Toolformer — Teaching Models When to Use Tools

### Toolformer: Language Models Can Teach Themselves to Use Tools

ReAct established one of the fundamental patterns behind modern agentic systems: an LLM can reason about a task, take an action, observe the outcome, and then continue reasoning based on what happened. But once we introduce actions into the workflow, another problem immediately appears. **How does the model know when it should actually use a tool?**

Consider something as simple as calculating **17 × 238**. An LLM can generate the answer using its learned knowledge and reasoning capabilities, but a calculator is fundamentally better suited for this task. The interesting capability, therefore, isn’t whether the model _can_ perform the calculation. It is whether the model can recognize that an external tool would produce a more reliable result and decide to use it.

This is the problem explored by **Toolformer**, the paper titled _“Toolformer: Language Models Can Teach Themselves to Use Tools.”_ The research investigates whether language models can learn to determine when external tools are useful and how those tools should be incorporated into the model’s reasoning process.

The model has to make several decisions during this process. It needs to recognize whether the current problem requires an external capability, determine which tool is appropriate, construct the input required by that tool, execute the call, and then incorporate the returned information into its subsequent reasoning. In other words, tool use becomes part of the model’s decision-making process rather than simply being a fixed function manually triggered by a developer.

The paper experimented with several types of tools, including calculators, search, translation and calendar-related capabilities. The broader idea was significant because it demonstrated that an LLM doesn’t have to solve every problem using only the knowledge encoded within its parameters. Instead, it can learn to extend its capabilities by interacting with external systems.

This idea is now deeply embedded in modern LLM application architecture. When an agent decides to query a database, invoke an API, perform a web search, execute Python code, retrieve information through MCP, or interact with an enterprise application, it is operating on the same fundamental principle: **the model provides reasoning and coordination while external tools provide specialized capabilities.**

Think about an enterprise agent that receives the request, _“Find the top five customers by revenue this quarter and prepare a summary for the leadership team.”_ The LLM shouldn’t attempt to invent the numbers from its own knowledge. It should understand that the information exists somewhere in an enterprise system, identify the appropriate data source, construct the query, retrieve the results, analyze them, and then produce the requested summary.

The architecture therefore becomes more interesting than a simple prompt-and-response interaction:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*9y9L8AWDLKqG_tK71Pc_kw.png)

The important architectural principle here is **delegation**. An LLM doesn’t need to be the best system at every individual operation. A calculator is better at arithmetic, a database is better at querying structured records, a search engine is better at retrieving current information, and a code execution environment is better at running and validating code.

The role of the LLM increasingly becomes that of a **reasoning and orchestration layer**. It determines what needs to happen, selects the capability required to accomplish it, interprets the result, and decides what should happen next.

This is also where tool use becomes more complicated in production environments. Calling a calculator is relatively harmless, but calling a payment API, modifying a customer record, sending an email, executing infrastructure commands, or deploying code can create real-world consequences. Once agents have access to powerful tools, the architecture needs to consider permissions, authentication, validation, sandboxing, approval workflows and observability.

[https://arxiv.org/pdf/2302.04761](https://arxiv.org/pdf/2302.04761)

That gives us a more realistic production pattern:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*NJu6NLMYQAL5ASSuKnI4vw.png)

This is why Toolformer is more than an interesting research paper from the early LLM era. It represents one of the conceptual steps toward the tool-using agents we are building today.

ReAct showed us that **reasoning and action can be interleaved**. Toolformer pushed the idea of **models deciding when external tools should participate in that process**. Later agent architectures would add memory, planning, reflection, multi-agent collaboration and increasingly sophisticated orchestration around these foundations.

The frameworks may have changed, but the underlying question remains the same:

> **_What should the model do itself, and what should it delegate to an external capability?_**

For an AI engineer, that is an important design decision.

The goal isn’t to give an agent access to as many tools as possible. The goal is to give it **the right tools, the right permissions and the right decision logic** to accomplish its objective reliably.

And as models become increasingly capable of acting in the real world, this principle becomes even more important:

> **_The smarter the agent becomes, the more carefully we need to design what it is allowed to do._**

## 3\. Generative Agents — Memory, Reflection and Planning

## 3\. Generative Agents — Memory, Reflection and Planning

### Generative Agents: Interactive Simulacra of Human Behavior

Reasoning and tool use solve one part of the agent problem. But what happens when an agent needs to operate across multiple interactions?

It needs **continuity**.

The _Generative Agents_ research introduced an architecture where agents maintain memories of their experiences, retrieve relevant memories, reflect on those experiences, and use the resulting insights to influence future planning and behavior.

[https://arxiv.org/abs/2304.03442?utm\_source=chatgpt.com](https://arxiv.org/abs/2304.03442?utm_source=chatgpt.com)

The core loop looks like this:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*-umwMFYuYjSqF_DOn8vx3Q.png)

The important idea is that **past experience can influence future decisions**. Instead of treating every request as an isolated interaction, the agent develops a persistent behavioral history.

This distinction becomes particularly important in modern agent systems.

**Context is not the same as memory.**

A larger context window gives the model access to more information during the current interaction. Memory determines **what information persists beyond that interaction and becomes available later**.

That matters for long-running agents, personalized assistants, coding agents, research systems, and any workflow where decisions made yesterday should influence what happens today.

The deeper lesson from Generative Agents is therefore simple:

> **_An intelligent agent shouldn’t just remember what happened. It should use what happened to decide what to do next._**

And once an agent can remember its experiences, the next question becomes even more interesting:

**What happens when it remembers that it failed?**

That’s where **Reflexion** enters the story.

## 4\. Reflexion — Learning From Failure

### Reflexion: Language Agents with Verbal Reinforcement Learning

Once an agent can reason, use tools, and maintain memory, another challenge becomes unavoidable: **it will still make mistakes**. It may choose the wrong tool, misunderstand a requirement, produce code that fails its tests, or take an incorrect action. The important question is therefore not whether an agent will fail, but **what it does after failure**.

_Reflexion_ introduced the idea of using **verbal feedback as a form of reinforcement**. Instead of immediately changing the model’s weights, the agent reflects on what went wrong, stores that feedback in memory, and uses it to improve its next attempt.

[https://arxiv.org/pdf/2303.11366](https://arxiv.org/pdf/2303.11366)

The process can be summarized as:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*ecyR-U7zh_u5Gc7IOXfD9w.png)

This creates a simple but powerful pattern:

**Execute → Evaluate → Reflect → Retry**

We see variations of this approach today in critic and evaluator agents, verification loops, code-repair systems, and other adaptive agent architectures.

The key insight is not that the agent becomes perfect. It is that **failure becomes feedback that can influence the next attempt**.

That shift from simply generating an answer to evaluating and improving an outcome — is one of the important steps toward more adaptive Agentic AI systems.

## 5\. AutoGen — When One Agent Isn’t Enough

### AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation

The earlier research we looked at primarily focused on improving the capabilities of an individual agent — how it can reason, use tools, remember previous experiences, and learn from its failures. **AutoGen** introduced another important direction: instead of asking one agent to handle every part of a complex task, what if multiple agents could work together?

Consider a task that requires research, implementation, analysis, and review. A single agent can potentially perform all of these steps, but as the workflow becomes more complex, separating responsibilities can make the system easier to structure and manage. One agent can focus on research, another can handle implementation, while another reviews the work and identifies potential problems.

[https://arxiv.org/pdf/2308.08155](https://arxiv.org/pdf/2308.08155)

A simplified architecture looks like this:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*dMBw3cUEbkIoNvgZf4w7BA.png)

The value of this approach isn’t simply having more agents. It is about **specialization and collaboration**. A researcher can gather the information needed for a task, a coding agent can turn that information into an implementation, and a critic can evaluate the result before it reaches the user.

This research direction helped shape many of the multi-agent patterns we see in today’s systems, including supervisor architectures, planner-executor workflows, researcher-writer systems, critic and reviewer agents, parallel execution, and human-in-the-loop designs.

But multi-agent architecture also introduces a trade-off that is easy to overlook. Every additional agent creates another layer of communication, context sharing, coordination, latency, and cost. Agents can also disagree, produce conflicting outputs, or make the overall system considerably harder to debug.

That leads to an important architectural question:

> **_Should you build one powerful agent, or several specialized agents?_**

There is no universal answer. A multi-agent design makes sense when specialization genuinely improves the quality, reliability, or maintainability of the workflow. Otherwise, adding more agents can simply add complexity without improving the outcome.

The lesson from AutoGen is therefore not **“use multiple agents.”**

It is:

> **_Decompose a problem when specialization creates real value — not simply because you can._**

That principle remains just as relevant as agentic systems become more sophisticated.

## Conclusion:

### From Research Papers to Agentic Systems

The five papers in this series tell a clear story about how Agentic AI evolved.

**ReAct** introduced the idea of reasoning through action and observation. **Toolformer** showed how models could extend their capabilities by learning when to use external tools. **Generative Agents** brought memory, reflection, and continuity into the picture, while **Reflexion** demonstrated how agents could use failure as feedback. **AutoGen** expanded the architecture from individual agents to systems where specialized agents collaborate.

Together, these ideas form many of the building blocks behind the agentic systems we are building today.

And that is why understanding the research matters.

Frameworks such as LangGraph, CrewAI, AutoGen, and OpenAI Agents SDK make it easier to implement agents, but the framework is not the architecture. The real engineering decisions are about **reasoning, tools, memory, coordination, evaluation, and control**.

As frontier models move toward computer use, long-horizon execution, and autonomous task completion, these foundations become even more relevant.

The future of Agentic AI will not be defined by how many agents we can create.

It will be defined by **how reliably those agents can reason, act, learn from outcomes, and accomplish real-world goals.**

> **_Learn the papers. Understand the patterns. Then build the systems._**

That is the difference between **using an agent framework** and **engineering an agentic system**.

## If this is the kind of stuff you’re into:

I write about **LLMs, Agentic AI, AI infrastructure, research, and what actually works in production** — not just what looks impressive in a demo.

👉 [**Subscribe to LLM Insider**](https://www.linkedin.com/newsletters/the-llm-insider-7259848356124860416/) — deep dives when something is genuinely worth your time. No filler. No forced weekly cadence. Just practical ideas, research, and engineering lessons.

👉 [**Follow me on Medium**](/) — so the next deep dive shows up in your feed.

👉 [**Connect with me on LinkedIn**](https://www.linkedin.com/in/lekhapriya/) — for shorter takes, architecture breakdowns, and ideas I want to share before they become full articles.

The goal isn’t to build agents because everyone is talking about agents.

It’s to understand **why they work, where they fail, and what it takes to make them reliable in production.**

**One research paper, one architecture, one engineering lesson at a time.**