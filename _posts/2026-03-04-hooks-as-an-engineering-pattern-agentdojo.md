---
layout: post
title: "Hooks Are an Engineering Pattern (and Why We Used Them in AgentDojo)"
date: 2026-03-04
categories: [engineering, architecture, agents]
tags: [hooks, design-patterns, observability, agentdojo, python]
---

Hooks show up everywhere in real software: build systems, web frameworks, game engines, CLIs, IDE plugins—basically anywhere a core system wants to stay stable while still allowing behavior to be extended.

In this post I’ll explain **hooks as a pattern**, then walk through a concrete example from **AgentDojo** (our “build an agent from zero to hero” project): we used hooks to make debug/observability pluggable without turning the main agent loop into a pile of print statements.

## 1) What is a hook?
A **hook** is a *well-defined callback point* exposed by a core system.

- The core system runs the main logic.
- At specific moments (events), it calls hook methods.
- Hook implementations can *observe* or *extend* behavior.

Importantly: the core system chooses **when** hooks fire. Hook implementers choose **what** to do when they fire.

### Hook vs “plugin” vs “middleware”
These terms overlap, but here’s a useful mental distinction:

- **Hook**: “I will notify you when X happens.” (callback/event surface)
- **Plugin**: a packaged unit of functionality (often built on hooks)
- **Middleware**: a chain that intercepts and can modify a request/response (more invasive)

AgentDojo uses hooks mainly for **observability**: print structured debug output without changing behavior.

## 2) Why hooks are useful
Hooks solve a classic tension:

- You want the **core loop** to be simple, readable, and safe.
- You still want flexibility: logs, traces, metrics, debug UIs, audits.

If you hardcode debug output into core logic, you get:

- tangled code paths
- duplicated printing
- inconsistent formatting
- harder testing

Hooks let you keep one clean orchestrator and swap in different “views.”

## 3) AgentDojo example: tool-call tracing without clutter
AgentDojo has an inner loop that can do multiple consecutive tool calls in one user turn:

LLM → tool call → execute tool → observation → LLM → …

We want to observe that process during development. But we do **not** want to bake debug output into every branch of `main.py`.

So we added a hook interface (`src/hooks.py`) roughly like:

- `on_tool_call_detected(...)`
- `on_tool_request(...)`
- `on_tool_result(...)`

And a `NullHooks` implementation that does nothing.

Then `main.py` becomes something like:

1. run LLM
2. parse tool call
3. call hooks
4. execute tool
5. call hooks

The key point: **the agent still behaves the same** regardless of hooks.

## 4) A good hook rule: “hooks should be side-effect-only”
A common failure mode is letting hooks change state.

In AgentDojo we treat hooks as:
- logging/printing only
- no mutation of agent state
- no decisions

Why? Because once hooks can change behavior, debugging becomes chaotic:

> “Was that outcome caused by the agent logic or by a hook?”

If you *do* want extensibility that changes behavior, you usually want a different mechanism (a policy engine, middleware chain, or strategy interface).

## 5) Hooks + aesthetics: collapsed debug panels
In early development we printed raw debug lines and it got messy.

Because we already use **Rich** in the CLI, we upgraded hook output to use Rich Panels:

- a single collapsed **Planning** panel per user turn
- a panel for each **Tool call** (syntax highlighted JSON)
- a panel for each **Tool result** (pretty printed)

This dramatically improves readability while keeping the core loop unchanged.

## 6) When to use hooks (practical checklist)
Use hooks when:

- you need observability
- you need optional features (debug UI, audit logs)
- you need extension points but want to keep the core stable

Avoid hooks when:

- you need to change control flow (use strategies/state machines)
- ordering and composition are the main point (use middleware)

## 7) Takeaway
Hooks are a simple but powerful pattern:

- expose event points
- keep the core loop clean
- make observability pluggable

In AgentDojo, hooks let us iterate quickly on debugging and trace aesthetics **without** making the agent loop brittle.

---

**Project:** AgentDojo (private org repo)

If you’re building your own agent, a good first hook surface is:
- tool call detected
- tool execution started
- tool execution finished
- planning started/finished
