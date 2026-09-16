---
title: 从296个Agent到一张图：GPT-6 Astra时代的图与循环工程
url: https://x.com/N01ennn/status/2096962591125905888
source_type: twitter
author: N01ennn
tags:
- Agent工程
- Graph Engineering
- Loop Engineering
- 多Agent编排
- LLM可靠性
summary: 296个Agent不是能力而是噪音，只有用Harness、Loop、Graph三层结构把它们收敛成一张可检查的执行图，才真正可控。
fetched_at: '2026-09-16T11:39:29.279136+00:00'
---

296 agents is a number. On its own it means nothing.

You can run 296 workers and have 296 separate policies, 296 ways to fail, and no way to say what the system did last Tuesday. Or you can run 296 workers as capacity inside one structure that says what happens next, what counts as done, and what an unattended run is allowed to touch.

The paper that sits under this title, Towards Agentic Cloud Engineering: Graph and Loop Engineering with a Zero-Trust Agent Harness (arXiv 2609.00050), puts it plainly: without three layers, 296 agents is noise. With them, 296 agents collapse into one structure.

This is what those layers are, what the repositories actually show, and where GPT-6 Astra fits inside them.

## Three layers people keep mixing

The confusion is understandable. All three ideas sit around the same model, all three influence reliability, and all three can contain loops. They are not synonyms.

Harness engineering is the surrounding machinery: context, tools, permissions, persistence, control, safety, observability. It answers what can the model access, and how is execution controlled?

Loop engineering is the repeated observe / act / verify cycle with explicit triggers and evidence-based stopping rules, rather than "keep trying." It answers how does the system keep working until evidence says it's done?

Graph engineering models workflow topology as nodes and edges to control allowed next steps, branching, concurrency, state transitions, and recovery paths. It answers which step runs next, under what condition, and with what state?

The memory aid: harness makes the model operate, loops make execution iterative and resumable, graphs make control flow inspectable.

The reason the distinction matters is that mixing them up produces expensive failures: drawing graphs before understanding behavior, letting the same model grade its own work without safeguards, creating unbounded retry loops, stuffing the harness with too many tools or overly broad permissions, and blaming the model for orchestration problems that belong to a different layer.

## Forty days of loop engineering

The term was coined in the first week of June 2026, and three posts landed almost on top of each other.

On June 2, Boris Cherny, who built Claude Code, said in an interview: "I don't prompt Claude anymore. I have loops that are running. They're the ones that are prompting Claude and kind of figuring out what to do. My job is to write loops."

On June 7, Peter Steinberger posted that you shouldn't be prompting coding agents anymore, you should be designing loops that prompt your agents. By mid-July that post had passed eight million views. The same day, Addy Osmani named the practice loop engineering and defined it as replacing yourself as the person who prompts the agent.

The technique predates the name. In July 2025, Geoffrey Huntley published the "Ralph Wiggum" loop, which in its purest form is a shell one-liner:

It works around the finite context window by restarting the agent with a fresh context on every iteration while persisting progress to the file system. The agent forgets. The repo doesn't.

Vendors absorbed it fast. Claude Code shipped a recurring /loop command in March 2026, Codex introduced goal objects with machine-checkable completion conditions in April, and /goal commands followed across harnesses in May.

Osmani's taxonomy orders loops by how much you hand off:

He later separated two dimensions that a single ordering tends to conflate: agency (how far one agent proceeds without human intervention) and orchestration (how many agents run at once and who coordinates them). And he set the practical bound: raise autonomy only as far as you can check the result at acceptable cost. Verification cost is what bounds delegation.

## What the repositories actually show

Everything above is discourse. Here is measurement.

Lulla and colleagues scanned 36,710 engineered software repositories, inspected 256 candidates by hand, and confirmed 217 running autonomous agent loops, which is 0.59% of the corpus. Claude Code accounted for 189 of the 217.

The shape of those loops is not what the discourse describes:

- 180 ran on repository events only, mostly reviewing each newly opened pull request

- 21 ran on a schedule only, mostly issue triage

- 15 ran on both

Now the part worth sitting with. The community consensus prescribes committed state files, verifier subagents, budget declarations, and machine-checkable stop conditions. Across all 36,645 scanned repositories, the scan found:

- 2 state files matching the content criterion, both rejected on inspection as unrelated uses of the word "loop"

- 0 verifier subagents referenced by a loop artifact

- 0 budget files

