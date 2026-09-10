# Workshop 1 Facilitator Guide — OTel Arcade Collector Workshop

> Reference guide only, from a different workshop (OTel Arcade Collector Workshop). Pasted in from Notion, which does not preserve formatting on copy — headers below are inferred from context, not verbatim source markup. Used as the structural/formatting template for `workshop/facilitator-guide/facilitator-guide.md`, not part of this project's own content.

**Application:** OTel Arcade (browser-based, Docker Compose)
**Delivery:** Instruqt sandbox, slide deck, usually via Zoom or in-person
**Audience:** SAs and engineers evaluating the Collector
**Total time:** ~108 min.

## Abstract

Workshop 1 introduces the OpenTelemetry Collector as a hands-on, production-realistic exercise using the OTel Arcade, a browser-based application that generates real traces, metrics, and logs as you play.

Participants configure a working Collector pipeline from scratch, connecting receivers, processors, and exporters to route telemetry simultaneously to a local Visualizer and Honeycomb.

The second half of the workshop focuses on telemetry quality: using the Collector's processor layer and the OpenTelemetry Transformation Language (OTTL) to identify and fix five real data quality problems: high-cardinality span names, PII in span attributes, health probe noise, over-long values, and redundant resource attributes, all without touching application code.

Participants leave with a working, tuned Collector pipeline and the conceptual foundation for Workshop 2, which extends the architecture to Collector self-telemetry, agent-to-gateway patterns, and tail sampling.

## SA Pre-Workshop Preparation Checklist

### Accounts & access

- Confirm your attendees have an active Honeycomb account with permission to create environments and API keys
- Confirm you have access to the Instruqt track and the correct link to distribute to attendees

### Technical validation (at least 48 hours before)

- Run through the full workshop yourself in an Instruqt sandbox if you haven't before — plan 2 hours
- Confirm the OTel Arcade app starts correctly and the Visualizer feed is live after sandbox startup
- Complete Challenges 1–5 end-to-end and verify the smells counter reaches 0
- Note the typical sandbox startup time (usually 2-3 minutes) — account for this during the Opening window!
- Confirm the OTTL transforms template loads correctly from the Deploy & Configure dropdown
- Know the primary troubleshooting command: `docker compose logs --tail=50 otel-collector-agent`

### Slides

- Open the Workshop 1 slide deck and load it onto whatever tool you're using to present
- Advance through the deck once, noting the transition points from deck to lab: slide 9 (into Fundamentals), slide 25 (into Challenges 1 & 2), slide 37 (into Challenge 3), slide 45 (into Challenge 4)
- Note slides 26, 27, and 47 as answer-key slides you'll put on screen while attendees are working — not just advancing past them

### Day-of logistics

