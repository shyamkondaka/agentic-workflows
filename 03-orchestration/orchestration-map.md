# Orchestration Map: Cortex PM Chief-of-Staff Agent

> Module 3 · Orchestration & Subagents, ★ Deliverable 3
>
> ✅ What this validates: nothing advances unchecked — by the end we'll have a justified topology, a roster, and a defined fail action.
>
> Builds on the M2 Loop Spec. Only split one agent into a team when there is a real reason; coordination has a cost.

## 1. Why split? (or why not)

Cortex stays a single agent because the loop already includes bounded data retrieval, drafting, and a human review checkpoint. There is no clear benefit from a parallel validator or a team split for this workflow, and splitting would add coordination overhead without improving the PM status-update task in a meaningful way.

## 2. Topology

**Pattern:** single-agent loop

```text
[PM task] → [Cortex: gather data, draft update + backlog] → [Human review checkpoint] → [approve / revise / reject]
```

## 3. Roster

| Agent / subagent | Responsibility | Runs which Loop Spec |
|---|---|---|
| Cortex | Pulls project facts, checks norms, drafts the leadership update and sprint stories | M2 loop |
| Human reviewer | Reviews the drafted output before any publication or commitment | Above the agent line from M1 |

## 4. Communication & hand-offs

- Input: the PM task brief, including project context and the request to draft a status update plus backlog suggestions.
- Internal hand-off: Cortex reads project state, engineering activity, norms, and prior updates; it then synthesizes a draft update and candidate stories.
- Output hand-off: the draft is passed to a human reviewer as a held draft, not as a published message.
- No auto-publish tool exists in this design, so every final decision remains above the agent line.

## 5. The validator

- **What the critic checks:** grounded claims, no invented numbers, alignment with project state, and no over-commitment or unsanctioned publishing.
- **Fail action:** hold the draft and escalate to a human for review instead of letting it move forward unchecked.
- **Revision cap:** use the bounded loop already defined in M2; the draft is not reworked indefinitely without human intervention.

## 6. State: shared vs isolated

- Shared state: task brief, project facts pulled from tools, prior updates, and the current draft.
- Isolated state: there is no separate validator state, no hidden reasoning channel, and no cross-agent memory. The human review step is outside the agent loop and remains the final gate.

## 7. Cost & latency budget

This single-agent design avoids the extra model calls and coordination cost of a validator subagent. The budget is therefore roughly one standard Cortex run: data retrieval plus a bounded number of drafting/revision passes before the final draft reaches the human reviewer. Compared to a split-agent setup, this version reduces latency and spend while keeping the human as the final decision-maker.
# Orchestration Map — Field 1: Why split (or not)

Decision: Cortex should remain a single primary agent, but paired with one independent critic subagent — not a full team.

Rationale: The trigger/listener (Hook), the dedupe/idempotency check, and the actual task-handling logic are distinct responsibilities that should live in separate layers, so the ingestion mechanism can be swapped, scaled, or debugged independently of the business logic. Parallelism is only needed if queue depth or latency demand it; add more instances only after bottlenecks appear. The independent validator is warranted to catch silent errors and policy violations before human review. Context-window pressure is not currently a blocker for per-task Hook runs; split only for large, multi-source tasks.


## Field 5 — The Validator / Critic

Checks (exact, actionable):
1) Project identity: the output must reference the correct project (project ID/name) and any referenced PR/issue IDs must be present in the pulled activity results.
2) Traceability: every numeric or statistical claim must include a source reference to the pulled activity (tool call id or short excerpt) so a human can trace it back to the evidence.
3) Embargo/sensitivity: flag and fail any line-items or roadmap points that match embargoed/confidential flags from the roadmap tool.
4) No commitments: detect explicit commitments (dates, GA language, dollar amounts) and flag them for escalation.

Fail-action: Revise — the critic returns structured reasons to Cortex; Cortex may attempt up to 2 revision iterations, after which the run escalates to a human if the critic still rejects.

Pass-action: Move the draft to the PM review checkpoint (create queued-draft metadata and do not auto-post).


(Notes: these checks should be implemented as deterministic or rule-based heuristics where possible; avoid fuzzy pass/fail rules. For complex judgments, prefer an explainable classifier and require evidence.)


## Field 2-4,6 — Topology, Roster, Hand-offs, State split

Topology (text diagram):

```
[Inbound PM task / Hook] → [Cortex (drafting agent): pulls data, drafts update + proposes stories]
                         ↘ (draft + metadata) → [Validator / Critic] 
                                               — fail → structured reasons → back to Cortex (max 2 revisions) → escalate
                                               — pass → [PM review checkpoint / queued-draft metadata]
```

Roster (one row per agent):
- Cortex (drafting agent): responsibility: pull project activity, synthesize into status update and candidate stories; runs the M2 Loop Spec. 
- Validator / Critic: responsibility: run the Field 5 checks (project identity, traceability, embargo, no commitments); returns structured reasons or pass verdicts; runs the M3 validator spec.

Hand-offs (what and how):
- From Hook → Cortex: event payload + idempotency_key (JSON)
- Cortex → Validator: draft text + run metadata + source_log (tool-call outputs and provenance) — passed as an isolated payload (no direct DB writes)
- Validator → Cortex: structured verdict {verdict: pass|fail|escalate, reasons: [...], evidence: {...}}
- Cortex → PM review: queued-draft metadata {draft_path, run_id, validator_summary, cost, idempotency_key}

Shared vs Isolated state decision:
- Shared: source data (pulled activity results) and run identifiers may be visible to both agents for traceability.
- Isolated: the validator's internal reasoning, confidence scores, and intermediate critique artifacts must remain isolated and NOT written back into Cortex's prompt history to avoid contaminating future generations. Draft text and sanitized evidence may be shared as the hand-off.

Operational notes:
- Ensure the validator cannot mutate Cortex state or write to external systems; it only returns verdicts.
- The work tree path and queued-draft metadata are the canonical artifacts for hand-off and auditing.

## Field 7 — Cost & latency budget

Budget (Option A — Balanced, chosen):
- Typical extra model calls per item: 1 critic call per draft cycle (so a typical run uses draft + critic = 2 model calls).
- Worst-case (revision cap = 2): up to 3 draft+critic cycles → up to 6 model calls total for the run.
- Typical added latency before PM review: ~2–6 seconds (model-response dependent); worst-case added latency: ~6–18 seconds.
- Enforcement / bound: the orchestrator enforces a per-run model-call cap of 6 and the existing global COST_CAP_USD (default $0.50). If the run would exceed 6 model calls or the COST_CAP_USD, it should halt and escalate (mark `stuck` or `escalated` per the Loop Spec).

Notes:
- These numbers assume the critic is a single model call per cycle; if you switch to a heavier multi-step validator, update Field 7 accordingly.
- The budget here becomes the M5 bound to monitor and tune (validator spend vs. velocity trade-off).
