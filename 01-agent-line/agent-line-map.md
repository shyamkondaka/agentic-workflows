# Agent Line Map: Cortex PM Chief-of-Staff Agent

> Module 1 · The Agent Line
>
> ✅ What this validates: every risky action has a clear owner — by the end we have an above/below-the-line map with HITL checkpoints, scored on reversibility, blast radius, and measurability.

## 1. The workflow, decision by decision

| Decision / action | Reversibility (H/M/L) | Blast radius (H/M/L) | Measurability (H/M/L) | Above / Below | HITL? |
|---|---|---|---|---|---|
| Pull project state + activity | H | L | H | Below | No |
| Decide relevant context | H | L | H | Below | No |
| Draft the weekly leadership status update | H | M | H | Below | No |
| Decide tone / commitment level | M | H | M | Above | Yes |
| Flag at-risk / escalation | H | M | H | Below | No |
| Choose what to escalate | M | H | M | Above | Yes |
| Propose next sprint's stories (within cap) | H | M | H | Below | No |
| Post an update / approve a company-wide one | L | H | H | Above | Required |

## 2. Golden rule, applied

- Pull project state + activity sits below the line because it is highly reversible, low blast radius, and easy to verify.
- Decide relevant context sits below the line because it is a bounded reasoning step with low operational risk.
- Draft the weekly leadership status update sits below the line because the draft is editable and remains a proposal until a human approves it.
- Decide tone / commitment level is a HITL step because commitment language has meaningful blast radius and can overstate certainty before a human reviews it.
- Flag at-risk / escalation sits below the line because identifying risk is part of the analysis, not a human decision to act.
- Choose what to escalate sits above the line because only a human can decide what deserves escalation and what a human must own.
- Propose next sprint's stories (within cap) sits below the line because it is a queued proposal, not a real commitment.
- Post an update / approve a company-wide one sits above the line because it is the irreversible, high-blast-radius action that must remain human-owned.

## 3. Agent anatomy (sketch)

- **Model:** the default fast model drafts the update and reasons over the project facts, while a human remains the approval layer for commitment language and external send actions.
- **Tools:** project + activity lookup (read), past-update search, roadmap, team norms, and a capped story proposal tool.
- **Memory:** current task brief, retrieved facts, and short-lived working context; durable project and policy memory are held only when they are still relevant.
- **Loop:** placeholder for M2 loop-spec.md
- **Bounds:** placeholder for M5 bounds-and-evals.md
- **Evals:** placeholder for M5 bounds-and-evals.md

## 4. Hardest call

The hardest call was `decide tone / commitment level`. It sits at the boundary between useful drafting and unsafe certainty. The deciding axis was blast radius: a confident sentence can sound like a commitment even when the agent has only prepared a draft. Because the act of making a commitment is not the same as gathering facts, it belongs in HITL rather than full autonomy.
