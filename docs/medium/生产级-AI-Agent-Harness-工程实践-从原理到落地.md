---
title: 生产级 AI Agent Harness 工程实践：从原理到落地
url: https://medium.com/@tort_mario/ai-agent-best-practices-production-ready-harness-engineering-2026-guide-c1236d713fac
source_type: medium
author: Tort Mario
tags:
- AI Agent
- Harness Engineering
- LLM 生产实践
- Agent 安全
- 系统设计
summary: 拆解 AI Agent 的确定性运行时封装层（Harness）设计原则，让 LLM Agent 在生产环境可靠、安全、可观测。
fetched_at: '2026-09-16T14:01:03.806167+00:00'
---

[Ai Engineering](/tag/ai-engineering?source=post_page---header_tags--c1236d713fac-----------------------------------------)

[LLM](/tag/llm?source=post_page---header_tags--c1236d713fac-----------------------------------------)

[Agentic Ai](/tag/agentic-ai?source=post_page---header_tags--c1236d713fac-----------------------------------------)

[Programming](/tag/programming?source=post_page---header_tags--c1236d713fac-----------------------------------------)

[AI](/tag/ai?source=post_page---header_tags--c1236d713fac-----------------------------------------)

# AI Agent Best Practices: Production-Ready Harness Engineering (2026 Guide)

## Build reliable LLM agents with a provider‑neutral harness — agentic loop, tools & permissions, context compaction, security evals, and a ready‑to‑use open‑source skill.

[

![Tort Mario](https://miro.medium.com/v2/resize:fill:64:64/1*MtsnWeEsQaAgcItw878s5A.png)


](/@tort_mario?source=post_page---byline--c1236d713fac-----------------------------------------)

[Tort Mario](/@tort_mario?source=post_page---byline--c1236d713fac-----------------------------------------)

9 min readMay 16, 2026

\--

\--

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*M8j-y5OFZ5qn3ESoEuH47w.png)

The AI agent landscape is exploding. From customer support bots to financial analysis co‑pilots, agents promise to automate complex workflows. Yet most agents fail in production — not because the underlying LLM is weak, but because the harness (the runtime wrapper that governs the agent) is brittle, insecure, or unpredictable.

After analysing hundreds of failed agent deployments, a clear pattern emerges: the missing piece is _disciplined harness engineering_.

Enter `agents-best-practices` – a provider‑neutral, production‑ready skill for designing, auditing, and refactoring agentic harnesses. Built by Denis Sergeevitch and inspired by the internals of Claude Code and Codex, this open‑source repository gives you concrete artefacts: from component models to checklists. And it follows the emerging Agent Skills standard, so you can plug it into your favourite AI‑assisted environment.

👉 Grab it here: [https://github.com/DenisSergeevitch/agents-best-practices](https://github.com/DenisSergeevitch/agents-best-practices)

In this article, we’ll dissect the repository, extract the core principles of agent harness engineering, walk through a step‑by‑step guide from idea to production, and show you how to deploy your agent harness on reliable infrastructure — including a special hosting bonus.

## What is an Agent Harness (and Why Do You Need One)?

An agent harness is the deterministic runtime layer that wraps an LLM. It validates, authorises, executes, and logs every action the model proposes. The key idea is clear separation of responsibilities:

-   Model proposes actions and tool calls.
-   Harness executes them — checking schemas, permissions, budgets, and safety rules.

Without a proper harness, agents become black boxes that leak tokens, run uncontrolled loops, and execute dangerous commands. With a harness, you get predictability, security, and observability.

## Non‑negotiable principles (from the repository)

The skill defines several hard rules that apply to _any_ domain:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*H5svJfr8WNAuUxtz1u2PLA.png)

These principles are provider‑neutral — they work with OpenAI, Anthropic, open‑source models, or any future LLM.

## Deep Dive into the `agents-best-practices` Repository

The repository follows the Agent Skills specification: `SKILL.md` is the entry point, and detailed references live in the `references/` folder.

## Reference files at a glance

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*2PVJJuCbeqm4PbXP9CAC-g.png)

## Why this is not “another list of tips”

Most blog posts give abstract advice like “validate tool inputs”. This skill gives you executable artefacts:

-   A component model with 15 modules (instruction manager, context builder, model adapter, tool registry, permission resolver, budget tracker, etc.)
-   A canonical agentic loop written in pseudocode — including budgets, compaction triggers, and stop conditions
-   A risk taxonomy (read\_only, financial, destructive, etc.) and a permission matrix
-   A cache‑aware ordering strategy to slash prompt caching costs
-   Security evals that test not just the model, but the harness itself (injection, timeouts, over‑tooling)

If you are an ML engineer, platform architect, or team lead, this skill will change how you build agents.

## Key Principles for Production‑Ready Agents (extracted from the skill)

Let’s zoom in on the most impactful principles. Each one addresses a real failure mode we’ve seen in production.

## 1\. Model proposes — harness executes

Never let the LLM call tools directly. Instead, the model returns a structured tool call; the harness validates the schema, checks permissions, executes, and injects the result back. This prevents prompt injection from escalating to arbitrary code execution.

