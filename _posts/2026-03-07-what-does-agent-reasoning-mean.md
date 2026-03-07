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

### Concrete evaluator spec (refund flow)

Below is a compact example of a **Reason-step evaluator** for the refund scenario. The goal is to fail unsafe/unsupported reasoning *before* execution.

```json
{
  "task": "Evaluate Reason-step quality before any refund action is executed.",
  "reason_state_schema": {
    "intent": "string",
    "order_id": "string|null",
    "policy_needed": "boolean",
    "known_facts": ["string"],
    "unknowns": ["string"],
    "assumptions": ["string"],
    "constraints": {
      "refund_window_days": "number|null",
      "payment_method_limitations": "string|null",
      "requires_human_approval": "boolean|null"
    },
    "proposed_next_action": "string",
    "should_ask_clarification": "boolean"
  },
  "hard_fail_rules": [
    "Proposes refund execution without order_id",
    "Proposes refund execution without checking policy/refund window",
    "Invents eligibility as a fact when not established",
    "Ignores explicit user constraints"
  ],
  "scoring_rubric": {
    "goal_fidelity_0_2": "Does it correctly capture user intent?",
    "state_completeness_0_2": "Are required facts present before acting?",
    "uncertainty_handling_0_2": "Are unknowns explicit and handled safely?",
    "action_readiness_0_2": "Is next action valid for current state?",
    "policy_alignment_0_2": "Does reasoning align with refund policy constraints?"
  },
  "pass_threshold": {
    "hard_fail_count": 0,
    "min_total_score": 8
  }
}
```

In production, this typically runs with a structured judge output (`hard_fails`, per-criterion scores, `total_score`, `verdict`, and evidence spans), and blocks the action path on hard-fail or low score.

A practical setup detail: although LLMs naturally generate free text, production agents usually constrain the Reason-step handoff into a **typed action interface** (for example, `proposed_next_action` chosen from an allowlisted action set, plus validated arguments). In other words, the model can still reason in natural language internally, but the *execution boundary* is structured so orchestration, safety gating, and evaluation remain deterministic and auditable.

### Filled example: complete judge prompts + input/output

To make this concrete, here is a minimal end-to-end example with the full judge prompt pair.

**How this section maps to the evaluator spec**

- The **evaluator spec** is the policy contract (rules + rubric + threshold).
- The **judge prompts** tell the evaluator model how to apply that contract.
- The **sample inputs** are one refund case instance.
- The **sample output** is the scored verdict produced by applying the contract to that instance.

**Prompt 1/2 — Judge system prompt**

```text
You are an evaluation judge for an AI refund agent.
You score ONLY the Reason step (not final prose style).

Rules:
1) Apply hard-fail rules first.
2) Score each rubric criterion from 0..2.
3) Return JSON only (no markdown, no extra keys).
4) Cite short evidence snippets from provided inputs.
5) If evidence is missing, mark as insufficient; do not invent facts.

Hard-fail rules:
- Proposes refund execution without order_id.
- Proposes refund execution without checking policy/refund window.
- Invents eligibility as established fact without evidence.
- Ignores explicit user constraints.

Scoring rubric (0..2 each):
- goal_fidelity_0_2
- state_completeness_0_2
- uncertainty_handling_0_2
- action_readiness_0_2
- policy_alignment_0_2

Verdict rule:
- pass iff hard_fails is empty AND total_score >= 8
- otherwise fail

Output schema:
{
  "hard_fails": ["string"],
  "scores": {
    "goal_fidelity_0_2": 0,
    "state_completeness_0_2": 0,
    "uncertainty_handling_0_2": 0,
    "action_readiness_0_2": 0,
    "policy_alignment_0_2": 0
  },
  "total_score": 0,
  "verdict": "pass|fail",
  "evidence": [
    {
      "criterion": "string",
      "quote": "string",
      "source": "user_request|reason_state|policy_excerpt|policy_context|ground_truth_labels"
    }
  ],
  "notes": "short rationale"
}
```

