# Production & Autonomy: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 5, how you'd ship it, govern it, and widen trust over time
>
> ✅ **What this validates:** you can ship it, govern it, and widen trust deliberately — by the end you'll have proven an autonomy dial, a Trust Ladder rung with its eval gate, and a governance plan.

## Autonomy Dial by segment

_Autonomy is a product decision per user, not one global setting._

| Segment | Desired autonomy | Why |
|---|---|---|
| New/Junior Admin (first 90 days) | assisted | needs Cortex to surface findings and draft recommendations; no destructive actions without senior review |
| Experienced Org Admin | supervised | can review and approve deletion candidates and story batches with explicit sign-off |
| Release / Platform Engineering Team | bounded-autonomous | allowed to auto-execute low-risk, high-confidence actions within a pre-approved scope |


### Autonomy Dial — Detailed blocks (approved)

## 1) New/Junior Admin (first 90 days) — assisted

- Trust Ladder rung: assisted
- Allowed actions: surface findings, draft recommendations, suggest edits and remediation steps; never execute destructive or irreversible actions.
- Required approvals: Senior Admin or Org Owner review required for deletions, major config changes, or any action that affects production state.
- Escalation policy: auto-escalate any items flagged as embargoed, containing PII, or with low grounding/traceability scores; urgent escalations go to the on-call org owner.
- Numeric gate (move up/down): maintain >=95% grounding score on retrievals and <=2 high-impact rejected suggestions per 30 days to be eligible for supervised rung; demotion if >3 rejected high-impact suggestions in 30 days.
- Monitoring cadence: weekly audit of Cortex suggestions and an automated weekly digest of suggested actions with senior reviewer feedback recorded.

---

## 2) Experienced Org Admin (owns this org day-to-day) — supervised

- Trust Ladder rung: supervised
- Allowed actions: review and approve deletion candidates, review and approve story batches for queueing; can accept Cortex recommendations but explicit human sign-off required before execution.
- Required approvals: explicit sign-off (UI confirm or signed-off ticket) required before any execution; Cortex may persist suggested changes to queued-draft store but must not post or execute without sign-off.
- Escalation policy: escalate ambiguous or cross-org impact cases to the org owner or platform team; embargoed or PII items always escalate.
- Numeric gate (move up/down): target >=90% grounding and traceability scores; downgrade autonomy if >=3 rejected/rolled-back actions within a 30-day window or if audit finds traceability gaps.
- Monitoring cadence: bi-weekly audit and a monthly metric report summarizing approvals, rejections, and system confidence trends.

---

## 3) Release / Platform Engineering Team (cross-org, high-volume cleanup) — bounded-autonomous

- Trust Ladder rung: bounded-autonomous
- Allowed actions: auto-execute low-risk, high-confidence actions within a pre-approved scope (examples: deleting classes or artifacts with zero references across static and dynamic analysis, pruning stale feature flags with confirmed owners, low-risk housekeeping tasks).
- Required approvals: platform leads pre-approve the execution scope and the exact automated rules; ambiguous or out-of-scope items must be escalated for human review before execution.
- Escalation policy: immediate halt and notification (to on-call and platform leads) when confidence <98%, when cross-repo touches are detected, or when static/dynamic checks disagree; embargoed/PII items must never be auto-executed and must escalate.
- Numeric gate (move up/down): require automated test pass rate = 100% for affected modules in CI, passing static+dynamic zero-reference checks, and a sustained high-confidence score (>=98%) over the last N runs; demote autonomy if any auto-executed action causes a rollback or an incident within 7 days.
- Monitoring cadence: daily runbook review and real-time alerts; a daily digest for platform leads showing auto-executed actions and confidence metrics.

---

Notes
- All segments inherit the global validator/critic checks (project identity, traceability, embargo/sensitivity, no-commitment rules). Any critic fail that recommends escalation must be treated as binding and may short-circuit auto-execution even for bounded-autonomous segments.
- Numeric gates should be instrumented as concrete metrics exposed in monitoring dashboards and used as programmatic checks prior to execution.


