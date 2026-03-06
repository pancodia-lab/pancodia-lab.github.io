---
layout: post
title: "Google's 64-Page AI Agents Guide — A Product-Neutral Technical Brief"
date: 2026-03-05
categories: [agents, architecture, agentops]
tags: [agents, react, rag, evaluation, agentops, reliability, planning, grounding, memory]
---

# AI Agents: A Technical Brief

**Primary source:** Google, *Startup technical guide: AI agents* (PDF, 64 pp)
<https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>

**Audience:** Technical founders, engineers, and applied scientists building production agent systems.

**Scope:** This brief distills the guide's three sections—core concepts, how to build, and how to operationalize—into a self‑contained research‑style summary. Google‑specific products are treated as concrete exemplars of broadly applicable architectural patterns.

---

## Abstract

Large language model (LLM)‑based agents are software systems that go beyond single‑turn question answering to execute multi‑step plans: they interpret goals, select and invoke tools, observe results, and iterate until a task is complete. The Google guide frames every agent as a composition of four primitives—**models, tools, orchestration, and runtime**—and argues that the central obstacle to production readiness is **non‑determinism**. Because agent behavior varies across runs, prompts, model versions, and tool environments, moving from prototype to production requires a discipline the guide calls **AgentOps**: systematic multi‑layer evaluation, full‑trace observability, governance, and defense‑in‑depth security.

This brief covers the guide's three arcs. Section 1 defines what agents are, decomposes them into architectural primitives, and presents grounding (RAG through agentic RAG) as the mechanism that makes agents trustworthy. Section 2 provides a practical toolkit‑oriented blueprint for building agents, including tool design, orchestration patterns, memory architecture, deployment, and inter‑agent protocols. Section 3 introduces AgentOps as an operational framework—a four‑layer evaluation methodology, continuous monitoring, and responsibility safeguards—for shipping agents safely at scale.

---

## 1. What is an AI agent?

### 1.1 Definition and significance

The guide defines an AI agent as a system powered by an LLM that can autonomously plan and execute multi‑step tasks and workflows—not merely answer questions. An agent interprets a goal, decides on a sequence of steps, calls external tools or APIs, incorporates the resulting observations, and iterates until completion. This closed‑loop behavior implies statefulness and real‑world action: even when the user sees only a final answer, the system may have executed a complex internal plan.

The guide positions this as a paradigm shift in software engineering. Traditional software deterministically executes pre‑specified logic. Agents introduce a fundamentally different contract: the LLM *decides* what to do next, which unlocks automation for workflows too variable or context‑dependent for rule‑based approaches—but also introduces new failure modes (unexpected tool calls, brittle parsing, unsafe side effects, prompt injection, upstream drift, and inconsistent outcomes).

### 1.2 Three engagement paths

Teams can engage with agents in three ways: **build** custom agents tailored to their product; **use** prebuilt agents for common tasks; or **partner** by integrating third‑party agents via open interoperability protocols (MCP, A2A). This framing recognizes that composition and integration are as important as building from scratch.

---

## 2. Core architecture: four primitives

### 2.1 Model ("the brain")

The guide stresses that model selection is about **efficiency, not raw power**. The optimal strategy is to select the leanest model whose capability meets the task's error tolerance. In multi‑agent systems, different sub‑agents can dynamically use different model tiers—heavyweight models for complex reasoning, lightweight models for routine classification or formatting. Reasoning tokens are presented as a configurable lever: allocating more reasoning effort trades latency and cost for accuracy.

A recurring distinction: **fine‑tuning is not grounding.** Fine‑tuning adapts a model's style and refines task‑specific knowledge; grounding connects the model to real‑time, verifiable data sources for factual accuracy.

### 2.2 Tools ("the hands")

Tools bridge the gap between an agent's reasoning and its ability to retrieve information or execute stateful operations. The guide categorizes tools as: internal functions (proprietary business logic), external APIs, data sources (databases, vector stores), and other agents (agent‑as‑tool). Critically, a tool's definition functions as an **API contract for the model**. Poor tool schemas—ambiguous names, missing type hints, unstructured returns—produce systematic tool misuse. Robust tool interfaces include:

