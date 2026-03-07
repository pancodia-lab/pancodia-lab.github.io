---
layout: post
title: "What We Mean by \"Agent Reasoning\" (When an LLM Is Just Next-Token Prediction)"
date: 2026-03-07
categories: [agents, llms, evaluation]
tags: [agents, reasoning, react, agentops, evaluation, trajectories]
---

When people hear "agent reasoning," a common (reasonable) objection is:

> *An LLM is just next-token prediction. It’s not literally thinking. So what does “reasoning” mean here?*

This post gives a technical, non-anthropomorphic answer. The short version: **"reasoning" is a functional / behavioral term**—it refers to the *quality of the agent’s decision process*, as observed through its multi-step behavior (tool choices, state updates, and stopping decisions), not to human-like conscious thought.

The framing here is aligned with Google’s *Startup technical guide: AI agents* (64pp). I’ll quote it directly where it defines evaluation targets.

---

## 1) Two levels of description can both be true

### Mechanistic description

Yes: an LLM produces tokens by learning a conditional distribution and sampling/decoding from it. Nothing in the training objective says “be conscious” or “reason like a human.”

### Functional description

But **a next-token model can still implement useful computation** when embedded in a system:

- It can represent intermediate steps (plans, decompositions, constraints)
- It can select actions (tools) based on context
- It can update its behavior based on observations

So in agent engineering, "reasoning" typically means: **the agent can follow a goal-directed procedure that tends to be coherent, efficient, and correct**.

---

## 2) In agent systems, “reasoning” means “trajectory quality”

Google’s guide is explicit that evaluating agents requires judging not only the final answer but also the process. It distinguishes:

> "Traditional testing often focuses on lexical correctness, but agent evaluation must address two harder problems: **semantic correctness** (did the agent understand and helpfully answer the user’s intent?) and **reasoning correctness** (did the agent follow a logical and efficient path to its conclusion?)." (pp. 50–51)

That “logical and efficient path” is what practitioners usually call the **trajectory**.

The guide defines a trajectory as:

> "A ‘trajectory’ is the full sequence of Reason, Act, and Observe steps the agent takes to complete a task." (p. 51)

So, in practice, **agent reasoning = the policy that generates the trajectory**.

---

## 3) Reason → Act → Observe is a control loop (not a mind)

The ReAct pattern is described as a loop that interleaves intermediate reasoning traces and actions:

> "ReAct establishes a dynamic, multi-turn loop where the model generates both **reasoning traces (thoughts)** and **task-specific actions** in an interleaved manner." (p. 14)

And it specifies the control loop:

> "1. Reason: The agent assesses the goal and the current state…\n2. Act: The agent selects and invokes the appropriate tool.\n3. Observe: The agent receives the output from the tool…" (p. 14)

This is important: calling the first step “Reason” doesn’t mean “a mind is thinking.” It means **the system is computing a next action based on goal + state**.

---

## 4) What “reasoning correctness” looks like in concrete terms

When we say an agent “reasoned well,” we usually mean it did things like:

- **Tool selection correctness**: chose the right tool for the subproblem
- **Parameter generation correctness**: passed valid, sufficient arguments
- **Observation integration**: used tool outputs in the next step instead of ignoring them
- **Progress / termination**: made measurable progress and stopped at the right time
- **Constraint adherence**: stayed within policies/permissions and avoided unsafe actions

Google’s guide makes this testable by breaking trajectory evaluation into the ReAct steps:

> "Reason step: Does the agent correctly assess the goal and current state…?\nAct step: Does it select the correct tool… and correctly extract and format the arguments…?\nObserve step: Does it correctly integrate the output…?" (p. 51)

These are operational criteria. They’re not philosophical claims.

---

## 5) A note on “chain-of-thought”

You’ll often see systems log a “thought trace” or “chain-of-thought.” That can be useful for debugging, but it’s not ground truth.

A pragmatic posture is:

- Treat the **trajectory and tool I/O** as the primary evidence of reasoning
- Treat natural-language “thoughts” as a *debug artifact* that may be incomplete or misleading

This is one reason why agent observability focuses on events, tool calls, and traces.

---

## 6) Why this matters: evaluation and AgentOps

If “reasoning” is behavioral, then improving reasoning is mostly about:

- better tool contracts
- better grounding and evidence gathering
- better state handling
- better policies / constraints
- and better evaluation harnesses

Google’s guide is blunt that the operational problem is non-determinism, and you need more than vibes:

> "Moving beyond superficial 'vibe-testing' requires a rigorous engineering approach…" (p. 48)

That’s why it calls trajectory evaluation “the most critical layer”:

> "This is the most critical layer for evaluating the agent’s reasoning process." (p. 51)

---

## Takeaway

When an agent engineer says “reasoning,” the safest interpretation is:

> **Reasoning = the agent’s goal-directed decision process as evidenced by its action trajectory (tool choices + observation integration), not human-like conscious thought.**

If you can evaluate and improve trajectories, you can make agents more reliable—without making any claims about whether the model “thinks.”

---

## References

- Google, *Startup technical guide: AI agents* (primary): <https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>
- ReAct: <https://arxiv.org/abs/2210.03629>
