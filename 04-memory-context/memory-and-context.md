# Context Engineering & Memory: Cortex PM Chief-of-Staff Agent

> Module 4 · Context Engineering & Memory
>
> ✅ What this validates: the agent reasons on the right, safe inputs — by the end we have a context budget, per-source retrieve-vs-long-context decisions, and a memory map with risk mitigations.

## 1. Context budget

Cortex should optimize for high-signal, auditable inputs rather than broad context. The current task brief and the team norms are the highest-priority in-context inputs because they define the exact ask and the hard safety boundaries. The live project state, recent activity, and relevant historical precedent are retrieved because they change often and need precise evidence trails.

Priority order:
1. Current PM task brief (`get_task`) — the direct user ask; stays in context.
2. Team norms (`get_norms`) — safety guardrails; kept in context because they define hard constraints and escalation rules.
3. Current project facts (`get_project`) — exact project metadata, status, flags, PRD linkage.
4. Recent engineering activity (`get_activity`) — retrieved because it is large, grows with each sprint, and must be cited precisely.
5. Past updates (`search_past_updates`) — retrieved to find tone and precedent without dumping the full history into context.
6. Roadmap (`get_roadmap`) — retrieved because it can include confidential or embargoed items; the agent must respect those flags.

The principle is simple: keep the prompt small, keep the evidence specific, and make every claim traceable to the source used.

## 2. Retrieve vs. long-context: per source

| Source | Size / volatility | Decision | Why |
|---|---|---|---|
| `get_task` | bounded / static | Long-context | This is the single PM brief and short enough to reason over directly. |
| `get_norms` | bounded / policy | Long-context | The norms are a compact safety layer and define hard constraints; keeping them in context makes the limits explicit. |
| `get_project` | bounded / structured | Long-context | The project metadata is small and directly relevant to the run, so it should be visible without a search step. |
| `get_activity` | large / changing | Retrieve | The activity log grows each sprint and needs precise PR/issue references; long-context would blur the evidence trail and risk over-inclusion. |
| `search_past_updates` | large / noisy | Retrieve | Past updates are unbounded and noisy; the useful precedent is a narrow slice tied to the project and style. Retrieval keeps the context focused. |
| `get_roadmap` | medium / controlled | Retrieve | The roadmap is medium-sized and can contain confidential or embargoed items. It should be treated as a controlled source, not a broad context dump. |

## 3. Retrieval quality plan

The retrieved sources all use the same retrieval-quality baseline: document grading + self-verification. This is the move that separates agentic retrieval from naive “search → dump.”

- **Document grading**: was the retrieved content actually relevant and safe to use?
- **Self-verification**: did the draft use the retrieved evidence and not invent numbers or claims?
- **Routing**: query the right source for the right question (project fact, activity, or precedent).
- **Reranking**: keep only the relevant slice from a larger search result set.
- **Caching**: reuse stable facts like norms or previous project summaries when they are unchanged.

| Source | Retrieval-quality move | Why this move fits |
|---|---|---|
| `get_activity` | document grading + self-verification | The agent must confirm that the PRs/issues/metrics it cites are actually present and that no claim is invented or over-generalized. |
| `search_past_updates` | document grading + self-verification | Past-update results need to be filtered to the relevant precedent; the model must verify the example matches the current project and tone, not just the keyword hit. |
| `get_roadmap` | document grading + self-verification | The model must confirm whether an item is in scope, shareable, or embargoed before it appears in an update. |

## 4. Memory map (your PM brain)

| Memory type | What Cortex stores | Scope / TTL |
|---|---|---|
| **Working** | the current PM task brief, the in-flight draft, retrieved evidence, and the current constraints or escalations | this run only |
| **Episodic** | recent prior drafts, decisions, and failure patterns from past runs | a short rolling window of recent runs |
| **Semantic** | project summaries, team norms, preferred update structure, and durable policy rules | durable until the project or policy changes |
| **Shared** | only sanitized, already approved facts and attributes relevant across agents | controlled, high-trust only |

This design keeps memory minimal: Cortex remembers only the facts and rules it genuinely needs to reason well without drifting into stale or over-broad context.

