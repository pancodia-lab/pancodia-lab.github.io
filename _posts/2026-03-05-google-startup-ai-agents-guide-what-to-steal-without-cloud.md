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

**Scope:** This brief distills the guide's three sections—core concepts, how to build, and how to operationalize—into a self‑contained research‑style summary. Google‑specific products are treated as concrete exemplars of broadly applicable architectural patterns. All direct quotations are from the original PDF with page references.

---

## Abstract

Large language model (LLM)‑based agents are software systems that go beyond single‑turn question answering to execute multi‑step plans: they interpret goals, select and invoke tools, observe results, and iterate until a task is complete. The Google guide frames every agent as a composition of four primitives—**models, tools, orchestration, and runtime**—and argues that the central obstacle to production readiness is **non‑determinism**. Because agent behavior varies across runs, prompts, model versions, and tool environments, moving from prototype to production requires a discipline the guide calls **AgentOps**: systematic multi‑layer evaluation, full‑trace observability, governance, and defense‑in‑depth security.

The guide opens by declaring that:

> "The development of AI agents represents a paradigm shift in software engineering, enabling startups to automate complex workflows, create novel user experiences, and solve business problems that were previously technically infeasible." (p. 1)

And it immediately foregrounds the core operational challenge:

> "But moving from a promising prototype to a production-ready agent means solving a new set of challenges. How do you manage their non-deterministic behavior? How do you verify their complex reasoning paths?" (p. 1)

This brief covers the guide's three arcs: (1) core concepts and grounding, (2) how to build agents, and (3) how to operationalize them reliably and responsibly.

---

## 1. What is an AI agent?

### 1.1 Definition and significance

The guide defines an AI agent as a system that goes beyond answering questions to orchestrate multi‑step tasks. Thomas Kurian (CEO of Google Cloud) frames the vision:

> "The agentive workflow is the next frontier. It's not just about asking a question and getting an answer. It's about giving AI a complex goal—like 'plan this product launch' or 'resolve this supply chain disruption'—and having it orchestrate the multi-step tasks needed to achieve it. This will fundamentally change productivity." (p. 4)

Similarly, Sundar Pichai offers a concise definition:

> "AI agents are systems that combine the intelligence of advanced AI models with access to tools so they can take actions on your behalf, under your control." (p. 9)

This closed‑loop behavior implies **statefulness** and real‑world action: even when the user sees only a final answer, the system may have executed a complex internal plan.

Statefulness here means that an agent must track and update information *across* the steps of a task—and potentially across sessions. A stateless chatbot treats each turn independently; an agent, by contrast, accumulates context as it works. Each Observe step feeds new information into the next Reason step, so the agent's decisions depend on the trajectory it has already taken. The guide makes this concrete in its data architecture (Section 1.2, pp. 12–13), where it separates three layers of state: **working memory** (transient session context like conversation history and intermediate tool outputs), **long‑term memory** (persistent knowledge and user preferences across sessions), and **transactional memory** (a durable audit log of actions taken, with ACID guarantees). The guide also highlights that ADK provides explicit support for this via context management: "A system that provides the agent with memory, allowing you to use the agent to recall user preferences and conversational history across multiple interactions to provide a coherent experience" (p. 5). At the tool level, statefulness is handled through `ToolContext`, which gives individual tools access to a "session-level state dictionary" (p. 33) so they can read and write persistent session state without breaking the tool's functional interface.

In short, statefulness is what turns a sequence of independent LLM calls into a coherent multi‑step process—and managing it correctly (with the right latency, durability, and integrity guarantees at each layer) is a core engineering challenge of agentic systems.

### 1.2 Three engagement paths

Teams can engage with agents in three ways, and the guide emphasizes that all three are underpinned by open interoperability:

> "Google Cloud supports the comprehensive development of agentic systems, whether you're building your own agents, using pre-built Google Cloud agents, or bringing in partner agents. Underpinned by the Model Context Protocol (MCP) and Agent2Agent (A2A) protocol, this common framework is designed for interoperability." (p. 4)