## 2\. Each tool call returns a result — even on failure

Whether it’s a successful API response, a permission denial, or a timeout, the agent always receives a structured observation. No dangling promises.

## 3\. Risk changes the process

Use at least three risk levels:

-   Read‑only (autonomous)
-   Draft (internal simulation, no external side effects)
-   External write (requires approval)

This is the draft‑commit pattern — dangerous actions are first drafted, then explicitly committed.

## 4\. Context is assembled, not dumped

Don’t shove entire conversation history into every turn. Use layers:

-   Policies (system‑level, rarely change)
-   Scoped instructions (per‑task or per‑domain)
-   Runtime hints (JIT‑retrieved from memory or tools)

Mark untrusted data (e.g., user input) with a trust label so the harness can treat it differently.

## 5\. Long tasks have budgets

Every agent loop must have:

-   Step budget (max iterations)
-   Time budget (wall‑clock)
-   Token budget (per turn and cumulative)
-   Cost budget (USD limit)

When a budget is exhausted, the harness terminates gracefully and returns a structured failure.

## 6\. Recurring failures become harness features

If your agent repeatedly fails because a tool returns a malformed response, don’t fix it in the prompt — write a validator function inside the harness. If the agent asks for the same missing information every time, build a tool that retrieves it automatically.

## Step‑by‑Step Guide: From Idea to Production Using the Skill

The repository provides a concrete methodology: Map → Identify → Blueprint → Implement → Launch.

## Phase 1: Map (ask the right questions)

Before writing any code, answer:

-   What domain? (customer support, finance, DevOps, etc.)
-   What level of autonomy? (Level 0 = human does everything, Level 4 = fully autonomous)
-   What risk level? (read‑only, financial, destructive)
-   Which external systems? (Slack, Linear, Drive, databases, APIs)

## Phase 2: Identify (choose MVP level)

Based on your answers, select an MVP level from the `mvp-agent-blueprint.md` table. For most first‑time agents, Level 1 (human approves every external write) or Level 2 (plans are human‑approved, execution may be autonomous for low‑risk steps) is the sweet spot.

## Phase 3: Blueprint (generate the harness design)

Now ask the skill to generate a blueprint. You can do this by simply describing your domain to an AI assistant that has the skill installed. The output will include:

-   Goal and domain boundaries
-   Agentic loop (stop conditions, budgets)
-   Tool registry (each tool with typed schema and risk class)
-   Permission matrix (who can call what, when approval is needed)
-   Context & memory layering (what stays in cache, what is JIT‑fetched)
-   Skills & connectors (which Agent Skills or MCP servers to use)

## Phase 4: Implement (strictly follow the blueprint)

Build the MVP exactly within the described boundaries. Start with the skeleton and the validation path — then add measured extensions. The `checklists.md` file provides a line‑by‑line implementation checklist.

## Phase 5: Launch (audit before going live)

Before production, run the audit checklists from `checklists.md`. Verify:

-   Budgets are enforced
-   Permissions are correct (no `execute_anything` tools)
-   Evals for injection and timeouts pass
-   Observability (traces, logs) is in place

## Real‑world case: Contract risk analysis agent

Using the skill, a team built an agent that:

-   Reads contract drafts (read‑only, autonomous)
-   Produces a risk brief and draft actions (draft mode)
-   Sends emails only after explicit approval (external write, requires approval)

The harness blueprint was generated in 15 minutes, implementation took two days, and the agent has run for six months with zero unauthorised actions.

## Case: Auditing an existing research agent

When auditing a failing agent, the skill revealed:

-   No hard budgets → agent loop ran for 200+ steps
-   Context compaction erased active approvals → agent lost state
-   No evals for injection → user could trick the agent into deleting files

The skill provided a step‑by‑step remediation plan: add budgets, fix compaction ordering, and write three security evals.

## Practical Implementation Tips (with pseudocode)

### The canonical agentic loop (simplified)

python

budgets = Budgets(step=25, time=120, tokens=8000, cost=0.50)
context = build\_initial\_context()
permissions = load\_permission\_matrix()while not budgets.exhausted():
    response = model.generate(context, tools=typed\_tool\_schemas)
    if response.finish\_reason == "stop":
        break
    if response.tool\_calls:
        for tool\_call in response.tool\_calls:
            if not permissions.is\_allowed(tool\_call):
                observation = "Permission denied: " + tool\_call.name
            else:
                # Execute with risk‑appropriate checks
                if permissions.risk(tool\_call) == "external\_write":
                    approval = request\_human\_approval(tool\_call.draft)
                    if not approval:
                        observation = "Human rejected: " + tool\_call.name
                    else:
                        observation = execute\_tool(tool\_call)
                else:
                    observation = execute\_tool(tool\_call)
            context.append(observation)
        # Optional compaction trigger
        if context.token\_count() > budgets.token\_per\_turn:
            context = compact\_context(context, preserve\_approvals=True)
    else:
        # No tool calls, just final answer
        break

### Tools: from bad to good

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/1*CH5OwencRE4roDYORBB3aQ.png)

