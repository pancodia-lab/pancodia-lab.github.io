---
layout: post
title: "The Architecture Decision That Made Our Agent Actually Reliable"
date: 2026-03-04
categories: [agents, architecture]
tags: [planning, execution, agents, react, tree-of-thoughts, safety, agentdojo]
---

We started building an LLM agent the way most people do: give the model some tools, let it call them, feed back the results, repeat.

It worked. Kind of.

The agent could read files, run shell commands, answer questions about code. But we saw a practical reliability gap: **multi-step tasks were not consistently guided by an explicit plan.**

In early iterations, behavior depended heavily on prompting style. Sometimes the agent chained steps in one turn; other times it paused after a partial step and needed another user nudge (for example, a follow-up like “proceed”). That inconsistency pushed us to make planning explicit instead of implicit.

So the issue wasn't “the agent can never act.” The issue was: action quality was okay, but multi-step control was too fragile without a planning contract.

## The lightbulb moment

We were reading through the research papers that inspired modern agent architectures — ReAct, Plan-and-Solve, Tree of Thoughts — and a common thread jumped out:

**The agents that work well separate planning from execution.**

Not as a theoretical nicety. As a practical engineering choice. Planning is where you decide *what to do* and *in what order*. Execution is where you *do it*. These are different concerns with different failure modes, and mixing them together makes both worse.

We decided to take this seriously in our codebase.

## What "separation" looks like in practice

In AgentDojo, we drew a line:

![AgentDojo planning vs execution design diagram](/assets/img/agentdojo-planning-vs-execution-design.svg)
*Design diagram: planning owns policy and plan quality; execution owns safe, auditable action loops.*

**Planning layer** (decides what):
- Proposes a structured todo list
- Can use multiple LLM calls to brainstorm, score, and refine plans
- Enforces invariants (e.g., only one task can be "in progress" at a time)
- Runs *before* the execution loop

**Execution layer** (does things):
- Calls tools (read files, run commands)
- Feeds observations back to the LLM
- Handles safety gates (human approval for dangerous commands)
- Runs *after* planning has produced a plan

The contract between them is simple: planning produces a **structured todo list**. Execution follows it.

## Why a todo list (and not something fancier)