- A descriptive name and precise docstring specifying when the tool should be called.
- Typed parameter schemas (required vs. optional, with Python type hints).
- A structured return dictionary including a `status` key, so the agent can reliably distinguish success from failure in its observation step.

For tools requiring persistent session state, a `ToolContext` object can be injected automatically. Tools can also be packaged into **Toolsets** (bundles of related capabilities) and shared across agents or ecosystems via MCP.

### 2.3 Orchestration ("the executive function")

Orchestration is the control policy that guides an agent through multi‑step tasks: which tool to call, in what sequence, and when to stop. The canonical pattern is **ReAct** (Reason + Act), which establishes a dynamic loop:

1. **Reason:** The agent assesses the goal and current context, forms a hypothesis about the next step, and determines whether a tool is needed.
2. **Act:** The agent selects and invokes the appropriate tool (or delegates to a sub‑agent).
3. **Observe:** The tool output is captured, integrated into context, and fed into the next reasoning step.

This loop repeats until a stopping condition is met. The guide distinguishes three agent architecture types that trade off flexibility against predictability:

- **LLM Agents:** Non‑deterministic; use an LLM for complex reasoning and dynamic decision‑making. The most common type.
- **Workflow Agents:** Deterministic orchestrators that execute sub‑agents in predefined patterns—sequential (fixed order, output piped forward), parallel (concurrent execution), or looping.
- **Custom Agents:** Fully custom execution logic beyond built‑in patterns.

A pragmatic production design often combines these: deterministic scaffolding (phases, constraints, approvals) wrapped around non‑deterministic reasoning steps.

### 2.4 Runtime ("the body")

The runtime is the execution environment that handles authentication, network boundaries, rate limiting, retries, scaling, logging, and deployment. The guide emphasizes that runtime concerns are not optional "later work"—they are the substrate that determines whether an agent can be safely deployed and diagnosed. Agents are exposed as standard web services (e.g., via FastAPI), containerized, and deployed to managed platforms.

---

## 3. Grounding: from retrieval to reasoning

### 3.1 Why grounding matters

An agent's credibility depends on providing accurate, verifiable answers. Ungrounded generation can be fluent but wrong; grounding reduces hallucination risk by tying agent outputs to external evidence. The guide presents grounding not as a luxury feature but as part of the control system that enables verification, provenance, and safer decision‑making.

### 3.2 A three‑stage progression

**RAG (Retrieval‑Augmented Generation)** is the foundational pattern: retrieve relevant chunks from a corpus via semantic search (vector embeddings) and condition the model on them. RAG provides timeliness (access to information beyond training cutoff), accuracy (reduces hallucinated outputs), and speed (vector embeddings enable fast semantic search at scale). However, RAG treats knowledge as a flat collection of disconnected facts.

**GraphRAG** augments retrieval with explicit relational structure via knowledge graphs, so the agent understands how concepts relate—not just matches similar phrases. This is valuable for multi‑hop queries where relevance depends on relationships (e.g., "symptoms → causes → treatments").

**Agentic RAG** is the most powerful approach. It transforms the agent from a passive recipient of retrieved data into an active, reasoning participant in the search for knowledge. Following the ReAct pattern, the agent can analyze complex queries, formulate multi‑step retrieval plans, execute multiple tool calls in sequence, and reason across heterogeneous evidence. Combined with multimodality (models that process text, images, and audio), agentic RAG enables perception plus reasoning across data types.

### 3.3 Tool‑mediated grounding