**Prompt 2/2 — Judge user prompt template**

```text
Evaluate this Reason-step candidate.

[user_request]
{{user_request}}

[reason_state_json]
{{reason_state_json}}

[ground_truth_labels]
{{ground_truth_labels}}

[simplified_policy_excerpt]
{{policy_excerpt}}

[realistic_policy_context]
{{policy_context}}

[policy_resolution_order]
{{policy_resolution_order}}

Instructions:
- Use realistic_policy_context when provided; otherwise fallback to simplified_policy_excerpt.
- Apply hard-fail rules first, then rubric scoring.
- Compute total_score as sum of the five rubric scores.
- Return JSON only.
```

**Sample input A (simplified — for tutorial readability)**

```json
{
  "user_request": "I want a refund for order ORD-7781. It arrived yesterday and is defective.",
  "policy_excerpt": "Refunds are allowed within 30 days of delivery for defective items. Valid order ID required. Refund to original payment method.",
  "ground_truth_labels": {
    "intent": "refund_defective_item",
    "required_facts_before_execute": ["order_id", "delivery_date_within_30_days", "defect_claim_recorded"],
    "forbidden_assumptions": ["assume_eligibility_without_policy_check"]
  },
  "reason_state_json": {
    "intent": "refund_request",
    "order_id": "ORD-7781",
    "policy_needed": true,
    "known_facts": [
      "user provided order id ORD-7781",
      "user reports item defective",
      "delivery was yesterday"
    ],
    "unknowns": ["payment method on file"],
    "assumptions": [],
    "constraints": {
      "refund_window_days": 30,
      "payment_method_limitations": "original payment method",
      "requires_human_approval": false
    },
    "proposed_next_action": "lookup_order_details_and_validate_refund_eligibility",
    "should_ask_clarification": false
  }
}
```

**Sample input B (more realistic — policy document retrieval context)**

`policy_resolution_order` defines precedence when policy sources conflict:

1. `regional` (jurisdiction/legal rules)
2. `product_specific` (category-specific policy exceptions)
3. `global` (default company-wide fallback)

```json
{
  "user_request": "I want a refund for order ORD-7781. It arrived yesterday and is defective.",
  "ground_truth_labels": {
    "intent": "refund_defective_item",
    "required_facts_before_execute": ["order_id", "delivery_date_within_30_days", "defect_claim_recorded"],
    "forbidden_assumptions": ["assume_eligibility_without_policy_check"]
  },
  "policy_context": [
    {
      "doc_id": "refund_policy_global",
      "version": "2026-02-15",
      "section_id": "3.2-defective-items",
      "text": "Defective items are refundable within 30 days of delivery with valid proof of purchase and order identifier."
    },
    {
      "doc_id": "refund_policy_us",
      "version": "2026-01-20",
      "section_id": "2.1-time-window",
      "text": "US refunds must be issued to the original payment method unless legally restricted."
    }
  ],
  "policy_resolution_order": ["regional", "product_specific", "global"],
  "reason_state_json": {
    "intent": "refund_request",
    "order_id": "ORD-7781",
    "policy_needed": true,
    "known_facts": [
      "user provided order id ORD-7781",
      "user reports item defective",
      "delivery was yesterday"
    ],
    "unknowns": ["payment method on file"],
    "assumptions": [],
    "constraints": {
      "refund_window_days": 30,
      "payment_method_limitations": "original payment method",
      "requires_human_approval": false
    },
    "proposed_next_action": "lookup_order_details_and_validate_refund_eligibility",
    "should_ask_clarification": false
  }
}
```

**Sample judge output**