The skill’s `tools-and-permissions.md` provides a complete taxonomy and a permission matrix you can copy directly.

## Context layering for cache efficiency

text

Layer 0: System policies (stable prefix) → Cached
Layer 1: Agent skill definitions (rarely change) → Cached
Layer 2: User session instructions (per conversation) → Not cached
Layer 3: JIT‑retrieved tool outputs (fresh) → Not cached

By arranging layers from most‑stable to least‑stable, you maximise prompt caching and reduce costs. The `prompt-caching-and-cost.md` file shows exactly how to implement cache‑aware ordering with deterministic serialisation.

## Deploying Your Agent Harness: Why Infrastructure Matters

You’ve built a robust harness — great. But where will it run? Agent harnesses are latency‑sensitive and stateful. They need:

-   Low‑latency compute (LLM API calls + local validation)
-   DDoS protection (agents are public‑facing endpoints)
-   Reliable uptime (production agents can’t go down)
-   Flexible scaling (from MVP to millions of requests)

### This is where Aeza comes in.

[Aeza](https://aeza.net/?ref=601757) is a modern cloud hosting provider launched in December 2021, specialising in VPS/VDS and dedicated servers with built‑in DDoS protection. Their infrastructure is perfect for running agent harnesses — whether you’re hosting a lightweight Python loop or a distributed swarm of agents.

### Why Aeza for your agent workloads?

-   Low‑latency VPS in multiple regions — minimises round‑trip time to LLM APIs.
-   Automatic DDoS protection — because agents are often targeted by prompt injection attacks that try to exhaust resources.
-   Flexible configurations — start with a small VPS for testing, upgrade to dedicated metal for production.
-   Developer‑friendly — no hidden quotas, full root access, support for Docker and Kubernetes.

**👉 Special offer for readers:
**When you register via [my Aeza referral link](https://aeza.net/?ref=601757), you get a 15% bonus for the first 24 hours after registration. Use that extra credit to spin up a VPS, deploy your agent harness, and test it under real load.

I’ve been using Aeza for several agent deployments — the combination of raw performance, DDoS protection, and transparent pricing makes it a hidden gem among European providers.

## Security, Observability, and Evals (Don’t Skip This)

The repository’s `security-evals-observability.md` is worth its weight in gold. It provides:

-   A threat model for agent harnesses (injection, denial‑of‑service, tool abuse, approval spoofing)
-   Multi‑level guardrails (input sanitisation, permission checks, output validation)
-   Tracing format (every step: prompt, tool call, observation, latency, cost)
-   Evals for the harness itself — not just model accuracy:
-   _Injection resistance_: can a user prompt overwrite system instructions?
-   _Timeout resilience_: does the harness stop when a tool hangs?
-   _Over‑tooling_: does the agent request unnecessary tools?

Run these evals before launch. The skill includes ready‑to‑use test cases.

## Conclusion and Next Steps

The `agents-best-practices` repository is a practical, deep, and provider‑neutral guide to building production‑grade agent harnesses. It’s not abstract theory – it’s a collection of concrete artefacts: component models, pseudocode, checklists, and security evals.

**Whether you are:**

-   An ML engineer building the next generation of autonomous agents,
-   A platform architect designing skill delivery and MCP infrastructure,
-   A team lead auditing existing agent codebases,
-   Or a security/compliance specialist implementing guardrails outside the model,

…this skill will save you months of trial and error.

## Your immediate action plan

1.  Install the skill

npx skills add DenisSergeevitch/agents-best-practices -g

Or clone manually:

git clone https://github.com/DenisSergeevitch/agents-best-practices.git ~/.codex/skills/

2\. Read the `SKILL.md` and pick one reference file that matches your current pain point (e.g., `agentic-loop.md` if your agent runs forever).

3\. Generate a blueprint for your domain — ask your AI assistant (Claude Code, Codex, etc.) to use the skill and produce a harness design.

4\. Deploy your harness on reliable infrastructure. Try Aeza for a low‑latency, DDoS‑protected VPS — and don’t forget the 15% first‑day bonus when you register via [my link](https://aeza.net/?ref=601757).

5\. Share your experience — open an issue or PR on the GitHub repo. The more production harnesses we build using these patterns, the better the whole ecosystem becomes.

The era of flaky, unpredictable agents is ending. With disciplined harness engineering, we can build AI agents that are safe, reliable, and cost‑effective — at any scale.

_Liked this guide? Clap, follow, and check out the original repository:_ [_github.com/DenisSergeevitch/agents-best-practices_](https://github.com/DenisSergeevitch/agents-best-practices)_. For infrastructure, see_ [_Aeza_](https://aeza.net/?ref=601757) _— cloud hosting with DDoS protection and a 15% new‑user bonus._

[**My Twitter**](https://x.com/Tort_Mario) — a fundamental analysis of coins and earnings on AirDrops. I buy cryptocurrency on [**Bybit**](https://partner.bybit.com/b/60057) | [**MEXC**](https://www.mexc.com/ru-RU/register?inviteCode=mexc-1RPe1) | [**CryptoBot**](https://t.me/CryptoBot?start=r-261272)