A concrete pattern: the agent uses semantic search (vector DB) to identify a product from a vague description, then triggers a function call (`check_inventory`) to fetch real‑time data from a live system. The combination of retrieval (knowing what's relevant) and action (performing real‑time operations) is what makes agents operationally useful.

### 3.4 Practical guidance

- Start with RAG and require citations for key claims.
- Use a **retrieve‑and‑re‑rank** approach: widen the recall aperture (retrieve a larger candidate set), then pass it to a re‑ranking step to improve precision.
- Use a **check grounding API** to verify whether answers are supported by retrieved evidence.
- Add GraphRAG only when queries genuinely require relational reasoning.
- Treat agentic RAG as an advanced capability that should ship only with strong observability and trajectory testing.

---

## 4. Memory architecture: three layers

The guide decomposes agent memory into three layers with distinct latency and integrity requirements:

**Long‑term knowledge base.** Persistent storage for grounding and retrieval: structured knowledge bases for RAG, persistent user interaction history (cross‑session personalization), and operational data lakes for raw material (conversation transcripts, workflow states).

**Working memory.** Transient session state that must be accessed at extremely low latency: conversation history, intermediate tool outputs, current task progress.

**Transactional memory (audit log).** A durable ledger for actions and state transitions with ACID guarantees: what was executed, with what parameters, when, with what result, and what downstream state changed.

This three‑layer separation is often missing in prototypes but becomes critical in production for forensic debugging, compliance, and replayability.

---

## 5. Building agents: practical blueprint

### 5.1 Tool design as an engineering discipline

The guide frames tool design as the highest‑leverage activity in agent development. A tool's definition serves as a clear, unambiguous API contract with four components: a descriptive function signature with type hints, a docstring that is the primary source of semantic information for the model, a structured return schema (dictionary with `status` key), and optional stateful context injection via `ToolContext`.

Tools are organized via Toolsets (bundles of related capabilities), and can be shared across ecosystems: MCP for tool interoperability, A2A for agent‑to‑agent communication, and wrapper integrations with LangChain and CrewAI.

### 5.2 Step‑by‑step agent definition

1. **Define the agent's job**: the scope of tasks it is responsible for.
2. **Specify tools**: what it can call, with clearly described interfaces.
3. **Provide instructions and examples**: the behavioral spec in natural language.
4. **Implement orchestration**: the ReAct loop or a workflow agent pattern.
5. **Add grounding** when correctness depends on external truth.
6. **Instrument**: logs, traces, tool call records.
7. **Evaluate and iterate** before exposing the agent to production workflows.

### 5.3 Interoperability protocols

**Model Context Protocol (MCP)** is an open standard for connecting LLMs with external data sources and tools—a "universal adapter." Agents can consume external tools (acting as MCP clients) or expose native tools (wrapping them in an MCP server).

**Agent2Agent (A2A)** is an open standard for inter‑agent collaboration regardless of framework. Key concepts: an **Agent Card** (JSON file advertising capabilities and endpoints for discovery), a task‑oriented interaction model, and modality‑agnostic communication (text, audio, video).

### 5.4 Deployment

Agents are deployment‑agnostic: logic is decoupled from serving infrastructure. The guide presents three deployment targets, each suited to different team maturity levels: a fully managed auto‑scaling engine (fastest to production), serverless container platforms (pay‑per‑use, handles traffic spikes), and Kubernetes (maximum control over networking, stateful workloads, accelerators).

### 5.5 Governing an agent fleet

As agent deployments scale, they become organizational infrastructure. The guide introduces managed approaches for cataloging, access control, lifecycle management, and observability across a fleet of agents—enabling non‑technical team members to build custom agents via no‑code interfaces while maintaining centralized governance.

---

## 6. AgentOps: making agents production‑ready

### 6.1 Why agents need a new operational discipline

Because agents are stochastic and act through tools, teams must be able to answer: What did the agent do? Why did it do it? What evidence did it use? Did it operate within policy? How do we detect regressions as models and prompts change? The guide argues that achieving production‑grade reliability requires moving beyond informal "vibe‑testing" to a systematic, automated, and reproducible framework.

### 6.2 A four‑layer evaluation framework

**Layer 1 — Component evaluation (deterministic unit tests).** Test predictable, non‑LLM components: tools with valid/invalid/edge‑case inputs, data processing (parsing, serialization), API integrations (success, error, timeout).

**Layer 2 — Trajectory evaluation (procedural correctness).** The guide calls this the *most critical* layer. A "trajectory" is the full sequence of Reason, Act, and Observe steps. Evaluate whether the agent correctly assesses goals and forms logical hypotheses (Reason), selects the correct tool with correct parameters (Act), and correctly integrates tool output for the next step (Observe). Implementation: instrument each step with distributed tracing; maintain "golden sets" of expected trajectories run on every pull request.

**Layer 3 — Outcome evaluation (semantic correctness).** Evaluate the final user‑facing response: factual accuracy/grounding, helpfulness/tone, completeness. Use programmatic grounding checks (anti‑hallucination), LLM‑as‑judge scoring, and human‑in‑the‑loop feedback.

**Layer 4 — System‑level monitoring (in‑production).** Evaluation does not stop at deployment. Monitor tool failure rates, user feedback scores, trajectory metrics (e.g., ReAct cycles per task), end‑to‑end latency, and resource consumption (CPU, memory). Use OpenTelemetry and structured logging piped to dashboards.

### 6.3 The AgentOps lifecycle

The guide advocates a five‑step workflow: bootstrap infrastructure (IaC), develop agent logic, commit and trigger CI/CD, run continuous evaluation against predefined test sets, and deploy automatically upon successful evaluation. The separation of concerns is explicit: agent application logic (orchestration, tools, LLM interactions) is distinct from operational infrastructure (CI/CD, observability, evaluation pipelines).

### 6.4 Responsible and secure agents

Agents must be designed from the ground up with safeguards. The guide identifies seven risk categories—incorrect behavior, misuse, overstated capabilities, bias amplification, inequality, unsafe deployment, and information hazards—and maps each to specific safeguards: safety attribute scoring, recitation checks, content moderation, terms of service, UI disclaimers, model evaluations, bias tooling, and responsible AI guides.

A recurring theme: responsible agents are built by **engineering constraints** into the system—permissions, interfaces, monitors, and policies—not merely by instructing the model to behave. Defense‑in‑depth includes least‑privilege tool access, secure defaults, constrained tool schemas, monitoring, auditability, and policies that limit unsafe actions.

---

## 7. Synthesis: a vendor‑neutral blueprint

Although anchored in Google's ecosystem, the guide's technical core distills to a transferable blueprint:

1. **Define the agent's scope** narrowly enough to be testable.
2. **Expose tools with explicit schemas** and robust error signaling (typed parameters, `status` key, precise docstrings).
3. **Ground the agent** when correctness matters—start with RAG, progress to GraphRAG or agentic RAG as query complexity demands.
4. **Separate memory layers**: working memory (low‑latency session state), long‑term memory (persistent retrieval), and transactional memory (ACID audit log).
5. **Implement orchestration** via ReAct or deterministic workflow patterns, or combine both.
6. **Instrument everything**: tool calls, intermediate reasoning, outcomes, and resource consumption.
7. **Adopt AgentOps**: four‑layer evaluation (component → trajectory → outcome → system monitoring), CI/CD gates, and continuous in‑production evaluation.
8. **Ship with security controls**: least privilege, sandboxing, auditing, and structured safeguards against bias, misinformation, and unsafe actions.

The guide's contribution is making the intersection of ML, software architecture, and operations explicit—and providing a roadmap from "first agent" to "safe and scalable agent workforce."

---

## References

- Google, *Startup technical guide: AI agents* (primary source): <https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>
- ReAct (Yao et al., ICLR 2023): <https://arxiv.org/abs/2210.03629>
- Plan‑and‑Solve: <https://arxiv.org/abs/2305.04091>
- Tree of Thoughts: <https://arxiv.org/abs/2305.10601>
- Reflexion: <https://arxiv.org/abs/2303.11366>
- Model Context Protocol (MCP): <https://modelcontextprotocol.io>
- Agent2Agent Protocol (A2A): <https://google.github.io/A2A>