- Arrive early enough to start a test Instruqt sandbox and confirm the stack is healthy before attendees arrive
- Load slide 1 on the main screen before the room fills
- Open this facilitator guide on a second screen
- Have the Instruqt track link ready to share (usually it's easiest to have a bitly link on the slide)
- Confirm screen sharing is working

### Nice to have

- Survey the room early on OTel experience & adjust the Fundamentals Recap pacing if most are experienced practitioners
- Identify likely fast finishers who can pair with attendees who fall behind during Challenge 4

## How to use this guide

This guide is structured challenge-by-challenge (like the way it's shaped in Instruqt). You will lecture using the slides intermittently. For each challenge you have three types of questions:

- **Opening question** — ask the room before they start. Activates prior knowledge and frames what they're about to do.
- **While circulating** — use mid-challenge to gauge where the room is and unblock anyone stuck.
- **Closing question** — ask after the challenge wraps. Surfaces what they learned and creates the bridge to the next one.

Slide references are called out inline as → Slide N throughout. Slides marked (on screen) stay visible while attendees work.

## Timing

| Section | Slides | Time |
|---|---|---|
| Opening | 1–3 | 5 min. |
| Workshop Orientation | 4–9 | 8 min. |
| Fundamentals Recap | 10–25 | 12 min. |
| Challenges 1 & 2 — First Pipeline | 26–28 | 20 min. |
| Processors lecture | 29–37 | 10 min. |
| Challenge 3 — Spot the Problems | — | 10 min. |
| OTTL lecture | 38–45 | 8 min. |
| Challenge 4 — Clean Your Telemetry | 46–48 | 30 min. |
| Challenge 5 — Checkpoint | — | 5 min. |
| **Total** | | **~108 min.** |

## Opening (5 min.) — Slides 1–3

**Learning objectives:** By the end of this section, attendees will be able to:
- Navigate their Instruqt sandbox and identify the three working tabs
- State the scope of Workshop 1 vs. Workshop 2

**→ Slide 1 — Title slide.** Welcome the room.

**What to do:** Get everyone into Instruqt. Orient them to the three tabs (OpenTelemetry Arcade, Terminal, Honeycomb) and the instruction pane on the right.

**What to say:**
- Intro: Introduce who you are and what this workshop covers.
- **→ Slide 2 — Agenda:** Walk through the sections. "Today we're covering sections 1 through 4. Sections 5 and 6 — self-telemetry, agent/gateway, and advanced patterns — are Workshop 2."
- **→ Slide 3 — Instruqt link:** "Open a browser and use the link on screen to load your Instruqt instance. Go ahead and press Start on your lab now." Wait for confirmation the room is in before continuing.
- Orient: "Three tabs at the top of your Instruqt window: OpenTelemetry Arcade is the app. Terminal is a shell for the handful of commands you'll need. Honeycomb opens your observability backend in a separate browser tab. The instruction pane on the right has all your tactical instructions and code; you can copy directly from there."

## Workshop Orientation (8 min.) — Slides 4–9

**Learning objectives:** By the end of this section, attendees will be able to:
- Describe how the Arcade app, Collector, Visualizer, and Honeycomb relate to each other
- Explain the generate → edit → inspect control loop and use it for every remaining challenge
- Navigate the OTel Arcade app's left navigation (Arcade vs. Collector sections)

**→ Slide 4 — Section break.** "While your sandboxes are loading, let me orient you to how this workshop is set up."

**→ Slide 5 — Meet the OTel Arcade**
"Here's the system. The Arcade app on the left is your traffic source. It generates real telemetry as you play. That telemetry flows through an OpenTelemetry Collector, which is the focus of everything we do today. From the Collector it fans out to two destinations: the Visualizer, a live local feed built into the app itself, and Honeycomb, your observability backend. The app is just generating traffic. The Collector is the workshop focus."

**→ Slide 6 — The workshop control loop**
"Every challenge uses the same loop. Generate traffic by playing a game or triggering load. Edit the Collector config in the browser editor. Inspect the result in the Visualizer or Honeycomb. The faster you move through this loop, the more you learn. You'll use it 10 or 15 times today."

**→ Slide 7 — Each game creates distinctive trace shapes** (brief)
"Wave Defender creates fan-out traces with many parallel child spans. Bid Wars creates retry patterns… failed attempts followed by success. Hot Cache creates cache hit/miss branching. Playing different games gives you a variety of real distributed system patterns to explore. Try a few different ones as we go."

**→ Slide 8 — From first pipeline to production patterns**
"Here's the arc across both workshops. Today is the first two blocks: building your first pipeline, about 20 minutes, and telemetry quality, about 40 minutes. The last three — Collector health, Agent/Gateway, and advanced patterns — are Workshop 2. Keep that in mind: everything you build today carries forward."

**→ Slide 9 — OTel Arcade Tour section break.**
"Your sandboxes should be up. Open the OpenTelemetry Arcade tab and click through the left navigation. You'll see two sections: Arcade at the top, the games, and Collector below your workshop tools. Take 60 seconds to explore. Play any game. You won't break anything." (Give the room a moment to orient.)

## Fundamentals Recap (12 min.) — Slides 10–25

**Learning objectives:** By the end of this section, attendees will be able to:
- Explain why the Collector exists as a layer separate from the SDK
- Name the three core Collector component types and how they connect in a pipeline
- Identify the three deployment modes and explain which applies in this workshop
- Recognize the three workshop exporters and their respective destinations

**→ Slide 10 — Fundamentals Recap — section break.**
"Quick OTel primer before we touch the config. If you've worked with OTel before, this is a fast refresher. If you're newer to it, these are the concepts you'll actually use today."

**→ Slide 11 — What is it and why should I care?** (brief)
"OTel has succeeded at a few challenging aspects of a standard: it's vendor-neutral, widely adopted, and gives you instrumentation that isn't locked to any backend. That's why you're here using it with Honeycomb instead of proprietary agents."

**→ Slide 12 — Straight from the SDK**
"The simplest setup: your application has the OTel SDK, which exports telemetry directly to a backend. This works, but every service needs to know where to send data. Changing backends means touching every service. The Collector solves this."

**→ Slide 13 — Collector components**
"The Collector sits between your services and your backend. Three component types: receivers pull telemetry in, processors do work on it, exporters push it out. These connect in a pipeline: one named route for one signal type. Extensions, in the yellow block at the bottom, are supporting services like health checks that run alongside the pipeline without being part of it. In Challenge 1, we'll be wiring those three blocks together."

**→ Slide 14 — Deployment modes**
"Three ways to deploy the Collector. Direct: no Collector, SDK sends straight to the backend, fine for getting started. Agent: a Collector runs close to each application, usually sidecar or DaemonSet, handles local processing. Gateway: a centralized Collector that all agents forward to, where you apply organization-wide policy. Today you're in Agent mode. Workshop 2 adds the Gateway tier."

**→ Slides 15–16 — Signal types / Structured events**
"The Collector handles traces, metrics, and logs through separate pipelines. Each pipeline is independent, so you can apply different processors per signal."

**→ Slide 17 — Deep, wide, structured events**
"This is what Honeycomb is built around: wide, structured events. Look at the event column on the right: a single event can carry up to 2,000 fields of context. That's high dimensionality: pack in whatever helps you debug later, because you can't always predict which field will matter. Now look at cardinality, the second axis: how many unique values a field can actually take. Boolean is low cardinality — just true and false. User ID and Transaction ID are high cardinality… potentially unique per event. Hang onto that distinction: high-cardinality span names and PII are two of the five problems you'll fix with OTTL in Challenge 4."

**→ Slide 18 — Architecture examples**
"Two production reference architectures: a generic multi-service setup and a Kubernetes DaemonSet + Gateway pattern. If you're asked 'how does this scale?' these are good starting points."

**→ Slide 19 — Receivers & Exporters — section break.** "Let's look specifically at the two components you'll wire in Challenge 1."

**→ Slides 20–21 — Receivers** (generic)
"The OTLP receiver listens on port 4317 for gRPC and 4318 for HTTP; the Arcade services are already configured to send there. In production you'd layer in host_metrics, prometheus, file_log, and the Kubernetes-specific receivers. The pattern is always the same: configure the receiver, wire it into a pipeline."

**→ Slide 22 — Receivers** (workshop-specific)
"In the Arcade config: OTLP in on gRPC and HTTP, arcade services send to otel-collector-agent, and the pattern for this workshop is OTLP in through processors out to multiple exporters. That's Challenge 1 in one line."

**→ Slides 23–24 — Exporters**
"Three exporters in the workshop config. otlp_grpc/backend goes to Honeycomb; it reads HONEYCOMB_API_KEY from your .env file and sends to api.honeycomb.io:443. otlp_http/visualizer sends to the local live feed; it uses JSON encoding and TLS disabled because it's localhost. debug writes to the Collector logs, useful for debugging when nothing else is working. The exporters are already defined in the config. Your job in Challenge 1 is to wire them into the pipelines. Important to note: exporters have no default endpoint. You configure where data goes."

**→ Slide 25 — Example Configuration — section break.**
"Let's go look at the actual config. In the Collector tab, go to Deploy & Configure."

## Challenges 1 & 2: First Collector Pipeline (~20 min.)

**Learning objectives:** By the end of these challenges, attendees will be able to:
- Wire a working Collector pipeline by connecting defined exporters to the pipelines block
- Connect the pipeline to Honeycomb by injecting an API key into the .env file
- Verify telemetry is flowing to both the Visualizer and Honeycomb from all three services

**→ Slide 26 — Challenges 1 & 2: First Collector Pipeline** (on screen while they work)
Show before they start. "Here's what you're building: OTLP receiver through memory_limiter, out to debug, Honeycomb, and the Visualizer. The components are already defined in the config: receivers, processors, exporters are all there. Your job is wiring them into the pipelines block. Success: the topology populates in the Visualizer and spans arrive from all three services."

**Opening question:** "How many of you have worked with an OpenTelemetry Collector before? And for those who have, what did you put through it?"
- Listen for: A range of experience. If most are new, spend an extra beat on the pipeline concept before they start editing. If there are experienced users, ask them what they found hardest to configure the first time… it surfaces good context for the room.

**Visual:** Slide 26 + OpenTelemetry Arcade tab → ◈ Visualizer — the feed is empty. The empty state is the hook.

**What to say:**
- Explain: "Go to the Visualizer. You should see… nothing. The Collector is receiving spans right now — your services are running and sending telemetry — but the pipelines aren't wired to send them anywhere. That empty feed is your starting state. By the end of this challenge it'll be filling with live data."
- Explain (while they read the config): "Receivers, processors, and exporters are all defined. Find the pipelines section. Each pipeline — traces, metrics, logs — currently exports only to debug. You'll see otlp_grpc/backend and otlp_http/visualizer defined under exporters but not connected to any pipeline. An exporter only receives telemetry if it's wired in. Add both to all three pipeline definitions."
- Explain (after Apply & Restart): "Notice what just happened. You changed one line in three places — the exporters list — and the Collector restarted with the new config. No service restart, no code deploy. That's the value of having the Collector as a dedicated layer."

**While circulating:**
- "Does the Visualizer show a topology now? You should see your receiver connecting to your exporters."
- "Is the feed filling up? Play a game if you haven't generated any traffic."
- "What three exporters do you see in the pipeline diagram?"
- "Feed still empty? Check the Terminal tab: `docker compose logs --tail=50 otel-collector-agent`. A YAML syntax error will show up there."

**→ Slide 27 — Challenges 1 & 2 Walkthrough** (on screen as answer key)
Put this up while the room is working through Challenge 2. "The walkthrough steps are on screen as a reference. You can also follow the instructions in your Instruqt panel; both get you to the same place."

**Closing question** after Challenge 2 — Question to the audience: "You're in Honeycomb now. Click into a trace. What can you tell about a request just from what's there?"
- Listen for: Service names, span names, HTTP routes, durations, status codes. "Take a closer look at the attribute values. Not everything in that feed is clean or useful. That's what we're going to investigate."

**→ Slide 28 — Checkpoint** — show after both challenges are complete.
"Four things before we move on. Live feed and topology in the Visualizer — check. Traces from all three services in Honeycomb — check. Debug exporter and Logs panel as a local fallback — check. And a troubleshooting reference if anything is still empty: check encoding: json, tls.insecure, API key, and Collector logs. Verify all four before the next section."

**Bridge:** "Your Collector is forwarding to two destinations simultaneously. Now let's talk about what happens in the middle — the processors — and look at the data quality problems that are hiding in your feed right now."

## Processors (10 min.) — Slides 29–37

**Learning objectives:** By the end of this section, attendees will be able to:
- Name the always-recommended processor (memory_limiter) and explain the consequence of omitting it
- Apply the FILTER → TRANSFORM ordering rule and explain why sequence matters
- Distinguish between the filter processor (drop spans) and the transform processor (modify spans)

**→ Slide 29 — Processors — section break.**
"The middle layer. Processors run in sequence between your receivers and your exporters. Let's look at what they can do and how to order them."

**→ Slide 30 — Always recommended**
"One processor that should be in every pipeline, and it's not enabled by default: memory_limiter prevents the Collector from crashing under load. When it gets close to its memory limit, it starts refusing new telemetry and applying backpressure to the receiver. Batching used to be a separate processor; it isn't anymore. It's now handled by the sending_queue settings on the exporter itself. The strategic ordering rule on this slide: drop, filter, or sample as early as possible, before spending any processing work on data you're about to throw away."

**→ Slide 32 — Order counts!**
"This is an important operational rule for Collector pipelines. For traces and logs: memory_limiter first, then filtering and sampling processors, then anything else. See the annotation at the bottom: FILTER → TRANSFORM. Dropping is cheap. Transforming costs CPU. Batching happens automatically in the exporter's sending_queue now, not as a pipeline step you place yourself. You'll wire this sequence directly in Challenge 4."

(Slide 31 — Managing resource telemetry: skip or reference briefly if a question comes up about the attributes or resource detection processors.)

**→ Slide 33 — Example attributes & spans** (brief)
"What goes on a span? Four categories — environmental (pod name, region, deploy ID), code-level (function name, status codes, errors), product (feature flags, user IDs, tenant), and external (third-party APIs, databases, queues). The Collector can add, modify, or remove any of these. Including things the application never explicitly instrumented."

**→ Slide 34 — Security Processors**
"Two processors for data security. The filter processor drops spans or attributes entirely based on OTTL expressions, including data that should never leave your infrastructure. The transform processor hashes or replaces sensitive values before export. The Redaction Pattern in the middle is exactly what you'll apply in Challenge 4: automated player.id redaction. The rule at the bottom: always drop or redact before export. You can't un-send something that's already gone out."

(Slide 35 — Sampling Processors: mention briefly as a preview. "Head sampling decides immediately at the start of a trace; it's simple but it can drop the errors you need. Tail sampling buffers the full trace and then applies policies. Tail sampling lives at the Gateway; that's later in Workshop 2")

**→ Slide 36 — Transform Processor** (brief/light)
"This is your Swiss army knife for telemetry modification."

**→ Slide 37 — When to transform structured events**
"Here are six scenarios where the Collector's transform capability pays for itself. Look at this list: normalize high-cardinality span names, redact or hash sensitive attributes, drop noisy probes with filter, truncate long values, delete redundant resource attributes, convert fields into backend-friendly shape. These are not hypothetical. Every one of these is in your Visualizer feed right now. Your next challenge is to go find all five of them."

**→ Transition to Challenge 3:** "Open the Visualizer in the OpenTelemetry Arcade tab."

## Challenge 3: Spot the Problems (~10 min.)

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Identify all five telemetry quality problems in a live Visualizer feed
- Articulate why each problem creates real cost, compliance, or query usability issues in production

Leave slide 37 on the main screen as a reference checklist while they work.

**Visual:** OTel Arcade tab → ◈ Visualizer — feed is live, some spans highlighted orange. ⚠ counter in the header is non-zero.

**Opening:** "See the orange-highlighted spans? The Visualizer marks spans with known telemetry quality problems. That ⚠ counter in the header is your score. Work through the five investigation questions in your instruction pane. Expand spans and look at the actual attribute values."

**While circulating:**
- "What's the current smells count? Play a few different games to get a representative sample."
- "Found the player.id attribute? What type of data is it? What's the compliance risk if that goes to a production backend?"
- "Look at the SQL span names on score-api. Could you write a useful query grouping by those as-is?"
- "How many health probe spans are you seeing in a 30-second window? What's the signal-to-noise ratio there?"
- "Find a leaderboard span. Does it have both app.name and service.name? What do they contain?"

**Closing question** — Pair/share: "You've seen all five problems. Pick one and tell your neighbor why it's actually a problem in production, not just for this exercise."
- Listen for: Cardinality arguments, PII compliance concerns, cost arguments, query usability. Let the conversation run for a bit.

**Bridge:** "Every one of those problems is fixable in the Collector, in the pipeline, before data ever reaches a backend. The language for doing that is OTTL."

## OTTL (8 min.) — Slides 38–45

**Learning objectives:** By the end of this section, attendees will be able to:
- Explain what OTTL is and where it executes within the Collector pipeline
- Read and reason about basic OTTL statements using set, delete_key, replace_pattern, truncate_all, and where
- Choose between the transform processor and the filter processor for a given telemetry problem

**→ Slide 38 — OTTL — section break.**

**→ Slide 39 — Introducing OTTL**
"OTTL is the Collector's built-in expression language for making decisions about and changes to telemetry. Four things it gives you: direct access to OTLP fields by name, transforms expressed as compact single-line statements, the same pattern reusable across traces metrics and logs, and the ability to clean data before it ever leaves your infrastructure. It's already in the Collector, no plugins, no external tools."

**→ Slide 40 — Language features**
"Four building blocks. Standard functions — set, delete, replace, truncate — cover most use cases. Built-in enums for common OTLP fields so you don't have to memorize string constants. where clauses for conditionals: 'do this, but only when this attribute has a value' or 'only when this condition is true.' And error_mode: ignore, which covers runtime data errors while processing live spans — not a malformed statement itself. A bad statement still fails Collector startup regardless of error_mode; that's a config-time error, not a data-time one. Look at the example on the right: `set(attributes["user_id"], "anonymous") where attributes["user_id"] == nil`. That's a complete, readable policy in one line."

**→ Slide 41 — OTTL Standard Library** (reference)
"You don't need to memorize this. The workshop config uses a handful of these: replace_pattern, set, truncate_all, delete_key. This slide is a reference for after the workshop. The note at the bottom is the right instinct: use the scaffold first, then expand deliberately."

**→ Slide 42 — Where can I use OTTL?**
"Three places. Processors: the transform processor and the filter processor, which is how you'll use it today. Connectors: routing and count connectors, which are Workshop 2. And specifically for this workshop: Challenge 4 is all transforms and filters; Workshop 2 builds on this with conditional routing at the Gateway tier."

(Slide 43 — OTTL Case Study: skip unless the room has time and appetite; it's a good deep-dive example of a production k8s log parsing pipeline.)

**→ Slide 44 — OTTL in practice**
"Four rules to take with you. Use transform when you want to modify a span. Use filter when you want to drop it entirely; don't transform something you're about to discard. Keep resource edits in the resource context and span edits in the span context …they're separate scopes in OTTL. Prefer readable statements over clever one-liners. And validate often, a YAML syntax error and an OTTL semantic error both prevent startup but look different in the logs."

(Slide 45 — Useful OTTL Examples: reference table. Point it out as a post-workshop resource, don't walk through each row.)

## Challenge 4: Clean Your Telemetry (~30 min.)

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Apply five OTTL transforms to fix real telemetry quality problems using transform/normalize and filter/drop_probes
- Verify each fix incrementally using the Visualizer Split view and smells counter
- Arrange processors in the correct order in the traces pipeline and explain the rationale

**→ Slide 46 — Challenge 4: Clean telemetry with OTTL** (on screen before they start)
"Here's what you're doing: fixing all five deliberate smells. Observe first… the orange highlights and the ⚠ counter show what remains after each fix. You'll use transform/normalize for span-level edits and filter/drop_probes to remove unwanted spans entirely. Success: counter reaches 0 and the Split view shows Before → After diffs for each change."

**Opening question:** "You have five problems. Before you touch anything, which one do you fix first, and why?"
- Listen for: Drop the health probes first (highest volume), fix PII first (compliance risk), fix cardinality first (query impact). All are defensible and the conversation is the point. Then land the principle: "There's a right answer for pipeline ordering: FILTER → TRANSFORM. Drops are cheap. Don't transform what you're about to drop."

**Visual:** Slide 46 + OTel Arcade tab → ◈ Visualizer → Split view. The Split view shows spans before and after your transforms side by side. Amber borders mean the span was modified.

**What to say:**
- Explain: "Load the OTTL transforms template from the Deploy & Configure dropdown. This gives you a scaffolded config with commented-out OTTL statements, one per fix. Uncomment them one at a time so you can see each one take effect in the Split view before moving on. The Split view is your feedback loop."
- Fix 1 — Normalize SQL span names: "This regex replaces any span whose name starts with a SQL keyword with the string db.query. Before: SELECT * FROM scores WHERE player_id = 'abc123'. After: db.query. One statement collapses thousands of unique per-query names into one stable, queryable label."
- Fix 2 — Redact player.id: "This replaces the value of player.id with *** on every span that carries it. The key is preserved (you can confirm PII was present) but the actual identifier is never stored. That's how you meet a PII requirement."
- Fix 3 — Drop health probes: "This is a filter processor, not a transform. Filter drops spans entirely. Health probes are never useful in Honeycomb. Notice where it goes in the pipeline: before transform. Drops are cheap. Don't spend transform work on spans you're about to discard."
- Fix 4 — Truncate long values: "truncate_all applies a character limit to every attribute on a span in one statement. Full user-agent strings, raw stack traces all capped at 128 characters. One line instead of one statement per attribute."
- Fix 5 — Remove redundant app.name: "The leaderboard service attaches app.name to its spans at the resource level. It's identical to service.name, the same value stored twice. delete_key removes it. Notice this fix uses the resource context in OTTL, not the span context, because app.name is a resource attribute. That's the OTTL context distinction from the last slide."
- Land it: "Look at the After column. PII redacted, cardinality controlled, noise dropped, storage optimized. Any service that sends to this Collector gets all five of these fixes automatically."

**→ Slide 47 — Challenge 4 Walkthrough** (put on screen mid-challenge as answer key)
"The walkthrough steps are on screen. If you're stuck on any fix, the expected outcome for each one is listed here."

**While circulating:**
- "What does the smells counter say now? Which fixes have you applied?"
- "Find a span in the Split view where Before and After look different. What changed?"
- "Look at the processors list in your traces pipeline. What order are they in? Does it match the rule?"
- "Still seeing smells after all five fixes? Select an orange-highlighted span to see which attribute is still flagged."

**Closing question** — Question to the audience: "You built this without writing a single line of application code. What does that tell you about where the Collector fits in your stack?"
- Listen for: "It's a centralized policy layer." "It doesn't care what language the services use." "One config change applies to every service." Guide toward: "The Collector is the right place to enforce telemetry quality: centralized, vendor-agnostic, in your infrastructure rather than your application. Change the pipeline config once; every service sending to this Collector benefits."

**→ Slide 48 — Checkpoint**
"Four things before we're done. Smells counter reads 0. Split view shows amber-highlighted rows where transforms changed spans. No health probe spans in the feed. And resource cleanup — the app.name deletion — happens in the resource OTTL context, not the span context, which is why you'll see it on leaderboard resource attributes specifically."

**Bridge:** "Counter is at zero. Let's verify the final config and close out."

## Challenge 5: Checkpoint and Handoff (~5 min.)

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Confirm their final config is complete with all five fixes applied in the correct processor order
- Articulate the Collector's role as a centralized, language-agnostic telemetry policy layer

**Visual:** OTel Arcade tab → ⚙ Deploy & Configure → Collector config. Confirm all five fixes are uncommented and the processor order reads [memory_limiter, filter/drop_probes, transform/normalize].

**What to say:**
- Explain: "Let's look at where you started and where you are now. You arrived with a Collector that was running but sending telemetry nowhere useful. The Visualizer was empty, Honeycomb had no data, and the raw feed had five telemetry quality problems."
- Explain: "In the last couple hours you: wired a working Collector pipeline from receivers through processors to two exporters simultaneously; connected that pipeline to Honeycomb with a real API key; and used OTTL to fix five real telemetry quality problems, like high-cardinality names, PII, health probe noise, over-long attribute values, and a redundant resource attribute. All of it in YAML, no app code."
- Note: "Keep your Honeycomb API key and your otel-arcade-workshop environment accessible! Workshop 2 picks up exactly where you are now."

**Closing question** — Open discussion: "Before we close out… think about your own systems. If your telemetry went through this same inspection today, what do you think you'd find in the feed?"
- Listen for: PII risk in their own traces, cardinality problems they already know about, health check noise they've never measured.