## 5. Memory risks & mitigations

| Risk | Mitigation |
|---|---|
| Drift | Refresh the data pack and retrieve fresh sources at runtime rather than relying on stale assumptions. |
| Poisoning | Treat task brief content as data, not instructions; keep norms in context and escalate if the brief tries to override the rules. |
| Staleness | Use project-specific retrieved examples and verify that past precedent still matches the current evidence. |
| Confidential / retention | Keep embargoed items out of broad context, require source checks before inclusion, and never reference them in company-wide updates. |

This memory plan keeps Cortex grounded, bounded, and audit-friendly: short in-context inputs, narrow retrieved evidence, and explicit safety rules that are checked before the draft is finalized.
# Memory & Context — §1 Context budget

Priority: ground Cortex in the freshest possible operational data while minimizing unnecessary pulls. Sources are prioritized by how often they must be fresh and by auditability needs:

1. get_task — highest priority (single-run trigger): retrieve per-run and do not cache.
2. get_activity — high priority (volatile engineering activity): retrieve per-run to ensure citation and freshness.
3. get_roadmap — high priority for embargo checks: retrieve per-run to verify confidential/embargo flags.
4. get_norms — medium priority: long-context with periodic TTL refresh since norms change infrequently but must be citable.
5. search_past_updates — lower priority for freshness but high audit value: store in long-context (memory) and use reranking for retrieval.


## §2 Retrieve vs long-context (per source)

- get_task: Retrieve per-run — the task brief is the trigger and unique per run, so it must be pulled fresh each invocation.

- get_activity: Retrieve per-run — activity is recent and volatile, and we must cite specific PR/issue evidence on every run for grounding and audit.

- get_roadmap: Retrieve per-run — roadmap items include embargo/confidential markers that must be checked freshly each run to avoid accidental leaks.

- get_norms: Long-context (store in memory and refresh periodically) — norms are stable and infrequently changing, so caching with a TTL reduces cost while keeping citations auditable.

- search_past_updates: Long-context (store in memory) — the corpus is large and low-volatility; caching with reranking enables stable precedent retrieval and lowers repeated cost.


## §3 Retrieval quality plan (source × agentic moves)

Proposed grid (recommended):

- get_task (Retrieve per-run):
  - Routing: parse the task brief to identify project_id, scope, and required artifacts.
  - Document grading: minimal — ensure the task contains a valid project and non-empty body.
  - Self-verification: confirm the parsed project_id matches a valid project via get_project.
  - Caching: none (single-run trigger).

- get_activity (Retrieve per-run):
  - Document grading: filter out irrelevant items (e.g., non-merged PRs if only merged PRs matter) and remove noisy logs.
  - Reranking: rank activity items by recency and relevance to the task (e.g., mentions of the requested project area).
  - Self-verification: verify that numbers/metrics cited in the draft match entries in the activity list.
  - Caching: short-lived cache (TTL seconds to minutes) to absorb retry storms while keeping freshest data.

- get_roadmap (Retrieve per-run):
  - Document grading: detect CONFIDENTIAL/EMBARGO markers and tag lines that must not be shared.
  - Self-verification: confirm any roadmap references in the draft are present and allowed for the intended audience.
  - Caching: short TTL but prefer fresh pull for embargo checks.

- get_norms (Long-context):
  - Document grading: ensure norms are parsable and include required policy anchors (dates, commitments, forbidden phrases).
  - Reranking: not usually needed; norms are small and cited directly.
  - Caching: store in long-context with TTL (e.g., 24h) and refresh on explicit admin update.

- search_past_updates (Long-context):
  - Reranking: use semantic reranking to surface the most relevant precedent documents for the current task.
  - Document grading: score retrieved updates for citation quality (presence of explicit evidence, links to PR/issue IDs).
  - Caching: keep an indexed long-context corpus and refresh periodically; use vector indexes for fast retrieval.


## §4 Memory map + risks & mitigations

Memory map (recommended):

- Working memory (per-run, ephemeral):
  - Contents: run_id, source_log (tool outputs), parsed task metadata, temporary draft versions, idempotency_key, retry counters.
  - TTL / retention: discard or archive after 24 hours unless run is marked `queued` or `escalated`.
  - Access: only the current run's processes; not shared across agents except via sanitized hand-off payloads.