**Path 1: Build your own agents.** This path is for teams that need maximum control over agent behavior. The guide presents two sub‑options. The first is a **code‑first approach** (ADK), described as best "for developers, technical startups, and teams that require a high degree of control over agent behavior" (p. 5). ADK provides orchestration logic, tool definition and registration, context management, evaluation and observability, containerization, and multi‑agent composition (p. 5). The second is an **application‑first approach** (Agentspace), which "empowers non-technical team members to build custom agents using a no-code designer" (p. 6) for teams that need to scale agent usage across an organization without consuming engineering resources.

**Path 2: Use prebuilt agents.** For teams with limited engineering resources or standard use cases, prebuilt managed agents offer rapid prototyping: "managed agents let you focus on core business logic rather than managing infrastructure" (p. 6). The guide highlights Gemini Code Assist as an example — an AI‑powered assistant that "integrates into multiple points of the software development lifecycle, providing assistance through IDE extensions, a command-line interface, GitHub integration, and within various Google Cloud services" (p. 6). Gemini Cloud Assist is another example, providing "context-aware assistance for infrastructure management and application operations" (p. 7).

**Path 3: Bring in partner agents.** For specialized use cases, teams can "integrate third-party or open-source agents into your stack using Google Cloud's open ecosystem and via the Google Cloud Marketplace" (p. 8). The guide points to Agent Garden for deploying "pre-built ADK agents that already support data reasoning and inter-agent collaboration" (p. 8), noting that you can "mix and match them with the agents you build, speeding up time to impact" (p. 8). This path relies heavily on the MCP and A2A protocols for interoperability.

The key insight across all three paths is that agents are compositional: a production system may combine custom‑built agents (Path 1) with prebuilt agents (Path 2) and third‑party specialists (Path 3), all communicating via standardized protocols.

---

## 2. Core architecture: four primitives

### 2.1 Model ("the brain")

The guide stresses that model selection is about finding the right trade‑off — not simply choosing the strongest model:

> "Choosing the right model is not about selecting the most powerful one available, but about finding the optimal balance of **capability, speed, and cost** for your use case. Every model can be evaluated on these **three conflicting characteristics**, and the goal is to identify the most efficient option for a specific job." (p. 9)

> "The most common mistake is over-investing in capability when a use case doesn't need it, leading to inefficient spending and slower performance. The optimal strategy is to select the most efficient model for any given task." (p. 9)

At the system level, the guide advocates heterogeneous model assignment:

> "Robust cognitive architectures employ multiple specialized agents, each dynamically selecting the leanest model for its specific sub-task. This ensures, for instance, that a heavyweight model is reserved for complex reasoning, while a lightweight model handles routine queries. This multi-agent approach provides the architectural flexibility to optimize the cost and performance of the entire system, not just a single component." (p. 9)

*Note: The guide uses "cognitive architecture" informally to describe the structural design of an agent system — how its reasoning, memory, and tool‑use components are organized — rather than referencing a specific formal framework. For historical background on formal cognitive architectures, see ACT‑R (Anderson et al., 2004) and Soar (Laird et al., 1987). In context, "robust cognitive architectures" simply means well‑designed multi‑agent systems where each agent selects the right‑sized model for its sub‑task.*

Reasoning tokens are presented as a configurable lever:

> "By allocating more reasoning tokens to a specific call, a developer can direct the model to expend more computational effort, directly trading a predictable increase in latency and cost for a potential increase in accuracy." (p. 10)

*Practical interpretation:* In day‑to‑day engineering terms, this is a **compute budget knob** on a single response. Increasing the reasoning budget typically allows the model to spend more internal steps on planning/checking before it produces the final answer. In agentic settings, this often shows up as (a) more careful decomposition of the problem, (b) more consistent tool selection/parameter generation, and (c) fewer “jump to answer” failures—at the cost of higher latency and token spend. Even without an explicit knob, teams often approximate “more reasoning” by adding extra verification steps in the agent loop (retrieve → re‑rank → validate → answer), which similarly increases cost/latency to improve reliability.*

A critical distinction is drawn between fine‑tuning and grounding:

> "Fine-tuning is not grounding. Fine-tuning adapts a model's style and refines its knowledge on a specific task. Grounding connects the model to real-time, verifiable data sources to ensure its responses are factually accurate." (p. 11)

