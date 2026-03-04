---
layout: post
title: "The Architecture Decision That Made Our Agent Actually Reliable"
date: 2026-03-04
categories: [agents, architecture]
tags: [planning, execution, agents, react, tree-of-thoughts, safety, agentdojo]
---

We started building an LLM agent the way most people do: give the model some tools, let it call them, feed back the results, repeat.

It worked. Kind of.

The agent could read files, run shell commands, answer questions about code. But it had a problem we didn't anticipate: **it kept losing track of what it was doing.**

Ask it to "read the README, list the source files, and write a summary," and it would read the README, then… write a summary. Skipping step two. Or it would do all three steps but in a random order that produced worse results. Or it would get halfway through and start doing something we never asked for.

The agent was *acting* fine. It just wasn't *planning*.

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

We considered more sophisticated plan representations — dependency graphs, hierarchical task networks, formal plan languages. But we kept coming back to the humble todo list:

- It's easy to inspect ("what is the agent planning to do?")
- It's easy to validate ("is exactly one item in progress?")
- It's easy to update ("mark step 1 done, move step 2 to in progress")
- It's easy to explain to the LLM ("here are your remaining tasks")

The constraint that **only one item can be `in_progress`** turned out to be surprisingly powerful. It prevents the agent from "parallel flailing" — starting three things at once and finishing none of them.

## The kitchen counter analogy

We use a metaphor internally that helps us think about the architecture:

- **TodoManager** is the **recipe card**: the structured plan with steps and statuses.
- **PlanningContext** is the **kitchen counter**: the recipe card *plus* the ability to ask the chef for advice (LLM), use kitchen tools (tool dispatch), write notes (observations), and check the timer (drift guard).

The planner needs the whole counter to do its job. But the recipe card is what gets passed to the cook (execution layer).

This separation means we can swap planning strategies without touching execution. Today we have two:

### Strategy 1: Todo planning (baseline)
The simplest approach. We rely on the system prompt to tell the model: "If this is a multi-step task, create a todo list first." The planner itself is essentially a no-op — it trusts the LLM to plan.

This works surprisingly well for simple tasks.

### Strategy 2: ToT-lite (planning-only branching)
For harder tasks, we wanted something smarter. Inspired by Tree of Thoughts, we built a lightweight planning-time search:

1. Ask the LLM to propose 3 candidate todo plans
2. Filter plans with a heuristic (must have exactly one `in_progress` item)
3. Score each plan (self-evaluation)
4. Keep the top 2
5. Refine them once (ask the LLM to improve them)
6. Pick the best

This is bounded: at most ~4 LLM calls, always converges, always produces a valid todo list. The branching happens *only* in planning. Execution stays single-trajectory.

We can switch between strategies with one environment variable:

```bash
PLANNING_MODE="todo"      # baseline
PLANNING_MODE="tot_lite"  # planning-only branching
```

## The drift problem (and how separation helped us solve it)

Even with good plans, agents drift. The model gets distracted by something in a tool result, or it "forgets" the plan exists.

Because planning and execution are separated, we could add a **drift guard** cleanly:

- The planner tracks how many tool calls have happened since the last todo update.
- After 3 non-todo tool calls, it injects a gentle reminder: "Update your todo list."
- This is a planning concern, so it lives in the planner — not in the execution loop.

If we'd mixed planning and execution together, adding drift detection would have meant scattering counter variables and reminder logic throughout the main loop. Instead, it's a few lines inside the planner class.

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
