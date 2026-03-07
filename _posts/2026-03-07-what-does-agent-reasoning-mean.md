---
layout: post
title: "What We Mean by \"Agent Reasoning\" (When an LLM Is Just Next-Token Prediction)"
date: 2026-03-07
categories: [agents, llms, evaluation]
tags: [agents, reasoning, react, agentops, evaluation, trajectories]
---

When people hear “agent reasoning,” a common (and reasonable) objection is:

> *An LLM is just next-token prediction. It’s not literally thinking. So what does “reasoning” mean here?*

This post gives a technical, non‑anthropomorphic answer. The short version is:

**In agent engineering, “reasoning” is a functional / behavioral term.** It refers to the *quality of the agent’s decision process* as reflected in its multi‑step behavior (tool choices, state updates, and stopping decisions). It is **not** a claim that the model is conscious or “thinking like a human.”

I’ll anchor the definitions below to Google’s *Startup technical guide: AI agents* (64pp) and quote it directly where it defines evaluation targets.

---

## The apparent contradiction: next-token prediction vs “reasoning”

At the mechanism level, you’re right: an LLM produces tokens by learning a conditional distribution and decoding from it. Nothing in the objective says “be conscious” or “reason like a human.”

But at the system level, a next-token model can still implement useful computation when embedded in an agent loop. It can represent intermediate structure (plans, decompositions, constraints), choose actions (tools) based on context, and update subsequent actions based on observations.

So when practitioners say an agent “reasoned,” what they usually mean is: **the agent followed a goal‑directed procedure that tends to be coherent, efficient, and correct.**

---

## What the Google guide calls “reasoning correctness”

Google’s guide is explicit that evaluating agents requires judging not only the final answer but also the process. It distinguishes:

> "Traditional testing often focuses on lexical correctness, but agent evaluation must address two harder problems: **semantic correctness** (did the agent understand and helpfully answer the user’s intent?) and **reasoning correctness** (did the agent follow a logical and efficient path to its conclusion?)." (pp. 50–51)

That “logical and efficient path” is what practitioners usually call the **trajectory**.

The guide defines a trajectory as:

> "A ‘trajectory’ is the full sequence of Reason, Act, and Observe steps the agent takes to complete a task." (p. 51)

So, in practice, *agent reasoning* is best understood as:

> **the policy that generates the trajectory** (what it does next, what tool it calls, how it uses observations).

---

## ReAct is a control loop (not a mind)

The ReAct pattern is described as a loop that interleaves intermediate “reasoning traces” and actions:

> "ReAct establishes a dynamic, multi-turn loop where the model generates both **reasoning traces (thoughts)** and **task-specific actions** in an interleaved manner." (p. 14)

And it specifies the control loop:

> "1. Reason: The agent assesses the goal and the current state…\n2. Act: The agent selects and invokes the appropriate tool.\n3. Observe: The agent receives the output from the tool…" (p. 14)

The important point: calling the first step “Reason” does not imply a mind is thinking. It means **the system is computing the next action based on goal + state**.

---

## What “reasoning correctness” looks like in concrete terms

When we say an agent “reasoned well,” we usually mean it did things like:

- chose the right tool for the subproblem
- produced valid, sufficient tool arguments
- incorporated observations instead of ignoring them
- made progress and stopped at the right time
- stayed within constraints (policies/permissions)

Google’s guide makes this testable by breaking trajectory evaluation into the ReAct steps:

> "Reason step: Does the agent correctly assess the goal and current state…?\nAct step: Does it select the correct tool… and correctly extract and format the arguments…?\nObserve step: Does it correctly integrate the output…?" (p. 51)

These are operational criteria. They’re not philosophical claims.

---

## A note on “chain-of-thought”

Many systems log natural-language “thought traces.” They can help debugging, but they are not ground truth.

A pragmatic stance is to treat:

- **tool calls + tool I/O + observable actions** as the primary evidence of “reasoning,” and
- natural-language “thoughts” as a *debug artifact* that may be incomplete or misleading.

---

## Why this matters (AgentOps)

If “reasoning” is behavioral, then improving reasoning is mostly about engineering:

- better tool contracts
- better grounding and evidence gathering
- better state handling
- better policies / constraints
- better evaluation harnesses

Google’s guide is blunt that the operational problem is non-determinism:

> "Moving beyond superficial 'vibe-testing' requires a rigorous engineering approach…" (p. 48)

That’s why it calls trajectory evaluation “the most critical layer”:

> "This is the most critical layer for evaluating the agent’s reasoning process." (p. 51)

---

## Takeaway

When an agent engineer says “reasoning,” the safest interpretation is:

> **Reasoning = the agent’s goal-directed decision process as evidenced by its action trajectory (tool choices + observation integration), not human-like conscious thought.**

You can evaluate and improve trajectories to make agents more reliable—without making any claims about whether a model “thinks.”

---

## References

- Google, *Startup technical guide: AI agents* (primary): <https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>
- ReAct: <https://arxiv.org/abs/2210.03629>