- 0 cost logs with measured values

- 0 stop conditions matching the phrase list

Configuration gets committed. Runtime state does not.

There's a reason, and it isn't laziness. A pull-request review run reads the PR, posts its review, and finishes. The next run starts from the next PR. There is nothing to persist. A scheduled triage run starts from whatever the issue tracker holds at that moment, so the tracker carries state between runs, not a file. The prescription assumed a loop shape that most projects aren't running.

One finding deserves to be framed and hung on a wall. In AI-Hypercomputer/xpk, an hourly Gemini triage workflow logged 6,290 successful runs and zero agent executions. Its gating issue search matched nothing, every time. Green pipeline. No work. A successful run is weak evidence that anything happened.

## What happens when a loop has no stop

The other empirical study is the one about failure. IAL-Scan analyzed 6,549 LLM agent repositories (246,748 Python files, 33.41M lines), looking for what the authors call Infinite Agentic Loops: an agentic feedback path that repeatedly triggers model calls, tool invocations, agent runs, or workflow transitions without an effective termination condition.

It reported 74 findings. 68 were confirmed IAL failures across 47 projects, at 91.9% precision.

The distribution:

And the impact: 95.6% API cost exhaustion, 95.6% model denial of service, 27.9% context window exhaustion.

Two frameworks produced 45 of the 68 findings: LangGraph (23) and AutoGen (22). Both encode feedback through APIs rather than visible loop syntax, which is exactly why the failures survive code review. Nobody sees a while True. They see add_conditional_edges.

Here is the shape, condensed from a real finding:

Model output → message state growth → another model call. No retry cap. No timeout. No context guard. The loop does have an exit, but the exit is decided by model output, and model output is exactly what's failing.

A visible exit is not a bound. That distinction is the whole lesson.

Every mainstream framework already ships loop control: max_iterations in LangChain, max_turns in the OpenAI Agents SDK, recursion_limit in LangGraph, max_iter in CrewAI. IAL isn't the absence of the feature. It's a bound that doesn't cover the repeated path: set on an inner agent call while the outer evaluator cycle runs free, or placed outside the feedback path entirely.

## The part that actually collapses 296 into one

The Graph Engineering survey gives the progression a clean formulation:

Harness engineering determines what capabilities are available. Loop engineering determines how the agent keeps interacting and adapting. Together they produce Individual Intelligence: one agent pursuing goals over time.

Individual intelligence has three structural limits, and every one of them shows up in a 296-worker deployment:

1. Parallel and interdependent work gets serialized. In a fault diagnosis task, log analysis, failure reproduction, and code inspection are independent branches; repair and testing depend on their results. A single agent loop compresses all of it into one sequential trace, loses the parallelism, and makes the faulty stage hard to localize.

2. Expertise and verification collapse into one role. When the same agent writes and evaluates the code, it can mistake its own judgment that the code is correct for evidence that it actually is, even when prompts assign it different roles.

3. State is context, not a record. An error that enters the loop early gets carried through later steps and stays hidden until the task fails near the end. By then, finding where it first appeared is expensive.

The survey's central claim: System Intelligence is not equivalent to increasing the number of agents. A multi-agent system can contain many capable agents and still lack work organization, responsibility boundaries, coordination mechanisms, and consistent state management.

Graph Engineering makes the relations explicit across three views:

- Task Organization: what to do. Goal decomposition into a graph of subgoals and dependencies; compilation into executable workflows with scheduling, concurrency, and verification constraints.

- Agent Coordination: who works. Capability modeling, team topology (chains, routing, fan-out/fan-in), and communication graphs that decide which information paths are worth maintaining.

- Runtime State Management: how the system operates. State recording with provenance and versions, fault localization that traces an error back through dependencies, and failure recovery that resumes from a validated boundary instead of discarding valid work.

## The graph, in code

LangGraph models this as a Pregel-style message-passing program that proceeds in discrete super-steps. Nodes that run in parallel belong to the same super-step; sequential nodes belong to separate ones. A node becomes active when it receives a message on an incoming edge, runs, and emits updates. Execution terminates when all nodes are inactive.

State is a shared schema, and each key gets its own reducer, a binary function where the left argument is accumulated state and the right argument is the node's update:

This is where a lot of retry logic quietly breaks. With a merging reducer, returning an empty value does not clear the field; the empty update gets merged in and previous values survive. Error buffers and retry counters that need clearing between attempts have to bypass the reducer explicitly with Overwrite.

