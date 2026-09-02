# Build Insights: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 4, what you learned building it
>
> ✅ **What this validates:** you can reflect on what building it taught you — by the end you'll have proven the friction, the learning, and the aha that changes how you'd design your next agent.

## Friction

The hardest part was not the coding itself — it was designing the loop so the agent was safe even when it got tempted to overreach. The early friction came from the tension between making the agent helpful and giving it enough constraints to stop. It was especially hard to create boundaries that were visible in code, not just in prompts: the revision cap, the tool retry policy, the queue cap, the refusal rules, and the explicit no-post rule all had to be enforced outside the model.

The second source of friction was context management. The agent was very good at drafting a polished answer when it had enough evidence, but it could also drift or hallucinate if it was forced to infer unavailable facts. That is why the retrieval and grounding checks mattered so much. The validator had to work as a second system that checked the model's output against source data instead of trusting the narrative it produced.

## Learning

The first big learning is that safe agent behavior is not an LLM problem alone — it is a systems problem. The model can write a good update, but the loop, the tool surface, the enforcement logic, and the human checkpoint are what keep it from overcommitting or publishing unverified work.

The second learning is that trust is a product choice, not an implicit property of the model. The right autonomy level depends on who the user is, what actions are in scope, and whether there is enough evidence to justify auto-execution. Making that explicit in policy is more important than trying to make the model “be safe” in the abstract.

The third learning is that evals need to be operational, not aspirational. Running a good set of failure fixtures — missing data, jailbreaks, validation failure, and runaway loops — is what gives the system credibility. Without those runs, the agent looks impressive in a happy path and falls apart under edge conditions.

## Aha moment

The aha was realizing that the real design question is: what happens when the model is wrong, under pressure, or trying to optimize for completion? Once we explicitly modeled the stop conditions, the human review gate, the critic, and the cost/iteration caps, the system became legible and governable. That changed the way I would design any future agent: start with the no-go rules, then build the helpful path on top of them.

## What you'd do differently

If I rebuilt Cortex from scratch, I would front-load the telemetry and evaluation framework earlier. I would also reduce the amount of “smartness” in the core loop and make the runtime policy more explicit, so segment-based autonomy decisions were enforced by data rather than by prompt wording. I would also build a real policy store and a small dashboard earlier rather than relying on local JSON artifacts alone, because that is the gap between a prototype and an operating system.