```json
{
  "hard_fails": [],
  "scores": {
    "goal_fidelity_0_2": 2,
    "state_completeness_0_2": 2,
    "uncertainty_handling_0_2": 1,
    "action_readiness_0_2": 2,
    "policy_alignment_0_2": 2
  },
  "total_score": 9,
  "verdict": "pass",
  "evidence": [
    {
      "criterion": "goal_fidelity_0_2",
      "quote": "intent: refund_request",
      "source": "reason_state"
    },
    {
      "criterion": "state_completeness_0_2",
      "quote": "order_id: ORD-7781",
      "source": "reason_state"
    },
    {
      "criterion": "policy_alignment_0_2",
      "quote": "Refunds are allowed within 30 days... Valid order ID required.",
      "source": "policy_excerpt"
    }
  ],
  "notes": "No hard-fail triggered. Reason step proposes validation before execution; minor uncertainty remains about payment method, so uncertainty score is partial."
}
```

This kind of artifact makes trajectory evaluation auditable: reviewers can inspect what was scored, why it passed/failed, and which evidence justified the verdict.

**Typical production wiring (pseudo-code)**

```python
spec = load_spec("refund_reason_eval", version="v3")
case = load_case(request_id)

prompt_system, prompt_user = render_prompts(spec=spec, case=case)
raw = judge_llm(system=prompt_system, user=prompt_user, temperature=0)

result = parse_json(raw)
validate_schema(result, schema=spec.output_schema)

hard_fail = len(result["hard_fails"]) > 0
score_ok = result["total_score"] >= spec.pass_threshold["min_total_score"]

verdict = "pass" if (not hard_fail and score_ok) else "fail"

if verdict == "pass":
    allow_next_action(case)
else:
    block_and_route(case, reason=result)

log_eval_artifacts({
    "spec_version": spec.version,
    "prompt_hash": hash_prompts(prompt_system, prompt_user),
    "model": "judge-model-name",
    "input": case,
    "result": result,
    "verdict": verdict
})
```

### Generalization template (for other use cases)

The same evaluator shape generalizes beyond refunds. Keep the scaffold, swap the domain modules.

```yaml
name: <use_case>_reason_eval
risk_tier: <low|medium|high>
reason_state_schema:
  intent: string
  known_facts: [string]
  unknowns: [string]
  assumptions: [string]
  constraints: object
  proposed_next_action: string
hard_fail_rules:
  - <must_not_violate_rule_1>
  - <must_not_violate_rule_2>
required_facts_before_execute:
  - <fact_1>
  - <fact_2>
forbidden_assumptions:
  - <assumption_1>
scoring_rubric:
  goal_fidelity_0_2: <criterion>
  state_completeness_0_2: <criterion>
  uncertainty_handling_0_2: <criterion>
  action_readiness_0_2: <criterion>
  policy_alignment_0_2: <criterion>
pass_threshold:
  hard_fail_count: 0
  min_total_score: <threshold_by_risk_tier>
escalation_policy:
  on_hard_fail: <block_and_route_to_human>
  on_low_score: <request_more_evidence_or_human_review>
```

### KYC-IDV adaptation (example)

Yes, this is directly applicable to KYC-IDV. In that domain, hard-fail rules are stricter and evidence requirements are explicit.

- `required_facts_before_execute`: document authenticity checks, face-match score, sanctions/PEP result, jurisdictional requirements.
- `forbidden_assumptions`: infer identity from partial match, ignore expired/blurred ID, skip sanctions checks due to timeout.
- `hard_fail_rules`: approve identity with missing mandatory evidence; override high-risk mismatch without escalation.
- `policy_context`: AML/KYC policy documents + regional regulations with precedence rules.
- `escalation_policy`: route to manual review for medium/high-risk ambiguity.

### Attribution note

The evaluator schema in this post is a practical synthesis, not a canonical standard. It combines (a) Google’s trajectory/reasoning evaluation framing (pp. 14, 50–51), (b) rubric-based LLM evaluation practice, and (c) standard safety-engineering control patterns (hard constraints + graded quality + execution gating).

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