### 2.2 Tools ("the hands")

Tools bridge the gap between reasoning and action:

> "Tools are defined capabilities that enable an agent to do more than the native functions of its core reasoning model, from performing a simple, internal calculation to interacting with external systems via API calls. They bridge the gap between the agent's reasoning and its ability to retrieve new information or execute stateful operations." (p. 11)

The guide categorizes tools as: internal functions and services, APIs, data sources (databases, vector stores), and other agents. Tool design is framed as the highest‑leverage activity:

> "For a model to use a tool correctly, its definition must serve as a clear and unambiguous API contract." (p. 33)

The API contract has four components (pp. 33):
- **Function signature**: "Use descriptive names for tools and their parameters. Python type hints are mandatory, as they provide the structural schema the model uses to generate valid arguments."
- **Docstring**: "This is the primary source of semantic information for the model."
- **Return schema**: "A tool must return a dictionary. [...] it is a best practice to include a status key (e.g., success or error) in this dictionary. This structure is essential for the agent to reliably distinguish between successful outcomes and failures in its Observation step."
- **Stateful tools and ToolContext**: "For tools that need to read or write to a persistent session state, an optional tool_context: ToolContext parameter can be added to the function signature."

The guide also warns against ambiguity:

> "Each tool you define adds a new choice for the model to consider. Especially when an agent has many tools, any ambiguity or overlap in their descriptions can confuse the model, leading to looping behaviors or incorrect tool selection." (p. 41)

### 2.3 Orchestration ("the executive function")

Orchestration is defined as:

> "Orchestration is the operational core that guides an agent through a multi-step task. For any process that requires more than a single action, it determines which tools are needed, in what sequence, and how their outputs should be combined to achieve a final goal." (p. 14)

The canonical pattern is **ReAct** (Reason + Act), based on Yao et al. (ICLR 2023):

> "ReAct establishes a dynamic, multi-turn loop where the model generates both reasoning traces (thoughts) and task-specific actions in an interleaved manner. This allows for greater synergy—reasoning helps the model track and update action plans, while actions gather information from external tools to inform the reasoning process." (p. 14)

The ReAct loop operates in three steps (p. 14):
1. **Reason:** "The agent assesses the goal and the current state, forming a hypothesis about the next best step and whether a tool is required."
2. **Act:** "The agent selects and invokes the appropriate tool."
3. **Observe:** "The agent receives the output from the tool. This new information is integrated into the agent's context and feeds into the next Reason step of the cycle."

The guide distinguishes three agent architecture types (pp. 29–32):
- **LLM Agent** (non‑deterministic): "uses an LLM like Gemini for complex reasoning, dynamic decision-making, and natural language understanding"
- **Workflow Agents** (deterministic): Sequential, Parallel, and Loop agents that "deterministically control how other agents execute in predefined patterns"
- **Custom Agent**: for "unique requirements and tailored workflows that go beyond a standard reasoning loop"

The guide notes the importance of mastering orchestration:

> "Mastering orchestration is the key to moving beyond simple, single-shot agents. When you get it right, you create sophisticated, autonomous systems that can tackle problems that, previously, were not technically feasible." (p. 15)

### 2.4 Runtime ("the body")

The runtime handles production deployment concerns:

> "Deploying a functional agent prototype into a production environment requires a robust runtime infrastructure. The runtime facilitates agent deployment at scale, turning a prototype into a reliable product that handles complex operational requirements like security, load balancing, and error handling, especially during periods of unpredictable user growth." (p. 16)

Core runtime capabilities include scalability, security, and reliability/observability (p. 16).

---

## 3. Grounding: from retrieval to reasoning

### 3.1 Why grounding matters

> "An agent's credibility and usefulness depends on its ability to provide accurate, trustworthy answers based on verifiable facts, a process known as grounding." (p. 17)

### 3.2 A three‑stage progression

**RAG (Retrieval‑Augmented Generation)** is the foundational pattern:

> "This approach enhances an LLM's responses by retrieving relevant information from an external knowledge base before generating an answer. Instead of relying solely on its pre-trained knowledge, the agent performs a semantic search to find verifiable data, which is then passed to the LLM as context." (pp. 17–18)

