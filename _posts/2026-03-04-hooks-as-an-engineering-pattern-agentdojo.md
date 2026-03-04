---
layout: post
title: "Our Debug Output Was a Mess — Hooks Fixed It Without Touching the Core Loop"
date: 2026-03-04
categories: [engineering, architecture, agents]
tags: [hooks, design-patterns, observability, agentdojo, python]
---

We're building an LLM agent from scratch — not using a framework, not wrapping LangChain, just Python and curiosity. The project is called **AgentDojo**, and the goal is to understand every layer by building it ourselves: tool execution, planning, memory, safety.

A few weeks in, we hit a familiar problem: **our debug output was unreadable.**

## The mess

Our agent has an inner loop that chains multiple tool calls in one turn. The model says "read this file," we execute it, feed back the result, the model says "now read that file," and so on until it produces a final answer.

During development, we needed to see what was happening: which tool got called, what the model returned, whether the planner triggered, what scores it assigned to candidate plans.

So we did what everyone does first: we added `print()` statements.

```python
# in main.py, scattered everywhere
print(f"[debug] Tool call: {tool_name} args={args}")
print(f"[debug] Result: {result}")
print(f"[debug] Planner triggered: tot_lite")
print(f"[debug] Plans proposed: 3, filtered: 3")
```

Within a day, the terminal output looked like someone had dumped a log file into a blender. Debug lines were interleaved with agent responses, tool results were unformatted walls of text, and half the prints were guarded by `if DEBUG:` checks that we kept forgetting to add.

We needed a better pattern.

## The insight: hooks are just "tell me when something happens"

If you've used Git hooks, React hooks, or WordPress hooks, you already know the core idea — even if those implementations look very different from each other.

A **hook** is a callback point that the core system exposes. The core system decides **when** to call it. The hook implementation decides **what** to do.

That's it. No magic.

The important thing hooks do is **separate concerns**: the main logic stays clean, and observability (or any side behavior) lives somewhere else.

### How hooks differ from similar patterns

This tripped us up initially, so it's worth clarifying:

- **Hook**: "I will notify you when X happens." The core system owns the timeline. You just react.
- **Middleware**: "I will intercept your request/response and can modify it." More invasive — the middleware is *in* the data path.
- **Plugin**: a packaged unit of functionality, often built *on top of* hooks.
- **Strategy**: "I will delegate a *decision* to you." The core system uses your return value.

For debug output, hooks are the right fit: we want to observe without changing behavior.

## What we built

We created a small hook interface in `src/hooks.py`:

```python
class AgentHooks(Protocol):
    def on_tool_call_detected(self, *, raw_text, tool, args) -> None: ...
    def on_tool_result(self, *, tool, result) -> None: ...
    def on_planning_log(self, *, text) -> None: ...
    def on_planning_flush(self) -> None: ...
```

And two implementations:

- **`NullHooks`**: does nothing. Used when debug is off. Zero overhead, zero noise.
- **`DebugToolCallsHooks`**: pretty-prints everything using Rich panels.

The agent loop in `main.py` calls hooks at specific moments:

```python
# After parsing a tool call
hooks.on_tool_call_detected(raw_text=response, tool=name, args=args)

# After executing the tool
hooks.on_tool_result(tool=name, result=result)

# During planning (buffered, printed as one block)
hooks.on_planning_log(text="ToTLitePlanner: chose plan 0")
hooks.on_planning_flush()  # → renders a single "Planning" panel
```

The key property: **remove all hooks, and the agent behaves identically.** Hooks are observation-only.

## The aesthetic upgrade: collapsed panels

With hooks in place, improving the output format became a localized change. We switched from inline `print()` to Rich panels:

- One collapsed **"Planning"** panel per user turn (all planning logs buffered, then rendered together)
- A **"Tool call → read_file"** panel with syntax-highlighted JSON
- A **"Tool result ← read_file"** panel with pretty-printed output

The terminal went from a noisy stream of debug lines to something that actually helped us think.

And crucially: we made this change by editing *one file* (`hooks.py`). The core loop didn't change at all.

## A rule we learned: hooks should not change state

Early on, we considered letting hooks influence behavior — maybe a hook could skip a tool call, or modify the observation before it goes back to the LLM.

We decided against it. Here's why:

Once hooks can change state, you lose the ability to reason about behavior from the core loop alone. Debugging becomes:

> "Is this happening because of the agent logic, or because of something a hook did?"

If you need behavior changes, use a different mechanism: a strategy pattern (for decisions), a middleware chain (for request/response modification), or a policy engine (for rules). Keep hooks side-effect-only.

## When to use hooks (and when not to)

**Good fit:**
- Logging, tracing, metrics
- Debug UIs
- Audit trails
- Optional notifications

**Not a good fit:**
- Changing control flow → use strategies or state machines
- Intercepting and modifying data → use middleware
- Core business logic → just put it in the core

## What we'd do differently

If we were starting over, we'd define the hook surface *before* writing the first `print()`. It's much easier to add hooks early than to extract them from scattered debug code later.

We'd also add a `on_turn_start` / `on_turn_end` pair from the beginning — it makes it trivial to measure turn duration, count tool calls per turn, and build session summaries.

## Takeaway

Hooks are one of those patterns that feel too simple to be worth naming. But naming it — and committing to the constraint that hooks don't change behavior — gave us a clean separation between "what the agent does" and "how we observe it."

If you're building an agent (or any system with a main loop and optional observability), hooks are a great first architectural decision.

---

*This post is part of the [AgentDojo](https://github.com/pancodia-lab/agent-dojo) series — building an LLM agent from zero to hero, grounded in research and engineering discipline.*
