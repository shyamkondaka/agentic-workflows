# Loop Spec: Cortex PM Chief-of-Staff Agent

> Module 2 · Loop Engineering, ★ Deliverable 2
>
> ✅ **What this validates:** the agent knows when to run and when to stop — by the end you'll have proven a one-page Loop Spec with a trigger, a definition of "done," and explicit stop conditions.
>
> Your one-page blueprint for how the work you handed to the agent (M1) actually *runs*.
> An agent is just a prompt that fires itself, this spec says when it fires, what "done" means, and what it needs to do the job. Living document; refine as the course progresses.

## 1. Trigger & loop type

**Chosen type:** Hook (trigger on inbound task/message)

**Why this type:**
Triggering on inbound task/message (vs. polling) gives near-real-time responsiveness while avoiding wasted compute and rate-limit pressure from constantly checking for new work.

**Deduplication / idempotency:**
Use an idempotency key (e.g., hash of message ID + payload) checked against a short-lived cache/store before processing, so retried or duplicate deliveries of the same event are skipped rather than re-executed.

## 2. Goal / definition of done

**Definition of done (chosen):** Draft queued for human approval — Cortex writes a draft into the run's isolated work tree, runs the automated validator (bounds, embargo/safety flags, basic grounding checks), and creates a queued-draft record for a human reviewer. Cortex must not post, publish, merge, or otherwise take external action without explicit human approval.

**Success criteria (detectable):**
- A draft file exists in the work tree (path recorded in the run metadata).
- An entry with status `queued` is created in the draft queue (or inbox) with a validator-passed flag.
- Automated validator outputs a pass result: bounds checks OK, no embargo/safety flags, grounding evidence present.
- Idempotency key confirmed so duplicate deliveries didn’t create duplicate queued drafts.

## 3. Stop conditions

| Condition | What it looks like | What happens |
|---|---|---|
| **Success** | Draft file present + queued record + validator passed | Loop stops; run marked `success`; notify reviewer(s) with queued-draft metadata |
| **Stuck / give up** | Any data-source pull fails after 3 retry attempts, OR model fails to produce an acceptable draft after 3 revision iterations (validator fails repeatedly) | Stop the run, mark `stuck`, log the failure with diagnostics, create an incident entry for human review and (optionally) schedule a retry according to backoff policy |
| **Escalate to human** | Any HITL condition from M1 (embargoed content, GA-date commitment, spending/cap breach) OR safety/policy flags exceed threshold | Immediately stop and raise an escalation: mark `escalated`, notify the human on-call/team with full run logs and evidence, and do not queue or publish the draft |


## 4. State

**Locked state (always-on):** The loop maintains a scoped, minimal state store that is explicitly versioned and available to the run but prevents cross-project leakage. The following items are persisted per-run (and when relevant per-project):

- run_id, start_ts, end_ts, status (queued | running | success | stuck | escalated)
- idempotency_key (hash of message ID + payload) and short-lived cache TTL
- path to the run's isolated work tree and draft file(s)
- automated validator result and evidence (bounds, grounding checks, embargo flags)
- retry/backoff counters and last-error diagnostics for external pulls
- queued-draft metadata (queue entry id, reviewer list, priority, created_ts)
- light audit trail: decision points, scores from automated checks, and a link to full run logs

State constraints:
- State is per-project and per-run; no cross-project secrets or long-lived sensitive fields are stored.
- The idempotency cache is short-lived (configurable TTL) to avoid unbounded storage.
- The state schema is intentionally small to reduce attack surface and accidental persistence of PII or secrets.

## 5. The five things a loop can lean on

**Overview:** `state` is always-on. Only enable `connectors` if credentials are already provisioned; otherwise list them as a planned connector. For `skills`, `subagents`, and `work tree`, prefer the smallest variant that satisfies the spec.

1) Work tree (isolated workspace per run)
- Purpose: provide an isolated filesystem for drafts, temporary artifacts, and any git worktree operations. Ensures runs are reproducible and auditable.
- Implementation notes: create a fresh temp worktree per run, record path in run metadata, and delete or archive on completion per retention policy.

2) Skills (reusable capabilities)
- Purpose: small, testable functions (e.g., extract-activity, summarize-commits, render-draft) that the loop calls instead of redeclaring prompts each time.
- Current plan: minimal set — summarize activity, grounding-check, apply-bounds — "not needed yet" for advanced orchestration features.

3) Plugins / connectors (optional)
- Purpose: access external systems (Jira, GitHub, Slack, Google Drive) when available. Only enable when credentials are provisioned and approved in the project.
- Fail-safe: if a connector is not present, the loop degrades to read-only drafting with a human review gate.

4) Subagents (independent checks)
- Purpose: run evaluation or validation steps the main loop shouldn't self-grade (e.g., a replay-eval or critic model). Useful for goal loops and for non-trivial validation.
- Current plan: placeholder → surface in M3 if/when needed. For M2 keep subagent usage minimal: automated validator is a light-weight local step.

5) State tracking / orchestration primitives
- Purpose: small schedulers, retry/backoff logic, and the queue/inbox entry model that records queued drafts and reviewer notifications.
- Implementation notes: inbox_entries (or a simple queue table) records status transitions; orchestrator ensures the 3-stop rules are enforceable and observable.


> Note: The spec locks `state` (the canonical per-run data) and treats connectors as optional. If a connector is added later, update the loop-spec and the build with explicit consent and provisioning steps.

> Context plan (M4) and the hand-off to bounds & evals (M5) come in later modules — you'll add them to their own deliverables then, not here.

## Link to live loop

_[path to your agent in `00-build/`]_