The guide notes RAG's limitation:

> "While foundational, this simple retrieve-then-generate process treats knowledge as a flat collection of disconnected facts." (p. 18)

**GraphRAG** adds relational structure:

> "GraphRAG builds a knowledge graph, so instead of just matching similar phrases, your agent understands how concepts relate." (p. 19)

**Agentic RAG** is presented as the most powerful approach:

> "The most powerful approach to grounding is Agentic RAG, a technique that helps you transform the agent from a passive recipient of retrieved data into an active, reasoning participant in the search for knowledge. Following frameworks like ReAct, the agent can analyze a complex query, formulate a multi-step plan, and execute multiple tool calls in sequence to find the best possible information." (p. 20)

Douwe Kiela (CEO of Contextual AI) provides additional context:

> "The conventional wisdom was that foundation model performance would improve exponentially, but we are reaching the inflection point where that climb is plateauing and real differentiation lies in specialization and context engineering. Agentic RAG forms a central pillar of the context layer, allowing AI agents to iteratively find, retrieve, and reason over ground truth data before generating a final answer." (p. 20)

The strategic significance:

> "Agentic RAG represents a fundamental leap, moving beyond simple information retrieval to genuine problem-solving." (p. 22)

### 3.3 Practical guidance

The guide recommends a **retrieve‑and‑re‑rank** approach:

> "Address the trade-off between recall (finding all relevant documents) and precision (ensuring every retrieved document is relevant) using the two-step 'retrieve and re-rank' approach. First, it widens the recall aperture by configuring [the vector search] to retrieve a larger-than-needed set of candidate documents. Second, this larger set is passed to the LLM or a specialized re-ranking service, which identifies the most relevant documents and discards any that are irrelevant or semantically opposite." (p. 21)

The guide also recommends using a **check grounding API** to "verify whether the AI's answers are based on grounded, up-to-date info" (p. 18).

---

## 4. Memory architecture: three layers

The guide decomposes agent memory into three layers with distinct requirements (pp. 12–13):

**Long‑term knowledge base** — "An agent's long-term memory is the foundation for its intelligence, grounding, and personalization. [...] A robust long-term memory architecture should comprise three core components: a structured knowledge base for fact-based grounding via retrieval-augmented generation (RAG); a persistent store for user interaction history to enable a continuous, personalized experience; and, an operational data lake for raw material like conversation transcripts and workflow states." (p. 12)

**Working memory** — "This layer manages the transient information required for an ongoing task or conversation. It must provide extremely low-latency access to maintain a responsive user experience." (p. 13)

**Transactional memory** — "This layer is responsible for recording actions and state changes with strong consistency and integrity. It serves as the system of record, often requiring ACID guarantees to ensure reliability." (p. 13)

On the frontier of memory, the guide introduces **memory distillation**:

> "Memory distillation is the next frontier. It uses an LLM to dynamically and continuously distill long conversation histories into a compact, structured set of essential facts and preferences." (p. 35)

---

## 5. Building agents: practical blueprint

### 5.1 The trade‑off founders face

> "When it comes to building a custom AI agent for your startup, founders face a critical trade-off: development velocity versus flexibility." (p. 27)

### 5.2 Step‑by‑step agent definition (pp. 40–41)

The guide walks through defining an LLM agent:

1. **Define the agent's identity**: name, description, and model.
2. **Guide the agent with instructions**: "The instruction parameter is the most critical component for shaping an agent's behavior." (p. 40)
3. **Equip the agent with tools**: "The LLM uses the tool's name, docstring, and parameter schema to decide which tool to call." (p. 41)
4. **Complete the development lifecycle**: test, evaluate, and deploy.

A key warning on prompt engineering:

> "An LLM uses every part of an agent's definition, from its name and description to the names and descriptions of its tools, for reasoning. And it interprets this information with a high degree of literalism. Avoid ambiguous, unclear, or contradictory naming and descriptions, which can lead to 'context poisoning,' where the agent becomes confused, pursues incorrect goals, or fails to use its tools correctly." (p. 40)

### 5.3 Interoperability protocols

