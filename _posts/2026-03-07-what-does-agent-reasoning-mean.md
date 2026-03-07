---
layout: post
title: "What We Mean by \"Agent Reasoning\" (When an LLM Is Just Next-Token Prediction)"
date: 2026-03-07
categories: [agents, llms, evaluation]
tags: [agents, reasoning, react, agentops, evaluation, trajectories]
---

When people hear "agent reasoning," the first reaction is often the same objection (and it's a fair one):

> *An LLM is just next-token prediction. It's not literally thinking. So what does "reasoning" mean here?*

If you're allergic to anthropomorphism, good—you should be. In agent engineering, "reasoning" is not a claim about consciousness. It's a **functional / behavioral term**. It means: *does the system follow a goal-directed decision process that tends to be coherent, efficient, and correct*, as evidenced by what it actually does over multiple steps (tool calls, state updates, and when it decides to stop).

This framing is consistent with Google's *Startup technical guide: AI agents* (64pp), which describes what we're trying to evaluate when we say an agent "reasoned correctly."

### Mechanism vs. behavior

At the mechanism level, you're right: an LLM produces tokens by learning a conditional distribution and decoding from it. Nothing in the objective says "be conscious" or "reason like a human."

But once you embed the model inside an agent loop, the *system* can implement useful computation. A next-token model can emit intermediate structure (plans, constraints, decompositions), choose actions (tools) based on context, and update future actions based on observations. The words "reason" and "think" are just labels we use for that decision process.

The guide is explicit that evaluation must go beyond the final answer. It distinguishes:

> "Traditional testing often focuses on lexical correctness, but agent evaluation must address two harder problems: **semantic correctness** (did the agent understand and helpfully answer the user's intent?) and **reasoning correctness** (did the agent follow a logical and efficient path to its conclusion?)." (pp. 50–51)

That "logical and efficient path" is what most practitioners mean by an agent's **trajectory**:

> "A 'trajectory' is the full sequence of Reason, Act, and Observe steps the agent takes to complete a task." (p. 51)

So in practice, "agent reasoning" is best understood as **the policy that generates the trajectory**: what it does next, which tool it calls, what arguments it passes, and how it incorporates what comes back.

### ReAct is a control loop (not a mind)

The ReAct pattern is a convenient way to name this closed loop:

> "ReAct establishes a dynamic, multi-turn loop where the model generates both **reasoning traces (thoughts)** and **task-specific actions** in an interleaved manner." (p. 14)

And the loop itself is straightforward control logic:

> **Reason:** The agent assesses the goal and the current state…  
> **Act:** The agent selects and invokes the appropriate tool.  
> **Observe:** The agent receives the output from the tool… (p. 14)

Calling step (1) "Reason" does not imply there's a mind thinking. It means the system computes the next action from goal + state.

![Trajectory evaluation loop for agent reasoning](/assets/img/trajectory-evaluation-loop.svg)
*Figure: ReAct trajectory with evaluation checkpoints at each step (Reason, Act, Observe) and explicit stop/iterate decision.*

One thing worth noting here: natural-language "thought traces" (chain-of-thought text) can appear in logs and look like evidence of "thinking." They *can* help debugging, but **they're not ground truth**. The more reliable signal is the **observable trajectory** — tool calls, tool I/O, and resulting state transitions. **Natural-language thoughts are a debug artifact that can be incomplete or misleading.**

### A concrete example

Suppose a user asks for a refund. A weak agent might skip straight to a plausible-sounding answer. A stronger agent will produce a trajectory that looks more like a procedure: retrieve the refund policy, check the user's order date, decide eligibility, call the refund tool, confirm success, and stop.

Whether the agent is "reasoning well" comes down to whether those steps happen correctly—especially the tool selection and the way observations are carried forward into the next step.

The guide makes this testable by breaking trajectory evaluation into the three ReAct steps:

> "Reason step: Does the agent correctly assess the goal and current state…?  
> Act step: Does it select the correct tool… and correctly extract and format the arguments…?  
> Observe step: Does it correctly integrate the output…?" (p. 51)

Those are operational criteria. They're not philosophical claims.

### Why this matters: AgentOps

If "reasoning" is behavioral, improving it is mostly an engineering problem: better tool contracts, better grounding, better state handling, better constraints, and better evaluation harnesses. This is why the guide is blunt about avoiding "vibes":

> "Moving beyond superficial 'vibe-testing' requires a rigorous engineering approach…" (p. 48)

And why it calls trajectory evaluation the core lever:

> "This is the most critical layer for evaluating the agent's reasoning process." (p. 51)

### Takeaway

When an agent engineer says "reasoning," the safest interpretation is:

> **Reasoning = the agent's goal-directed decision process as evidenced by its action trajectory (tool choices + observation integration), not human-like conscious thought.**

You can evaluate and improve trajectories to make agents more reliable—without making any claims about whether a model "thinks."

---

### References

- Google, *Startup technical guide: AI agents* (primary): <https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>
- ReAct: <https://arxiv.org/abs/2210.03629>

### Further reading (methodology sources)

- WebArena (realistic web-agent benchmarks and trajectory-grounded evaluation): <https://arxiv.org/abs/2307.13854>
- SWE-bench (end-to-end software task evaluation for coding agents): <https://arxiv.org/abs/2310.06770>
- G-Eval (LLM-as-judge with structured criteria): <https://arxiv.org/abs/2303.16634>
- MT-Bench / LLM-as-a-judge study: <https://arxiv.org/abs/2306.05685>
- Toolformer (tool-use decision behavior): <https://arxiv.org/abs/2302.04761>
- Reflexion (self-correction loops for improved reliability): <https://arxiv.org/abs/2303.11366>
- OpenAI Evals guide (practical eval pipeline patterns): <https://platform.openai.com/docs/guides/evals>
- LangSmith evaluation docs (trace + dataset + evaluator workflow): <https://docs.smith.langchain.com/evaluation>
