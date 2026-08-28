# Workshop Outline: Introduction to Refinery

**Target length:** 3.5–4 hours

## Workshop-level learning objectives

By the end of this workshop, learners will be able to:
1. Explain what sampling is, why it matters for cost/fidelity, and the tradeoffs between head, tail, and dynamic sampling.
2. Explain how Refinery ingests, caches, and makes trace-level sampling decisions.
3. Read and modify a Refinery `rules.yaml` to target specific traffic with deterministic and dynamic samplers.
4. Diagnose why a dynamic sampler is failing to hit its goal sample rate (high-cardinality FieldList) and correct it.
5. Apply a sensible default rule set as a starting point for their own environment.

## Prerequisites

- Familiarity with data types (metrics, logs, traces), cardinality, and dimensionality
- Existing instrumentation on their own services (not required for the workshop itself, but assumed context)
- Comfortable using the Honeycomb Query Builder
- Basic familiarity with Docker Compose (they will not need to write it, just recognize what it's doing)

---

## Welcome & Environment Setup (Facilitator-led, no slides, 10 min)

**Objective:** Learners understand the agenda and can access their running Instruqt environment.

**Artifacts needed:**
- Facilitator guide: welcome script, agenda slide, housekeeping notes
- Instruqt: track landing page with access instructions

## Module 1: Understanding Sampling (Lecture, 35 min)

Condenses Academy Section 1's four videos into one lecture block — no hands-on component; this is pure conceptual grounding before learners touch Refinery.

**Learning objectives:**
- Define sampling and articulate the cost/fidelity tradeoff, including the effect on cardinality
- Delineate head sampling vs. tail sampling and why teams often combine both
- Explain how dynamic sampling adjusts rates based on observed key frequency to hit a goal sample rate
- Explain why high-cardinality fields undermine dynamic sampling, and when full-fidelity (no sampling) or targeted tail-sampling rules are the better call instead

**Artifacts needed:**
- Slide deck covering: "What Is Sampling?" (**visual asset still pending from video producer**), Head v. Tail Sampling (storyboard in hand), What is Dynamic Sampling? (storyboard in hand), Sampling and Fidelity Tradeoffs (storyboard in hand) — needs to be consolidated into one cohesive deck rather than four separate ones
- Facilitator guide: talking points and timing per sub-topic, suggested transition lines between the four concepts
- Decide canonical example for the fidelity-tradeoff section (see `visual-asset-inventory` memory — script uses "profile writes on a dating app," storyboard uses "payment transactions")

---

## Module 2: Getting Started with Refinery — Concepts (Lecture, 35 min)

**Learning objectives:**
- Explain that Refinery can run standalone or clustered, and the resource-planning implications (memory > CPU > network) of each
- Summarize how Refinery caches spans and what triggers a trace decision (root span + SendDelay, TraceTimeout, SpanLimit)
- Read the structure of a Refinery `config.yaml` and `rules.yaml` (RulesVersion, Samplers by environment, RulesBasedSampler, Conditions, deterministic vs. dynamic samplers)

**Artifacts needed:**
- Slide decks (already in hand): Refinery Architecture storyboard, How Refinery Processes Data, Refinery Configuration and Rules Files
- Facilitator guide section with timing and any live-demo notes

## Module 3: Getting Started with Refinery — Guided Environment Tour (Hands-on lab, 20 min)

**Adapted from Academy A4/A5** — per the [[workshop-adaptation-decisions]] memory, learners do **not** build the pipeline by hand. The Instruqt environment starts with loadgen → collector → Refinery → Honeycomb already running.

**Learning objectives (adapted):**
- Locate and read the pre-provisioned `docker-compose.yaml`, collector config, and Refinery `config.yaml`/`rules.yaml` to understand what's already wired together
- Run a baseline Honeycomb query (group by `app.function` / `app.endpoint`) against live, already-sampled data to confirm the pipeline is working

**Artifacts needed:**
- Instruqt: pre-provisioned track environment (app + pipeline running at track start)
- Instruqt challenge: "tour" checkpoints (open file X, answer what it configures; run query Y, confirm expected shape of results)
- **Open item (flagged in [[workshop-adaptation-decisions]]):** exactly how much of the config gets shown vs. explained here, and whether any editing starts in this module or waits for Module 4 — needs a decision before the Instruqt track is built

---

## Break (Facilitator-led, no slides, 10 min)

---

## Module 4: Tuning Refinery Rules (Hands-on lab, ~100 min)

The workshop's hands-on core — three sequential Instruqt challenges on the same running environment, ported from Academy Section 3 A1–A3.

### Challenge 1: Using Rules to Change How Data Is Sampled (~30 min)
**LOs:** Update `rules.yaml` to keep all traces with `status_code >= 500` plus a deterministic catchall; interpret the change via a calculated field, `meta.refinery.reason`, and Honeycomb Usage Mode.
**Artifacts:** Instruqt challenge + check script (validates rules.yaml content and/or query result), lab guide steps, facilitator notes on common mistakes (YAML indentation, rule ordering).

### Challenge 2: Adding a Dynamic Sampler (~35 min)
**LOs:** Replace the deterministic catchall with `EMADynamicSampler` (GoalSampleRate + FieldList); verify the sampler is hitting its goal using `meta.refinery.sample_key` in Usage Mode.
**Artifacts:** Instruqt challenge + check script, lab guide steps, facilitator notes.

### Challenge 3: High Cardinality in Dynamic Samplers (~35 min)
**LOs:** Add a high-cardinality field (`app.user_id`) to the FieldList and observe the sampler fail to hit its goal (keyspace exceeds the 500-key default max); explain why and revert/correct it.
**Artifacts:** Instruqt challenge + check script, lab guide steps, facilitator notes — this is the "productive failure" moment, so facilitator guide should explicitly call out that the failure is the intended learning outcome, not a bug in the lab.

---

## Module 5: Sensible Defaults & Wrap-up (Lecture, 25 min)

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

- Module 3's exact scope (how much config-reading vs. editing) — see [[workshop-adaptation-decisions]]
- Whether Module 4's three challenges share one continuous Instruqt track environment (state carries forward) or reset between challenges
- Canonical example for Module 1's fidelity-tradeoff section (dating app vs. payment transactions)
- This schedule totals ~4 hours before trimming — first candidates to cut/shrink if the slot is tighter: Module 2 (concepts) could compress, or Challenge 3 (high cardinality) could become a facilitator demo instead of a hands-on challenge
