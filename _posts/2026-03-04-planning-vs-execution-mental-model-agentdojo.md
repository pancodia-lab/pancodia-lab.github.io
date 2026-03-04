---
layout: post
title: "Planning vs Execution: A Mental Model for Building Reliable Agents (AgentDojo Walkthrough)"
date: 2026-03-04
categories: [agents, architecture]
tags: [planning, execution, agents, react, tree-of-thoughts, safety, agentdojo]
---

When people say “build an AI agent,” they often jump straight to tool calling.

But in practice, the difference between a toy agent and a reliable one comes down to architecture—specifically: **do you cleanly separate planning from execution?**

In this post, I’ll explain the **Planning vs Execution** mental model in plain language, why it matters for safety and reliability, and how we apply it in AgentDojo.

## 1) The core idea
A useful agent architecture distinguishes:

- **Planning**: deciding *what to do* (and in what order)
- **Execution**: actually *doing it* (running tools, reading files, producing outputs)

This sounds obvious, but without the separation you tend to get:

- messy control flow
- drift (“we’re doing stuff but not sure why”)
- difficulty verifying correctness
- hard-to-debug behavior

## 2) Why separation matters
### Reliability
Planning is where you want branching, scoring, and “thinking.”
Execution is where you want deterministic, safe, observable steps.

If you mix them, you get partial plans changing mid-execution without a clear trace.

### Safety
The most dangerous parts of an agent live in execution:

- shell commands
- network access
- file writes

You want execution to be:
- constrained
- auditable
- gated (human approval for risky actions)

Planning should not be able to bypass those constraints.

### Verification
Verification is easier when you can say:

- Here is the plan (structured)
- Here are the executed steps
- Here is the result
- Here is the check that says we’re done

## 3) A concrete planning contract: the todo list
In AgentDojo, we chose a simple planning representation:

- a structured todo list
- exactly one item can be `in_progress`

Why this works:
- it is easy to inspect
- it is easy to validate
- it is easy to update
- it prevents “parallel flailing” (multiple half-started actions)

Even if we change planning strategies (simple todo vs ToT-lite), we keep the same contract.

## 4) Planning strategies: todo vs ToT-lite
AgentDojo supports multiple planning strategies:

- **Todo planning (baseline)**: rely on prompt rules to force “todo-first” on multi-step tasks.
- **ToT-lite planning (planning-only branching)**:
  - propose multiple candidate todo plans
  - score them
  - refine top candidates
  - pick the best

Important: branching happens in *planning*, not execution. Execution stays single-trajectory.

That gives us much of the benefit of Tree-of-Thoughts style search without exploding runtime complexity.

## 5) Execution loop: tool chaining within a single user turn
AgentDojo is an interactive REPL:

- outer loop: one user input at a time
- inner loop: the agent may perform multiple tool calls consecutively

Mechanically:

1) call LLM
2) if it outputs a tool call → execute tool
3) append tool result as an observation
4) call LLM again
5) stop when the model returns a normal (non-tool) response

This lets a multi-step plan sometimes complete in one “turn,” as long as the model keeps emitting tool calls.

## 6) The glue: PlanningContext (keep the seam small)
A clean separation requires a clean interface.

In AgentDojo we introduced a small seam called `PlanningContext`:

- it contains the planning state (`TodoManager`)
- plus the minimal “capabilities” planners need (call LLM, call tools, emit logs)

This avoids planners reaching deep into `main.py` internals and keeps the orchestrator readable.

## 7) Takeaway
Planning vs Execution is a high-level design choice that pays off immediately:

- safer tool use
- easier debugging
- better upgrade path (verification, reflexion, memory)
- ability to add smarter planning (branching/search) without destabilizing execution

If you’re building agents, my suggestion is:

1) pick a planning contract (todo list is a great start)
2) keep execution constrained and auditable
3) add smarter planning strategies later

---

**Project:** AgentDojo — build an LLM agent from zero to hero, grounded in industry practice and state-of-the-art research (spec-driven, security-first).