**Model Context Protocol (MCP)** — "MCP is an emerging open standard for connecting AI and LLMs with external data sources and tools. You can plug your AI applications into various data sources and tools without the hassle of building custom point-to-point integrations for each one." (p. 34)

**Agent2Agent (A2A)** — enables inter‑agent collaboration via **Agent Cards**: "A digital 'business card' (typically a JSON file at a well-known endpoint) that an agent uses to advertise its capabilities, endpoint URL, and authentication requirements, enabling discovery by other agents." (p. 38)

### 5.4 Deployment

> "ADK is deployment-agnostic by design. The core agent logic you define in Python is decoupled from the serving infrastructure so you can develop and test locally, then deploy the same agent to various production environments." (p. 36)

### 5.5 Governing an agent fleet

> "As your startup moves from building a single agent to deploying a portfolio of specialized agents, you face a new set of challenges: How do you manage them? How do non-technical team members leverage them? How do you govern their access to data and tools?" (p. 43)

---

## 6. AgentOps: making agents production‑ready

### 6.1 Why agents need a new operational discipline

> "Due to the non-deterministic nature of LLM-based systems, it can be hard to achieve production-grade reliability. Moving beyond superficial 'vibe-testing' requires a rigorous engineering approach to ensure an agent operates safely and provides consistent value." (p. 48)

The guide identifies three focus areas for this rigor (p. 48): **correctness and reliability** (accuracy of outputs and validity of intermediate reasoning), **performance and scalability** (latency and throughput under load), and **safety and responsibility** (safeguards, monitoring, and operating within defined boundaries).

Harrison Chase (CEO and Co-Founder of LangChain) frames the challenge:

> "Agents hold the key to a new level of productivity, but their success depends on our guidance." (p. 49)

**AgentOps** is defined as:

> "Agent Operations (AgentOps) is an operational methodology that addresses the challenges of reliability and responsibility in production. It adapts the principles of DevOps, MLOps, and DataOps to the unique challenges of building, deploying, and managing AI agents across their lifecycle. And it gives you a systematic, automated, and reproducible framework for handling the complexities of non-deterministic, LLM-based systems in production environments." (p. 50)

### 6.2 A four‑layer evaluation framework

The guide presents a rigorous multi‑layered evaluation approach. It notes:

> "Evaluating non-deterministic, agentic systems is one of the most complex challenges in modern software engineering. Traditional testing often focuses on lexical correctness, but agent evaluation must address two harder problems: semantic correctness (did the agent understand and helpfully answer the user's intent?) and reasoning correctness (did the agent follow a logical and efficient path to its conclusion?)." (pp. 50–51)

**Layer 1 — Component evaluation** (p. 50): "This layer focuses on the predictable, non-LLM components of the agent system." Tests tools with valid/invalid/edge-case inputs, data processing, and API integrations.

**Layer 2 — Trajectory evaluation** (p. 51): "This is the most critical layer for evaluating the agent's reasoning process. A 'trajectory' is the full sequence of Reason, Act, and Observe steps the agent takes to complete a task." Tests whether the agent correctly assesses goals (Reason), selects correct tools with correct parameters (Act), and integrates tool output (Observe).

**Layer 3 — Outcome evaluation** (p. 51): Evaluates the final user‑facing response for "factual accuracy and grounding," "helpfulness and tone," and "completeness."

**Layer 4 — System‑level monitoring** (p. 51): "Evaluation doesn't stop at deployment. Continuously monitoring the agent's live performance is critical." Monitors tool failure rates, user feedback scores, trajectory metrics, and end‑to‑end latency.

The guide concludes on evaluation:

> "Adopting a systematic evaluation framework is not merely a best practice but a competitive advantage. It establishes a rigorous, data-driven, and automated process that allows teams to innovate faster, deploy with confidence, and build agents that are demonstrably safer and more effective." (p. 51)

### 6.3 The AgentOps lifecycle

The guide advocates a clear separation of concerns:

> "ADK handles the agent's runtime behavior: As a Python/Java SDK, ADK provides the APIs and core abstractions for defining an agent's application logic. [...] The Agent Starter Pack handles the operational environment: As a scaffolding tool, it generates the infrastructure as code (Terraform) to provision deployment targets [...] and the CI/CD pipeline configurations (Cloud Build) to automate the entire lifecycle." (p. 53)

