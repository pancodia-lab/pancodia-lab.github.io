---
layout: post
title: "I Read Google’s 64-Page AI Agents Guide — Here’s the Product-Neutral Playbook (and What We’re Stealing)"
date: 2026-03-05
categories: [agents, architecture, agentops]
tags: [agents, react, rag, evaluation, agentops, reliability, planning]
---

Google published a **64-page** “Startup technical guide” on AI agents.

It’s a good read — but it’s also long, and (understandably) it’s written through a Google Cloud lens.

So I did two things:

1) I distilled the guide into a **product-neutral playbook** you can apply whether you’re building on GCP, AWS, or a Raspberry Pi.
2) I mapped the best ideas into our ongoing project, **AgentDojo** (a from-scratch, spec-driven, security-first agent build).

This post is the “Medium version.” No product catalog. Just the architecture and operations ideas that matter.

---

## The real shift: from “chatbot” to “agent”

An agent is not just “LLM + prompt.”

An agent is a system that:
- takes a goal,
- **chooses actions** (tools),
- **observes outcomes**,
- and iterates until it reaches a result.

Once you accept that, you stop obsessing over the prompt and start obsessing over the *system*: tool contracts, state, evaluation, and safety.

---

## The 4 parts every agent system has

The guide frames agents as four components. It’s a clean mental model and it matches what we’ve seen building AgentDojo.

### 1) Model (the brain)
The most useful advice here is counterintuitive:

> The best model is rarely “the strongest model.” It’s the leanest model that reliably works.

In practice, capability fights cost and latency. You’ll want a “big brain” reserved for the truly hard moments, and a cheaper one for routine turns.

### 2) Tools (the hands)
Tools are how agents stop hallucinating and start *doing*.

A tool can be:
- a local function (e.g., `read_file`)
- an API call
- a database query
- even another agent

The key lesson: a tool is not “just code.” It’s an **API contract for the model**.

Which means:
- descriptive names
- clear parameter schema
- docstring/spec that explains when to use it
- structured return values (ideally with `status: success|error`)

If tool contracts are vague, the model’s behavior will be vague.

### 3) Orchestration (the executive function)
Orchestration is just: *how do we decide what to do next?*

The dominant loop is **ReAct**:

1. Reason (decide next step)
2. Act (call tool)
3. Observe (feed result back)

Repeat until done.

If you’ve ever built an agent loop, you’ve basically built ReAct — even if you didn’t call it that.

### 4) Runtime (the body)
This is the boring part that becomes the most important part:

- scalability
- security boundaries
- logging
- monitoring
- error handling

If the runtime is weak, your agent is a demo, not a product.

---

## Grounding: how agents earn trust

“Grounding” is the umbrella term for making outputs verifiable.

The guide lays out a progression that’s worth internalizing:

### RAG (Retrieval-Augmented Generation)
Retrieve relevant text from a knowledge base → put it into context → answer.

This is the baseline for “don’t hallucinate.”

### GraphRAG
Same idea, but the knowledge isn’t just “documents.” It’s a graph of relationships.

This helps when questions require structure: cause/effect, dependencies, multi-hop reasoning.

### Agentic RAG
Now retrieval itself becomes multi-step:

- the agent decides what to search
- runs multiple retrieval steps
- synthesizes from the best evidence

This is where agents feel powerful — and where complexity spikes.

---

## Memory: three layers, not one

One of the best parts of the guide is its memory taxonomy:

1) **Working memory**: what’s in the current session (chat history)
2) **Long-term memory**: persistent knowledge / preferences (often via RAG)
3) **Transactional memory**: the system-of-record for actions (audit trail)

That third one matters a lot.

Most toy agents have “history.” Few have a durable, queryable record of **what actions were taken**, with integrity.

If you ever need compliance, incident response, or “why did the agent do that?”—transactional memory is the missing piece.

---

## The operational heart: AgentOps (stop vibe-testing)

The guide’s strongest point is not architecture. It’s operations.

It argues that “it worked once in my terminal” is meaningless for non-deterministic systems.

Instead, you need **AgentOps**: DevOps/MLOps ideas applied to agent loops.

### The 3-layer evaluation stack
This is the framework we’re most likely to directly adopt.

**Layer 1: Component tests**
- deterministic unit tests for tools, parsers, config

**Layer 2: Trajectory tests (the big one)**
- test the *procedure*, not just the answer
- did the agent pick the right tool?
- did it generate correct arguments?
- did it incorporate the observation?

**Layer 3: Outcome tests**
- quality of the final answer: factuality, helpfulness, safety

Trajectory testing is how you prevent your agent from quietly degrading over time.

---

## What we’re stealing for AgentDojo (concretely)

We’re not going to copy the product stack. We’re going to copy the discipline.

Here are the three upgrades we’ll implement next:

1) **Trajectory evaluation as a first-class concept**
   - A golden set of prompts
   - Expected tool-call sequences
   - A regression gate (eventually CI)

2) **Transactional memory (audit trail)**
   - Serialize hook events (tool calls/results) to JSONL
   - Later: route to durable storage if needed

3) **Tool contract tightening**
   - consistent `status` fields in tool returns
   - clearer specs/docstrings
   - avoid overlapping tool purposes

---

## Closing thought

If you remember one thing from the whole guide, let it be this:

> Agents are not a prompt. Agents are an operational system.

The moment you treat tool calls, state, evaluation, and auditing as first-class concerns, your “agent demo” starts to look like an “agent product.”

---

**Further reading (foundational papers):**
- ReAct: https://arxiv.org/abs/2210.03629
- Plan-and-Solve: https://arxiv.org/abs/2305.04091
- Tree of Thoughts: https://arxiv.org/abs/2305.10601
- Reflexion: https://arxiv.org/abs/2303.11366