Routing and state updates combine in a single return:

Command(graph=Command.PARENT) navigates from a subgraph node to a node in the parent graph, the primitive behind multi-agent handoffs. Send handles the case where the number of edges isn't known ahead of time, which is the map-reduce pattern a swarm actually needs:

And now the bound. LangGraph's default recursion limit is 1000 super-steps, after which it raises GraphRecursionError. That's the reactive path: catch the exception outside the graph, after execution has already failed. The proactive path uses RemainingSteps, a managed value that LangGraph populates automatically:

The difference is not cosmetic. Reactive handling terminates execution and throws away everything. Proactive handling lets the graph reach a completion node, checkpoint its intermediate state, and return a partial result. Same limit, different outcome.

That's the bound the IAL study found missing in 100% of confirmed failures, expressed at the layer that can actually enforce it.

On orchestration, the OpenAI Agents SDK draws a line worth keeping: agents-as-tools when a manager should own the final answer and call specialists for bounded subtasks, handoffs when routing is itself the workflow and the chosen specialist should own the rest of the turn. Mixing them is fine. Confusing them is how you end up with two agents both narrating the same result.

## Where GPT-6 Astra sits

Astra is the model inside the nodes (do, check, fix), not a prompt-only orchestrator. Four things about it change how you build the graph.

Async tool calling. Set async: true on a function or custom tool, and Astra continues reasoning, calls other tools, or answers independent parts of a request while your application runs the long one, returning the result later against the original call_id. That maps directly onto parallel branches inside a super-step instead of a serialized trace.

Mid-turn steering. You can send additional user instructions while Astra is working, a correction or a changed requirement, and the Responses API preserves completed work and folds the update into a continuation. This is human escalation without restarting the node, which is exactly the escalation point the loop-engineering literature keeps prescribing and almost nobody implements.

Reasoning effort per stage. A configuration_update input item raises effort for a hard node and lowers it for routine follow-ups, without rewriting the original prompt prefix, so the prompt cache survives the change. Effort as a property of the node rather than the run.

Misalignment monitoring. Systems asynchronously monitor for misalignment and trigger alerts. That belongs to the harness layer's job description, and it's shipping in the model tier.

Then there's the economics, which is where the IAL study stops being academic. Astra is $10 per 1M input tokens and $50 per 1M output, with cached input at $1. Context is 1,050,000 tokens, but prompts over 272K input tokens are billed at 2x input and 1.5x output for the entire request. A loop that appends to messages every iteration is not just growing context. It is walking toward a pricing cliff, and then walking off it repeatedly. Astra also doesn't support none reasoning effort, so every node call reasons. An unbounded loop costs more per iteration than it used to.

Three prompting notes that matter specifically for unattended nodes:

- Astra asks more clarifying questions by default. That makes it a better collaborator and a worse unattended worker. If a node runs without a human, prompt for follow-through explicitly: infer intent from context, complete authorized work, ask only when the answer changes the outcome.

- It is more sensitive to instructions in skills and AGENTS.md. Vague or conflicting guidance in a skill file can make it pause and block work early. Audit those files, and make the precedence of user instructions over skill instructions explicit.

- It delegates less than you may want. If your harness has subagents, prompt the delegation behavior directly.

## Five rules to take into the build

Straight from the zero-trust paper, and each one maps to a finding above:

1. Progress only on evidence: code, deploy, test green. Not on agent prose, and not on a workflow run that exited zero without doing anything.

2. Cap retries, or the swarm never stops. Then check that the cap covers the feedback path, not an inner call inside it.

3. The end state is a verified output or a logged refusal. Nothing else terminates a node.

4. Permissions are per step, not per session. The model proposes an action; the harness decides whether it may run.

5. You own the graph. Astra does not freely pick the next of 296.

## The number, finally

Multiple terminals (parse, code, computer, report) should not each spawn a new swarm. Each terminal is an entry node into one graph. 296 is pool capacity for workers inside graph + loop + harness. It is not 296 separate policies, and it never was.

Loop engineering got forty days between being coined and being declared dead. On July 17, Steinberger asked whether the conversation was still about loops or had moved to graphs, and within hours the "loop engineering is dead, enter graph engineering" posts landed.

The label churns. The design problem doesn't. Bounding a non-deterministic worker with verification, budgets, and escalation is the same engineering whatever you call it next.