This separation manifests in a five‑step workflow (p. 53): bootstrap → develop → commit and automate → evaluate continuously → deploy confidently.

### 6.4 Responsible and secure agents

> "Building powerful agents comes with the non-negotiable responsibility of ensuring they are safe, secure, and aligned. This means designing them from the ground up with safeguards to prevent harmful or unintended outcomes, including unfair bias, privacy violations, and security vulnerabilities." (p. 54)

Jia Li (Co-Founder, President and Chief AI Officer of LiveX AI) notes:

> "As AI agents integrate into our lives, it's crucial for us to address new challenges around trust, privacy, and security." (p. 54)

The guide identifies seven risk categories (p. 54): not performing as intended, misapplication/harmful use, creating false impression of capabilities, amplifying negative societal biases, creating/worsening inequality, unsafe deployment, and information hazards—each mapped to specific safeguards.

The defense‑in‑depth strategy includes (p. 55): secure infrastructure and access control, input and output guardrails, and auditing and monitoring.

### 6.5 Observability as a foundation

The guide stresses that observability is not optional:

> "Production-grade observability means looking beyond application metrics. You must also measure low-level operational metrics like CPU and memory usage. Diligently tracking resource consumption is essential for diagnosing performance bottlenecks, optimizing your runtime, and directly reducing operational costs." (p. 49)

And that standard testing is insufficient for agentic systems:

> "Agentic systems are non-deterministic and can exhibit emergent behaviors. Standard unit tests are insufficient. Rigorous evaluation is the only way to ensure the quality and reliability of your agent." (p. 41)

---

## 7. Synthesis: a vendor‑neutral blueprint

The guide's conclusion frames the overall message:

> "The journey from prototype to a production-grade system is about disciplined engineering. By using a code-first framework like ADK and the operational principles in this guide, you can move beyond informal 'vibe-testing' to a rigorous, reliable process for building and managing your agent's entire lifecycle." (p. 59)

> "For your startup, this disciplined approach becomes a powerful competitive advantage. Your team can iterate and innovate faster, automating resource-intensive evaluations along the way. Plus, you can scale with confidence, without compromising on safety or security." (p. 59)

Although anchored in Google's ecosystem, the guide's technical core distills to a transferable blueprint:

1. **Define the agent's scope** narrowly enough to be testable.
2. **Expose tools with explicit schemas** and robust error signaling (typed parameters, `status` key, precise docstrings).
3. **Ground the agent** when correctness matters—start with RAG, progress to GraphRAG or agentic RAG as query complexity demands.
4. **Separate memory layers**: working memory (low‑latency session state), long‑term memory (persistent retrieval), and transactional memory (ACID audit log).
5. **Implement orchestration** via ReAct or deterministic workflow patterns, or combine both.
6. **Instrument everything**: tool calls, intermediate reasoning, outcomes, and resource consumption.
7. **Adopt AgentOps**: four‑layer evaluation (component → trajectory → outcome → system monitoring), CI/CD gates, and continuous in‑production evaluation.
8. **Ship with security controls**: least privilege, sandboxing, auditing, and structured safeguards against bias, misinformation, and unsafe actions.

---

## References

- Google, *Startup technical guide: AI agents* (primary source): <https://services.google.com/fh/files/misc/startup_technical_guide_ai_agents_final.pdf>
- ReAct (Yao et al., ICLR 2023): <https://arxiv.org/abs/2210.03629>
- Model Context Protocol (MCP): <https://modelcontextprotocol.io>
- Agent2Agent Protocol (A2A): <https://google.github.io/A2A>
- Anderson, J. R., Bothell, D., Byrne, M. D., Douglass, S., Lebiere, C., & Qin, Y. (2004). *An integrated theory of the mind.* **Psychological Review**, 111(4), 1036–1060. <https://doi.org/10.1037/0033-295X.111.4.1036>
- Laird, J. E., Newell, A., & Rosenbloom, P. S. (1987). *SOAR: An architecture for general intelligence.* **Artificial Intelligence**, 33(1), 1–64. <https://doi.org/10.1016/0004-3702(87)90050-6>