- Episodic memory (past runs / threads):
  - Contents: queued-draft metadata (run_id, draft_path, validator_summary), incident records (stuck/escalated runs), reviewer actions.
  - TTL / retention: retain for 90 days for audit and replay; older entries archived to read-only storage.
  - Access: read-only to agents for precedence; writes only via explicit append operations (no silent mutations).

- Semantic memory (durable facts & preferences):
  - Contents: project canonical IDs and aliases, team norms snapshots (latest validated copy), persistent configuration knobs (queue cap), reviewer lists.
  - TTL / retention: durable with manual admin rotation; norms refreshed on admin change or TTL (24h) if sourced externally.
  - Access: read by agents; writes require human-approved update flow (HITL).

- Shared index / vector store (long-context corpus):
  - Contents: embeddings and metadata for `search_past_updates`, optional cached snippets for `get_norms` and other long-contexts.
  - TTL / retention: refresh cadence weekly or on explicit ingest; retain vectors for 180 days.
  - Access: read-only for Cortex; validator may read for cross-checks but must not write to the vector index.

Risks & mitigations:

- Drift (memory diverges from reality):
  - Mitigation: TTLs, periodic refreshes, and an ingest pipeline that updates vector indexes and semantic memory under human supervision.

- Poisoning (malicious or corrupted data in long-context):
  - Mitigation: document grading + source signing, restricted ingest authority, and spike-detection alerts on sudden content shifts.

- Staleness (outdated norms or roadmap):
  - Mitigation: short TTL for embargo-sensitive sources, versioned norms with visible timestamps, and an admin-triggered refresh flow.

- PII / sensitive leakage: 
  - Mitigation: redact PII on ingest, enforce per-source confidentiality markers (do not include CONFIDENTIAL items in shared contexts), and audit logs for every read that includes redacted material.

- Over-reliance on cached sources (false confidence):
  - Mitigation: require self-verification for any cited numeric claim (trace back to the raw tool output) and include citation tokens in drafts for human review.


✅ CHECKPOINT: memory map specifies working/episodic/semantic/shared stores with TTLs and concrete mitigations for drift, poisoning, staleness, and sensitive data leakage.

Proposed grid (recommended):

- get_task (Retrieve per-run):
  - Routing: parse the task brief to identify project_id, scope, and required artifacts.
  - Document grading: minimal — ensure the task contains a valid project and non-empty body.
  - Self-verification: confirm the parsed project_id matches a valid project via get_project.
  - Caching: none (single-run trigger).

- get_activity (Retrieve per-run):
  - Document grading: filter out irrelevant items (e.g., non-merged PRs if only merged PRs matter) and remove noisy logs.
  - Reranking: rank activity items by recency and relevance to the task (e.g., mentions of the requested project area).
  - Self-verification: verify that numbers/metrics cited in the draft match entries in the activity list.
  - Caching: short-lived cache (TTL seconds to minutes) to absorb retry storms while keeping freshest data.

- get_roadmap (Retrieve per-run):
  - Document grading: detect CONFIDENTIAL/EMBARGO markers and tag lines that must not be shared.
  - Self-verification: confirm any roadmap references in the draft are present and allowed for the intended audience.
  - Caching: short TTL but prefer fresh pull for embargo checks.

- get_norms (Long-context):
  - Document grading: ensure norms are parsable and include required policy anchors (dates, commitments, forbidden phrases).
  - Reranking: not usually needed; norms are small and cited directly.
  - Caching: store in long-context with TTL (e.g., 24h) and refresh on explicit admin update.

- search_past_updates (Long-context):
  - Reranking: use semantic reranking to surface the most relevant precedent documents for the current task.
  - Document grading: score retrieved updates for citation quality (presence of explicit evidence, links to PR/issue IDs).
  - Caching: keep an indexed long-context corpus and refresh periodically; use vector indexes for fast retrieval.


✅ CHECKPOINT: each retrieved source has at least one agentic move justified by its failure mode; the plan balances freshness, auditability, and cost.