To be precise about our actual development path: we did **not** start by evaluating multiple formal plan representations. We first followed the [learn-claude-code s03 todo example](https://github.com/shareAI-lab/learn-claude-code/blob/main/agents/s03_todo_write.py) pattern (structured todo list), then iteratively refined it in AgentDojo.

That todo-first choice turned out to be a strong baseline:

- It's easy to inspect ("what is the agent planning to do?")
- It's easy to validate ("is exactly one item in progress?")
- It's easy to update ("mark step 1 done, move step 2 to in progress")
- It's easy to explain to the LLM ("here are your remaining tasks")

The constraint that **only one item can be `in_progress`** has been practically useful for us so far. We use it as a guardrail against parallel task drift (starting too many threads at once without clear completion order).

## The kitchen counter analogy

A metaphor that helped us explain this more clearly is a kitchen setup.

<img src="/assets/img/kitchen-counter-real.jpg" alt="Real kitchen counter with cookbook stand" width="520" />

*Real-life anchor for the analogy: a recipe on a kitchen counter. (Image source: [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Cookbook_stand_in_kitchen_on_counter_with_cutting_board_and_knife_with_basil_leaves_(18501227740).jpg))*

Imagine the planner as someone cooking from a **recipe card**. The card itself is just the plan: what steps exist and what status each step is in. In our code, that role maps to `TodoManager`.

But real cooking needs more than a card — you need a **counter** where all your working context lives: the recipe card, access to tools, and room to adjust as you go. In our code, that broader working surface maps to `PlanningContext` (plan state + LLM calls + tool dispatch + observation logging + drift checks).

So the planner works with the whole counter, while execution consumes the concrete steps from the card.

This separation means we can swap planning strategies without touching execution. Today we have two:

### Strategy 1: Todo planning (baseline)
The simplest approach. We rely on the system prompt to tell the model: "If this is a multi-step task, create a todo list first." The planner itself is essentially a no-op — it trusts the LLM to plan.

In our current implementation, this is a practical baseline for simple tasks.

### Strategy 2: ToT-lite (planning-only branching)
For harder tasks, we wanted something smarter. Inspired by Tree of Thoughts, we built a lightweight planning-time search:

1. Ask the LLM to propose 3 candidate todo plans
2. Filter plans with a heuristic (must have exactly one `in_progress` item)
3. Score each plan (self-evaluation)
4. Keep the top 2
5. Refine them once (ask the LLM to improve them)
6. Pick the best

This controller is bounded in the current implementation: up to 4 planning LLM calls in the full path (propose, score, refine, choose), with earlier fallback on parse/validation failures. So runtime terminates predictably, but plan quality is not theoretically guaranteed. The branching happens *only* in planning. Execution stays single-trajectory.

We can switch between strategies with one environment variable:

```bash
PLANNING_MODE="todo"      # baseline
PLANNING_MODE="tot_lite"  # planning-only branching
```

## The drift problem (and how separation helped us solve it)

Here’s what “drift” looks like in our actual AgentDojo loop.

A typical turn is:

LLM → tool call → observation → LLM → tool call → …

That’s powerful (it lets the agent finish multi-step tasks in one user turn), but it has a side effect: the model can start reacting to the *latest* tool output and slowly lose the “plan anchor.” In practice, that can look like:

- continuing to call `read_file` repeatedly because the last file mentioned another file
- taking more steps without updating the todo state, so the todo no longer matches reality

We addressed this with a small, very specific guardrail that matches our architecture:

- In both `TodoPlanner` and `ToTLitePlanner`, we maintain an internal counter (`_rounds_since_todo`).
- After each tool call:
  - if the tool was `todo`, we reset the counter to 0
  - otherwise we increment it
- If we hit a threshold (today: `reminder_every=3`) without a `todo` update, the planner injects a **reminder observation**:
  - "Update your todo list if this is a multi-step task."

Two important implementation details that make this clean:

1) The drift guard lives in the **planner** (`before_tool`/`after_tool` hooks), not in `main.py`. The execution loop stays simple and doesn’t accumulate “planning policy” branches.

2) The reminder is injected as an **observation** (via `ctx.add_observation(...)`), so it becomes part of the next LLM context without requiring a special control channel.

This is a good example of why the separation pays off: “drift handling” is planning policy, so we can evolve it (thresholds, smarter triggers, verification) without destabilizing execution.

## Why this matters for safety

The separation has a security benefit we didn't fully appreciate at first.

Execution is where the dangerous things happen: shell commands, file writes, network calls. By keeping execution constrained and auditable, we can enforce safety gates without worrying about planning logic interfering.

In AgentDojo:
- `run_bash` requires explicit human approval and runs in a Docker sandbox (no network, non-root user, 30-second timeout)
- `read_file` is read-only and blocks path traversal
- Planning *cannot bypass* these constraints because planning and execution communicate only through the todo list

This is a much easier security story than "the LLM can do anything and we hope the prompt tells it not to."

## What we haven't built yet (and why the separation makes it easier)

Because planning and execution are cleanly separated, we can see a clear upgrade path:

- **Verification stage**: after execution, check whether the result matches the plan. This is a natural extension — add a "verify" step after "act."
- **Reflexion**: if verification fails, generate a reflection and re-plan. This is a planning concern, so it goes in the planner.
- **Memory / RAG**: retrieve relevant context before planning. This feeds *into* planning without touching execution.

Each of these would be much harder to add if planning and execution were interleaved.

## Takeaway

If you're building an agent and you take away one architectural idea, let it be this:

**Separate planning from execution. Use a simple, inspectable plan representation. Keep execution constrained and auditable.**

You can start with the simplest possible planner (a no-op that trusts the LLM). You can upgrade to smarter planning later. The important thing is drawing the line early, because it's much harder to separate them after they're tangled together.

---

*This post is part of the [AgentDojo](https://github.com/pancodia-lab/agent-dojo) series — building an LLM agent from zero to hero, grounded in research and engineering discipline.*