"Committed-by" and next steps
- This draft is intended for review. Next action options: (A) refine gates and monitoring thresholds per segment, (B) convert these blocks into runtime-enforced policies in 00-build/agent.py + policy store, or (C) proceed to Step 2 (define autonomy evaluation gates and metrics) when ready.

## Trust Ladder

- **Current rung:** supervised
  - Cortex is a read-only, queued-draft agent in production: it can gather evidence, draft status updates, and queue backlog proposals, but it never posts or executes without a human review gate.
- **Eval gate to reach the next rung:** the next rung is bounded-autonomous only for the Release / Platform team, and the gate is: >=98% grounding score, >=99% critic pass rate, 100% CI pass for affected modules, and zero rollbacks or incidents across the last 7 days of runs; minimum sample set of 50 runs or 200 runs in a high-volume week.
- **Incident record so far:** the repo demonstrates the expected incident pattern: missing-data and jailbreak cases are held and escalated, embargo/PII findings are rejected, and the bounded high-risk actions are never executed without human review. This is the clean incident record we want to maintain when widening autonomy.

## Deployment plan

- **Runtime:** self-hosted + managed control plane, not an uncontrolled public agent. The runtime is the repo-local Cortex build (`00-build/agent.py`) with a bounded tool layer, independent critic, and explicit HITL checkpoint. This keeps the loop readable, auditable, and reversible.
- **Operator / on-call owner:** the Release / Platform Engineering lead owns the runtime, and the Org Owner owns the policy for any high-impact or cross-org execution. The model is not the operator; the human owners are.
- **Rollback:** drop the agent to the previous rung, disable the auto-exec env flag (`CORTEX_AUTO_EXEC=0`), disable the tool set, or revert to a prior prompt/version; all safe actions are still possible because the agent never posts or executes by default.
- **Monitoring:** monitor grounding score, critic pass rate, auto-exec blocks, rejections, cost spent vs. cap, and incidents. The local audit artifacts in `00-build/run-output/` serve as a first-tier evidence store for on-call review and dashboarding.

## ROI metrics (beyond adoption & tokens)

| Metric | Target |
|---|---|
| Task completion rate | >90% of queued drafts are accepted by human reviewers without needing a rewrite |
| Time saved / cost-to-serve | reduce triage + status-drafting time by 30–50% for standard PM reporting flows |
| Trust incidents | 0 embargo/PII or execution incidents in any 30-day window; any incident triggers a freeze and review |
| Human review latency | median approval within 1 business day for standard drafts; >2-day latency triggers investigation |
| Cost discipline | stay under the configured cost cap for 95% of runs |

## Widen-autonomy decision rule

We widen the dial by one rung only when: (1) the relevant M5 eval thresholds are met for two consecutive evaluation windows, (2) there are no rollback or compliance incidents, and (3) the segment-specific human reviewers sign off on the policy. In short: metric evidence first, human approval second, and no hidden exceptions.

## Governance & forward strategy

- **Compliance:** never put embargoed, confidential, or PII-bearing data into prompts or tool results outside the authorized project context; treat anything with embargo markers as a hard-block and escalate. The agent line and critic remain the final guardrails.
- **Safety:** posting, committing, or destructive actions remain above the line for everyone. The kill switch is the runtime flag that disables auto-exec and reduces the agent to assist-only mode; the model never negotiates around those bounds.
- **Reliability:** keep the cost cap, max iterations, and revision cap explicit; escalate on stuck tool failures; and maintain a model-down fallback that falls back to a human review queue instead of silent retries.
- **Strategy:** the next capability to widen is the Release / Platform Engineering team into bounded-autonomous execution on a narrow, pre-approved low-risk action set. The eval gate remains the strict CI + grounding + critic pass gate above; if it fails, the segment returns to supervised until metrics recover.
