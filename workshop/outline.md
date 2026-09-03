# Workshop Outline: Effective Sampling with Honeycomb Refinery

**Target length:** ~4 hours as drafted below

## Learning Objectives

By the end of this workshop, learners will be able to:
1. Explain what telemetry sampling is, why it matters for cost and fidelity, and the tradeoffs between head, tail, and dynamic sampling.
2. Explain how Honeycomb Refinery ingests, caches, and makes trace-level sampling decisions.
3. Read and modify a Refinery `rules.yaml` file to target specific traffic with deterministic and dynamic samplers.
4. Diagnose why a dynamic sampler is failing to hit its goal sample rate (high-cardinality `FieldList`) and correct it.
5. Apply a sensible default rule set as a starting point for their own environment.

## Prerequisites

- Familiarity with telemetry data types (metrics, logs, traces), cardinality, and dimensionality
- Existing instrumentation on their own services (not required for the workshop itself, but assumed context of how to understand data in Honeycomb)
- Comfortable using the Honeycomb Query Builder
- Basic familiarity with Docker Compose (they will not need to write it, just recognize what it's doing)

---

## Welcome & Environment Setup (Facilitator-led, no slides, 10 min)

**Objective:** Learners understand the agenda and can access their running Instruqt environment.

**Artifacts needed:**
- Facilitator guide: welcome script, agenda slide, housekeeping notes
- Instruqt: track landing page with access instructions

## Module 1: What Is Sampling? & Head vs. Tail Sampling (Lecture, 20 min)

**Learning objectives:**
- Define sampling and articulate the cost/fidelity tradeoff, including the effect on cardinality
- Delineate head sampling vs. tail sampling and why teams often combine both

**Artifacts needed:**
- Slide deck: "What Is Sampling?" (**no storyboard/deck exists for this one — only the final rendered video; stills already extracted from it, still need matching to the script**) + Head v. Tail Sampling (storyboard in hand)
- Facilitator guide: talking points and timing

## Module 2: Refinery Architecture & How It Processes Data (Lecture, 25 min)

**Learning objectives:**
- Explain that Refinery can run standalone or clustered, and the resource-planning implications (memory > CPU > network) of each
- Summarize how Refinery caches spans and what triggers a trace decision (root span + SendDelay, TraceTimeout, SpanLimit)

**Artifacts needed:**
- Slide decks: Refinery Architecture (storyboard in hand), How Refinery Processes Data (deck in hand)
- Facilitator guide section with timing and any live-demo notes

## Module 3: Guided Environment Tour (Hands-on lab, 20 min)

Note: Learners do **not** build the pipeline by hand. The Instruqt environment starts with loadgen → collector → Refinery → Honeycomb already running.

**Scope decision:** Read-only tour — no editing here. Learners view and answer questions about the existing files; the first edit they make is in Module 5, Challenge 1.

**Learning objectives:**
- Locate and read the pre-provisioned `docker-compose.yaml`, collector config, and Refinery `config.yaml`/`rules.yaml` to understand what's already wired together
- Run a baseline Honeycomb query (group by `app.function` / `app.endpoint`) against live, already-sampled data to confirm the pipeline is working

**Artifacts needed:**
- Instruqt: pre-provisioned track environment (app + pipeline running at track start)
- Instruqt challenge

---

## Break (Facilitator-led, no slides, 10 min)

---

## Module 4: Refinery Configuration & Rules Files (Lecture, 15 min)

**Learning objectives:**
- Read the structure of a Refinery `config.yaml` and `rules.yaml` (RulesVersion, Samplers by environment, RulesBasedSampler, Conditions, deterministic sampler)

**Artifacts needed:**
- Slide deck: Refinery Configuration and Rules Files (deck in hand) — **only the config / RulesBasedSampler / Conditions / deterministic-sampler portion.** Hold back its `EMADynamicSampler` slide for Module 6, where it belongs with the dynamic sampling concept.
- Facilitator guide

## Module 5: Challenge 1 — Using Rules to Change How Data Is Sampled (Hands-on lab, ~30 min)

**Learning objectives:** Update `rules.yaml` to keep all traces with `status_code >= 500` plus a deterministic catchall; interpret the change via a calculated field, `meta.refinery.reason`, and Honeycomb Usage Mode.

**Artifacts needed:** Instruqt challenge + check script (validates rules.yaml content and/or query result), lab guide steps, facilitator notes on common mistakes (YAML indentation, rule ordering).

## Module 6: What Is Dynamic Sampling? (Lecture, 15 min)

**Learning objectives:**
- Explain how dynamic sampling adjusts rates based on observed key frequency to hit a goal sample rate
- Configure Refinery's `EMADynamicSampler` (`GoalSampleRate`, `FieldList`)

**Artifacts needed:**
- Slide deck: What is Dynamic Sampling? (storyboard in hand) + the `EMADynamicSampler` slide held back from Module 4's deck
- Facilitator guide

## Module 7: Challenge 2 — Adding a Dynamic Sampler (Hands-on lab, ~35 min)

**Learning objectives:** Replace the deterministic catchall with `EMADynamicSampler` (`GoalSampleRate` + `FieldList`); verify the sampler is hitting its goal using `meta.refinery.sample_key` in Usage Mode.

**Artifacts needed:** Instruqt challenge + check script, lab guide steps, facilitator notes.

## Module 8: Sampling and Fidelity Tradeoffs (Lecture, 10 min)

**Learning objectives:** Explain why high-cardinality fields undermine dynamic sampling, and when full-fidelity (no sampling) or targeted tail-sampling rules are the better call instead.

**Artifacts needed:**
- Slide deck: Sampling and Fidelity Tradeoffs (storyboard in hand; canonical example is **payment transactions**)
- Facilitator guide — keep this one short, since it leads directly into watching the failure happen live in Module 9

## Module 9: Challenge 3 — High Cardinality in Dynamic Samplers (Hands-on lab, ~35 min)

**Learning objectives:** Add a high-cardinality field (`app.user_id`) to the FieldList and observe the sampler fail to hit its goal (keyspace exceeds the 500-key default max); explain why and revert/correct it.

**Artifacts needed:** Instruqt challenge + check script, lab guide steps, facilitator notes — this is the "productive failure" moment, so facilitator guide should explicitly call out that the failure is the intended learning outcome, not a bug in the lab.

---

## Module 10: Sensible Defaults & Wrap-up (Lecture, 20 min)

**Learning objectives:**
- Recall why rule ordering matters (most-specific rules first, catchall last)
- Apply a sensible starting rule set (keep errors/500s, drop health checks, keep long-duration traces, dynamically sample the rest) as a template for their own environment

**Artifacts needed:**
- Slide deck (already in hand): Sensible Default Refinery Rules — **note the Rule 6 value bug flagged in `visual-asset-inventory` memory needs fixing before this deck ships**
- Facilitator guide: recap talking points, pointers to Honeycomb docs for going deeper, workshop close script

---

## Q&A / Close (Facilitator-led, no slides, 15 min)

**Artifacts needed:** Facilitator guide close script, feedback survey link.

---

## Open questions / next decisions

- **Slide-decks:** the existing "Refinery Configuration and Rules Files" deck currently covers both deterministic rules AND the `EMADynamicSampler` in one deck. It needs to be split across Module 4 and Module 6 when we actually build the slide deck — flag this for that drafting session.
- ~~**"What Is Sampling?" visuals**~~ — **Resolved.** Stills were extracted from the source video and matched to the script; Module 1's slide deck is built (see `workshop/slide-briefs/module-01-understanding-sampling.md`). The extracted frames were deleted once no longer needed.
