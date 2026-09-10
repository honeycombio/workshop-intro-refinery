# Facilitator Guide — Effective Sampling with Honeycomb Refinery Workshop

**Application:** Refinery sample app (Docker Compose: `loadgen1`/`loadgen2`(/`loadgen3`) → OpenTelemetry Collector → Honeycomb Refinery → Honeycomb)
**Delivery:** Instruqt sandbox, slide deck, usually via Zoom or in-person
**Audience:** SAs and engineers evaluating Refinery for tail/dynamic sampling
**Total time:** ~4 hr 20 min as scheduled below (`workshop/outline.md`'s header rounds this to "~4 hours")

## Abstract

This workshop introduces Honeycomb Refinery as a hands-on exercise in tail and dynamic sampling. Attendees start with a fully wired, already-running telemetry pipeline.  Two (later three) load generators sending synthetic traffic through an OpenTelemetry Collector, into Refinery, and on to their own personal Honeycomb environment, and spend the workshop shaping what Refinery keeps.

The first half covers sampling fundamentals: what sampling is and why it trades cost for fidelity, head sampling vs. tail sampling, and how Refinery's architecture and internal trace-caching model work. Attendees then write their first `rules.yaml`, using a `RulesBasedSampler` to guarantee errors are never lost while everything else samples at a flat rate.

The second half introduces dynamic sampling: attendees replace their deterministic catchall with an `EMADynamicSampler`, watch it hit its goal sample rate cleanly on a moderate-cardinality field, and then — deliberately — watch it fail once a high-cardinality field is added to the keyspace. That failure is the intended learning outcome: it's the fastest way to understand why field choice matters. The workshop closes with a sensible default rule set attendees can adapt for their own environment.

## SA Pre-Workshop Preparation Checklist

### Accounts & access

- Confirm you have access to the Instruqt track and the correct link to distribute to attendees
- **Note:** unlike some workshops, there is no shared/track-level Honeycomb credential here. Each attendee creates their own personal Honeycomb account and API key as part of Instruqt Challenge 1. Nothing to provision on your end per-attendee, but you do have to budget time for it and make sure attendees know to bring or create a personal account, not rely on a company SSO team account.

### Technical validation (at least 48 hours before)

- Run through the full workshop yourself in an Instruqt sandbox if you haven't before — plan at least the full ~4 hr 20 min, since several challenges (2–4) build on each other in a continuous environment rather than resetting
- Confirm the custom host image (`effective-sampling-refinery-image`) is current. **If any of `refinery_configs/`, `collector_configs/`, or `docker-compose.yaml` changed since the image was last built, flag CEd and rebuild the image before this workshop runs** — restarting a track does not pick up repo changes that are already baked into an existing image 
- Confirm `docker compose ps -a` shows `otel-collector` and `refinery` as `Up` shortly after sandbox start (`loadgen1`/`loadgen2` will show `Up` or `Exited (0)` depending on timing — both are expected, they're a finite ~2-minute burst, not a bug)
- Complete Instruqt Challenges 1–4 end-to-end yourself, including creating your own scratch Honeycomb environment and API key exactly as attendees will
- Use each challenge's **Check** button as you go, but know that it validates `rules.yaml`/`docker-compose.yaml` content and container state, not Honeycomb query results, so a passing check doesn't replace actually looking at the data
- Know the primary troubleshooting commands: `docker compose logs otel-collector`, `docker compose logs refinery`, and `docker compose ps -a`
- Know the most common attendee error: a `401 response for AuthInfo request` or `check your API key` error in the Refinery logs almost always means `.env` still has the placeholder key, or the pipeline wasn't restarted (`./stop && ./run`) after editing it; env vars only load at container start

### Slides

- Open the slide deck and load it onto whatever tool you're using to present
- Decide whether to keep slide 3 ("Instruqt Lab — [link here]") and, if so, paste in the real track link before presenting
- Advance through the deck once, noting transition points from deck to lab

### Day-of logistics

- Arrive early enough to start a test Instruqt sandbox and confirm the stack is healthy (`docker compose ps -a`) before attendees arrive
- Load the title slide on the main screen before the room fills
- Open this facilitator guide on a second screen
- Have the Instruqt track link ready to share (usually easiest to use a bitly link on the slide)
- Confirm screen sharing is working

### Nice to have

- Survey the room early on Honeycomb/OTel/sampling experience and adjust pacing accordingly — the Prerequisites section of `workshop/outline.md` assumes familiarity with cardinality/dimensionality and basic Docker Compose recognition, but not Refinery itself
- Identify likely fast finishers early — Instruqt Challenges 3 and 4 (~35 min each) are the longest and most likely to produce a wide spread in pace; fast finishers can pair with attendees who fall behind

## How to use this guide

This guide is structured the way the workshop actually runs: alternating lecture modules and hands-on Instruqt challenges. For each hands-on challenge you have three types of questions:

- **Opening question** — ask the room before they start. Activates prior knowledge and frames what they're about to do.
- **While circulating** — use mid-challenge to gauge where the room is and unblock anyone stuck. Several of these are drawn directly from real issues found while live-testing this lab, not hypothetical ones.
- **Closing question** — ask after the challenge wraps. Surfaces what they learned and creates the bridge to the next section.

**A numbering note:** This guide always uses the **Instruqt sandbox's own numbering** (Challenge 1 = tour, Challenge 2 = using rules, Challenge 3 = dynamic sampler, Challenge 4 = high cardinality) since that's what's on screen in front of you and attendees.

Slide references are formatted `**→ Slide N — Title**` and match the final 60-slide deck.

## Timing

| Section | Time |
|---|---|
| Welcome & Environment Setup | 10 min |
| What Is Sampling? & Head vs. Tail Sampling | 20 min |
| Refinery Architecture & How It Processes Data | 25 min |
| Instruqt Challenge 1: Explore Your Refinery Pipeline | 20 min |
| Break | 10 min |
| Refinery Configuration & Rules Files | 15 min |
| Instruqt Challenge 2: Using Rules | 30 min |
| What Is Dynamic Sampling? | 15 min |
| Instruqt Challenge 3: Adding a Dynamic Sampler | 35 min |
| Sampling and Fidelity Tradeoffs | 10 min |
| Instruqt Challenge 4: High Cardinality in Dynamic Samplers | 35 min |
| Sensible Defaults & Wrap-up | 20 min |
| Q&A / Close | 15 min |
| **Total** | **~4 hr 20 min** |

## Welcome & Environment Setup (10 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Navigate their Instruqt sandbox and identify the three working tabs
- State the workshop's agenda and total time commitment

**What to do:** Load slide 1 before the room fills. Get everyone into Instruqt. Orient them to the three tabs (Refinery Sample Application code editor, Terminal, Honeycomb) and the instruction pane on the right.

**What to say:**
- **→ Slide 1 — Effective Sampling with Honeycomb Refinery**. Intro: Introduce who you are and what this workshop covers — Honeycomb Refinery, tail and dynamic sampling, hands-on the whole way.
- **→ Slide 2 — Today's Agenda.** "Today alternates short lectures with hands-on labs in your own Instruqt sandbox. You'll end up with a working, tuned Refinery pipeline and a sensible starting rule set you can adapt for your own environment."
- **→ Slide 3 — Instruqt Lab.** If you've kept this slide, this is where you show/click the track link — paste the real URL in before presenting.
- Housekeeping: bathroom/break location, ask questions any time, one scheduled 10-minute break partway through.
- Instruqt link: share the track link, wait for confirmation the room is in before continuing.
- Orient: "Three tabs at the top of your Instruqt window: Refinery Sample Application is the sample app's code; you'll edit files here starting in Challenge 2. Terminal is a shell for the handful of commands you'll need. Honeycomb opens your observability backend in a new window, since Honeycomb blocks being embedded in an iframe. The instruction pane on the right has all your tactical instructions and code; you can copy directly from there."
- Set expectations: "You will not build this pipeline by hand. It's already running: loadgen traffic, through a Collector, through Refinery, into Honeycomb. Today's job is understanding it and then shaping what Refinery keeps."

## What Is Sampling? & Head vs. Tail Sampling (Lecture, 20 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Define sampling and articulate the cost/fidelity tradeoff, including the effect on cardinality
- Delineate head sampling vs. tail sampling and why teams often combine both

**→ Slide 4 — What Is Sampling?** (title + objectives)
"By the end of this section you'll be able to define sampling, explain why it's reliable, and see how it helps balance cost and performance in tools like Honeycomb."

**→ Slide 5 — A Century-Old Idea**
"In 1908, a statistician at a brewery figured out something remarkable: you don't need to test every pint to know if the beer's good. Using a small sample, he could predict with confidence how the entire batch would turn out. That method — what we now call the T-Test — proved that a well-chosen slice of data can tell the whole story. We're bringing that same principle to telemetry. Think of it like lossy compression for images or audio: you don't have every pixel or sound wave, but you still see the picture, hear the music, get the message — still useful."

**→ Slide 6 — Sampling Your Telemetry**
"In telemetry, sampling means sending just a subset of your data instead of everything — you're still aiming to understand trends and patterns, just with less volume. Done right, sampling gives you a statistically useful view of your system's behavior, without drowning in data."

**→ Slide 7 — Won't Sampling Affect My Ability to Query My Data?**
"Refinery expresses this as a sample rate, in a 1/N format. The ratio between total events generated and events actually sent to Honeycomb. Honeycomb adjusts for this automatically: it adds metadata about how much data was sampled, and your COUNT/SUM/etc. results get scaled back up, so your queries still reflect reality even though you sent less data."

**→ Slide 8 — Sample Rate 1000, In Practice**
"A sample rate of 1000 means each event sent to Honeycomb represents 1,000 actual events that occurred. You're sending 1 out of every 1,000, and the other 999 are discarded, with COUNT and SUM multiplied by 1,000 to compensate."

**→ Slide 9 — The Biggest Reason for Sampling: Cost** (brief)
"The biggest reason for sampling is cost. Ingesting and storing high-volume, high-cardinality telemetry gets expensive fast, both economically and computationally. Smart sampling reduces both, while keeping the observability that matters."

**→ Slide 10 — Key Takeaways** (recap)
"Sampling means sending a smaller, representative slice of your telemetry. It reduces cost while keeping visibility. Tools like Refinery let you adjust for sampling, so your insights stay accurate."

**→ Slide 11 — Head Sampling v. Tail Sampling** (title)
"What if you really need to make sure you capture certain data, like errors, payment transactions? That's where different sampling strategies come in. There are two main kinds: head sampling and tail sampling."

**→ Slide 12 — Head vs. Tail, at a Glance**
"Head sampling makes decisions early, before the telemetry is generated, or by quickly comparing something like a trace ID against a hash to decide if all spans in that trace should be kept or dropped. It's fast and cheap: the decision logic is simple and happens immediately, no need to store or process trace data first.
Tail sampling makes decisions after the telemetry has been received, once the entire trace is available. By inspecting the data first, you can keep the most critical data and drop more of the less critical data. This is much more computationally intensive. You have to hold the telemetry until the full trace arrives, then parse all its attributes to decide. At really high volumes, tail sampling can be expensive on its own."

**→ Slide 13 — Why Not Both?** (recap)
"So how do teams balance the two? Head sampling is fast but uninformed. Tail sampling is expensive but smart. Many companies combine both: head sampling to reduce overall volume, tail sampling on top of that to retain the most important traces. Here's the key idea: at high trace volume, individual traces become less critical, because any important pattern is likely to repeat. Head-sampling just 10–50% of your traffic makes it feasible to tail-sample the rest.

Head sampling: fast, probabilistic. Tail sampling: precise, compute-intensive. Combined: scalable, insightful observability."

**Bridge:** "Refinery is a tail-sampling proxy: it needs the whole trace before it can decide. Let's look at how it's actually architected to do that."

## Refinery Architecture & How It Processes Data (Lecture, 25 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Explain that Refinery can run standalone or clustered, and the resource-planning implications (memory > CPU > network) of each
- Summarize how Refinery caches spans and what triggers a trace decision (root span + `SendDelay`, `TraceTimeout`, `SpanLimit`)

**→ Slide 14 — Refinery Architecture** (title, brief)

**→ Slide 15 — Deploying Refinery**
"Refinery can be deployed as a standalone binary or as a cluster of nodes: as a container on Kubernetes or ECS, or as a binary on a Linux system like EC2. Here's the most important thing to understand: Refinery makes its best sampling decisions when it can see the entire trace, not just individual spans. All services that create spans belonging to the same trace must be routed to the same Refinery instance; otherwise, you risk keeping part of a trace and dropping the rest, making your data incomplete."

**→ Slide 16 — Standalone or Clustered?**
"Standalone is ideal for smaller telemetry volumes - simple and lightweight. Clustered deployments give you horizontal scalability and high availability at higher volumes. Instead of scaling one instance vertically, you add nodes."

**→ Slide 17 — Clustering Isn't Always Better — But Dynamic Sampling Still Wins**
"Clustering isn't always strictly better, though: each node in a cluster only sees the traffic it receives, so dynamic samplers get a less complete view to build their keyspace from. Dynamic sampling is still the best sampler for data that changes as your app changes; it just works best with a full view of traffic."

**→ Slide 18 — Resource Planning**
"Whichever you choose, plan for memory, CPU, and network bandwidth. **Memory** is the workhorse; spans in a trace don't all arrive at once, so Refinery holds them until it has enough to decide, plus queues for ingest, peer routing, and upstream sending. It's probably the most important resource to plan for. **CPU** handles ingest and sampling decisions; if ingest queues back up while CPU isn't fully used, that's a sign to add nodes, not cores. **Network** bandwidth covers both inbound telemetry and, in a cluster, peer traffic between nodes forwarding parts of traces to the right place; don't underestimate that internal traffic."

**→ Slide 19 — Recap** (Refinery Architecture)
"Standalone for low volume, clustered for scale, but plan memory, CPU, and bandwidth either way, and remember dynamic sampling and trace completeness both want the full trace on one node."

**→ Slide 20 — How Refinery Processes Data** (title, brief)
"Let's go one level deeper on what happens after your data arrives at Refinery."

**→ Slide 21 — From Ingest to Trace Cache**
"Refinery accepts two kinds of traffic: OTLP over gRPC or HTTP (traces and logs only; no bare metrics, since Refinery only accepts telemetry that carries trace information), and Honeycomb events sent as JSON, individually or in batch. Ingested spans go into an ingest queue for light processing, then into the trace cache because spans in a trace don't all arrive at once. Refinery caches until it's reasonably sure it has the whole trace."

**→ Slide 22 — Caching Until the Trace Is Ready**
"Three things cause Refinery to pull a trace out of cache and decide. First and most important: the **root span arrives**. Refinery waits a short `SendDelay` afterward in case async spans are still coming. Second: **`TraceTimeout`**. If the root span hasn't shown up but too long has passed since the first span, Refinery decides with what it has. Third: **`SpanLimit`**. If a trace's span count exceeds the configured limit before the root span arrives, same thing. You'll recognize these three settings directly in `refinery_configs/config.yaml` in the next challenge."

**→ Slide 23 — Refinery Only Samples Whole Traces**
"Refinery always keeps or drops the entire trace, never partial spans. If you need to drop or modify individual spans, that's the OpenTelemetry Collector's job, not Refinery's."

**→ Slide 24 — Samplers, and the Rules That Combine Them**
"Two sampler categories. **Deterministic** uses only trace metadata — no other fields — similar to head sampling. **Dynamic** adjusts sample rates based on the values of specific fields in a trace. You'll use both today. You don't have to pick one sampler for all your traffic either. `RulesBasedSampler` divides traffic by conditions and applies a specific sampler once a trace matches, for example treating error traces differently than everything else."

**→ Slide 25 — Build Rules to Sample Smart**
"Here's an example rule set: keep anything with an error; deterministically keep 1-in-5 of long-duration root spans; drop health checks entirely; dynamically sample normal successful traffic by service name and route; catchall everything else with a higher dynamic goal rate, since the interesting stuff was already caught earlier."

**→ Slide 26 — Recap** (How Refinery Processes Data)
"Refinery ingests OTLP and Honeycomb events, caches spans until the trace looks complete (or hits a timeout/limit), makes an all-or-nothing trace-level decision using deterministic or dynamic samplers via rules, and caches that decision so late-arriving spans get treated consistently."

**Bridge:** "Let's go look at a real pipeline built exactly this way, already running in your sandbox."

## Instruqt Challenge 1: Explore Your Refinery Pipeline (~20 min)

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Locate and read the pre-provisioned `docker-compose.yaml`, collector config, and Refinery `config.yaml`/`rules.yaml` to understand what's already wired together
- Create their own personal Honeycomb environment and connect the pipeline to it with a real API key
- Run a baseline Honeycomb query (`app.function`/`app.endpoint`) against live, already-sampled data to confirm the pipeline is working

**→ Slide 27 — Instruqt Challenge 1: Explore Your Refinery Pipeline**

**What to do before they start:** "Your sandbox is already running a full telemetry pipeline: two load generators sending synthetic traffic through a Collector, into Refinery, and on to Honeycomb. You won't build any of this. Your job is to understand what's running, then connect it to your own Honeycomb environment."

**Opening question:** "Without opening anything yet... based on what we just covered, what do you expect `docker compose ps -a` to show you, and which of those services do you expect to still be running versus already finished?"
- Listen for: recognizing `otel-collector`/`refinery` as long-running, and `loadgen1`/`loadgen2` as a finite burst — this primes them not to panic when a loadgen shows `Exited (0)`.

**Visual:** Terminal tab → `docker compose ps -a`.

**What to say:**
- "Walk through `docker compose ps -a` output: `otel-collector` and `refinery` should show `Up`; `loadgen1`/`loadgen2` may show `Up` or `Exited (0)` depending on timing — both are fine, they're a fixed ~2-minute burst. `./run` restarts just the load generators for a fresh burst."
- "Read the pipeline configuration together: `docker-compose.yaml`'s `refinery` service block (mounts `config.yaml` and `rules.yaml`), `refinery_configs/rules.yaml` (the `__default__` sampler — currently a flat `DeterministicSampler`), `collector_configs/otelcol-config.yaml` (exporters send to `refinery`, not directly to Honeycomb)."
- "Each learner needs their **own** Honeycomb environment and API key; there's no shared credential. Let's walk through signup/login, creating a `refinery-workshop` environment, and copying its API key from the 'Send data...' page on 'Home`."
- "Editing `.env` alone does nothing until the containers restart — `./stop && ./run` picks up the new key."

**While circulating:**
- Seeing a `401 response for AuthInfo request` or `check your API key` error in `docker compose logs refinery`? That's almost always the placeholder key still in `.env`, or the pipeline wasn't restarted after editing it.
- Loadgen1/2 showing `Exited (0)`? That's expected, not broken — reassure them and offer `./run` if they want fresh live traffic right now.
- Did they create their **own** environment (`refinery-workshop`), not just reuse a default/shared one? Double-check before they copy the API key.
- In their Honeycomb query, did they simplify down to just `GROUP BY: app.function` after confirming `app.endpoint` too? That simplified view is what they'll keep coming back to in later challenges.

**Closing question:** "What sampler and `SampleRate` is currently configured in `rules.yaml`? Given that, roughly what fraction of traces are actually reaching Honeycomb right now?"
- Listen for: `DeterministicSampler`, `SampleRate: 10` → about 1 in 10.

**Checkpoint**: all four services appear via `docker compose ps -a` with `otel-collector`/`refinery` `Up`; `.env` has a real API key, not the placeholder; attendees can state the current sampler/rate; a Honeycomb query grouped by `app.function` shows three distinct values with live data in the last 10 minutes.

**Bridge:** "Take 10 minutes, then we'll talk about how to actually shape what Refinery keeps."

## Break (10 min, no slides)

## Refinery Configuration & Rules Files (Lecture, 15 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Read the structure of a Refinery `config.yaml` and `rules.yaml` (`RulesVersion`, `Samplers` by environment, `RulesBasedSampler`, `Conditions`, deterministic sampler)

**→ Slide 28 — Refinery Configuration and Rules Files**
"Let's explore Refinery's configuration options. These give you control over how telemetry is processed before it reaches Honeycomb. At the heart of it is the rules file: it defines your sampling logic: what to keep, what to drop, and under what conditions. Refinery uses YAML for both."

**→ Slide 29 — Sample Configuration Options** (brief)
"Here's a sample `config.yaml`. We won't walk every line; the next slide breaks down what these options actually control."

**→ Slide 30 — Key Configuration Options**
"**Networking:** Refinery listens on up to three ports: HTTP, gRPC (in `GRPCServerParameters`, not enabled by default), and peer traffic, only needed in a clustered deployment. **Memory & Buffering:** `AvailableMemory` and `MaxMemoryPercentage` control how much memory goes to the trace cache; queues and buffers control how many spans can back up before Refinery starts dropping or blocking. **Trace Processing:** this is `SendDelay`, `TraceTimeout`, and `SpanLimit` again; the same three triggers from earlier, now as real config keys. **Monitoring & Debugging:** `RefineryTelemetry.AddRuleReasonToTrace` adds metadata showing which rule handled each trace... you'll rely on this directly in Challenge 2. Refinery can also send its own metrics to Honeycomb via the Refinery Operations Board template."

**→ Slide 31 — Refinery Rules**
"Here's a rules file targeting a specific Honeycomb environment: keep everything with `status_code >= 500` at `SampleRate: 1`, dynamically sample the rest. `RulesVersion` must be 2. `Samplers` are grouped by Honeycomb environment name or `__default__` for anything that doesn't match a named one. If an environment is explicitly listed, only its own rules apply to it; `__default__` doesn't backstop it."

**→ Slide 32 — How Rules Match**
"`Rules` is an ordered array. Refinery checks top to bottom; the first match wins and no further rules are tested. Every rule should have conditions except optionally the last one, a rule with no conditions matches everything, so it should be your catchall, placed last, and there should only be one. Conditions are ANDed; all must be true. Operators include `=`, `!=`, `<`, `<=`, `>`, `>=`, `starts-with`, `contains`, `exists`, and more; `Datatype` casts values, useful when instrumentation sometimes sends numbers as strings."

**→ Slide 33 — What a Rule Does With a Match**
"Per rule: a plain `SampleRate` integer for deterministic behavior, `Drop: true` to drop everything matching, or a `Sampler` block for something more sophisticated, which is exactly what we'll cover next lecture."

**→ Slide 34 — Recap**
"Refinery gives you control over how telemetry is sampled — what's kept, what's dropped, and under what conditions — using `RulesBasedSampler` with ordered, conditional rules. Rules can route matching traffic to a dynamic sampler too. Refinery configuration and rules are complex... the docs at docs.honeycomb.io are worth a look after today if you want the full detail."

**Bridge:** "You've seen the shape of a rules file. Now you'll write one yourself, against a real pipeline that's already sending you live data."

## Instruqt Challenge 2: Using Rules to Change How Data Is Sampled (~30 min)

**→ Slide 35 — Instruqt Challenge 2: Using Rules to Change How Data Is Sampled**

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Update `rules.yaml` to keep all traces with `status_code >= 500` plus a deterministic catchall
- Interpret the change via a calculated field, `meta.refinery.reason`, BubbleUp, and Honeycomb Usage Mode

**Opening question:** "Right now everything samples flat at 1-in-10, errors included. What's the risk of that, concretely?"
- Listen for: a real error could get dropped just as easily as a routine successful request (that's the motivating problem for this challenge).

**Visual:** Refinery Sample Application tab → `refinery_configs/rules.yaml`, then Honeycomb tab.

**What to say:**
- "Replace `rules.yaml` with the two-rule version: `SampleRate: 1` for `http.response.status_code >= 500`, `SampleRate: 10` catchall for everything else. **YAML is picky about indentation** — `Conditions` and its `-` list items need to line up exactly, or Refinery fails to load the file. This is the single most common thing to go wrong here."
- "Restart via `./run` so Refinery picks up the new rules."
- "Build the `if 500 true` calculated field, then visualize the difference: `WHERE app.function exists`, `GROUP BY` the calculated field (removing `app.function` left over from Challenge 1), `VISUALIZE COUNT, HEATMAP(duration_ms)`. 500-status traces should now show up consistently, and the duration heatmap should look smoother, not randomly missing data the way flat 1-in-10 sampling could."
- Confirm which rule matched via `meta.refinery.reason` — swap it in for the calculated field in `GROUP BY`, drop the heatmap, keep `COUNT`. Two values should appear, one per rule.
- BubbleUp: hover a point on the error line, select BubbleUp Outliers, search `app.` to focus on relevant attributes.
- Usage Mode: swap `GROUP BY` to `app.function`, add `AVG(Sample Rate)`. `clock` and `bulb` stay flat at 10 (no errors); `drawer` blends down to about 9.4, since it sent some `SampleRate: 1` error data alongside its `SampleRate: 10` normal data.

**While circulating:**
- Ask to see their `Conditions` block specifically if they're having issues; indentation mistakes here are the #1 cause of 'nothing changed after I restarted.'
- Is `GROUP BY` actually swapped, not just added to? The query builder carries state forward between steps; remind them to remove the prior field, not stack the new one alongside it.
- Seeing only one value under `meta.refinery.reason`? Check that the 500-rule's `SampleRate` is 1 and the catchall's is 10, not accidentally swapped.
- In Usage Mode, did they remove `meta.refinery.reason`/`http.response.status_code` from the last step before replacing them with `app.function`? Same carry-over issue.

**Closing question:** "`clock` and `bulb` sit flat at 10. `drawer` dips to about 9.4. Why does only `drawer`'s number move?"
- Listen for: all the 500-errors happened on `drawer`, so its blended average reflects the mix of `SampleRate: 1` (error) and `SampleRate: 10` (normal) data it sent; `clock`/`bulb` had no errors at all, so nothing pulls their average down.

**Checkpoint:** `rules.yaml` has the 500-status rule and catchall in the right order; grouping by `meta.refinery.reason` shows both rule names with 500s landing under "Keep all…"; Usage Mode shows `drawer` blended near 9.4 while `clock`/`bulb` stay at 10.

**Bridge:** "Rules like this work great when you know exactly what you're targeting: a status code, an error flag. But what about everything else... normal, healthy traffic you still want representative coverage of, without hand-writing a rule for every case?"

## What Is Dynamic Sampling? (Lecture, 15 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Explain how dynamic sampling adjusts rates based on observed key frequency to hit a goal sample rate
- Configure Refinery's `EMADynamicSampler` (`GoalSampleRate`, `FieldList`)

**→ Slide 36 — What Is Dynamic Sampling?** (title, brief)

**→ Slide 37 — Sampling That Adjusts Automatically**
"What if the telemetry you care about isn't easy to describe with simple rules... not just 'has an error,' but wanting all parts of your application, even the quiet ones, properly represented? This is where dynamic sampling comes in. Refinery's dynamic samplers adjust sample rates automatically, based on the observed frequency of specific field values in your traces. Say you're sampling by HTTP status code: you'd want to sample 200s less frequently than 500s, since 500s are a lot rarer."

**→ Slide 38 — Keys, Volume, and the Goal Sample Rate**
"Refinery builds keys from the values of selected attribute fields, then sets a sample rate per key based on its volume, aiming to meet an overall goal sample rate. A trace with only 200-status spans builds one key; a trace with both 200 and 500 status spans builds a different key."

**→ Slide 39 — How Refinery Tracks Frequency**
"Refinery keeps a running count of how often each key shows up, and uses that to calculate dynamic sample rates in real time. High-frequency keys, like `status_code=200`, get a high sample rate, so only a small percentage is kept. Low-frequency keys, like `status_code=500`, get a low sample rate, so more of that rare data is retained."

**→ Slide 40 — Rebalancing Without Manual Rules**
"Refinery does this continuously; if a previously rare pattern becomes common, or new patterns emerge, sample rates rebalance automatically, with no manual rule-writing."

**→ Slide 41 — The Recommended Dynamic Sampler**
"Here's the sampler that does this: `EMADynamicSampler` — EMA for Exponential Moving Average — regularly re-evaluates traffic to adjust sample rates across the keyspace. `GoalSampleRate` is the target *average* rate across all traffic through the sampler, not a rate applied to every single event. `FieldList` defines the keyspace, the fields whose combined values become your keys. High keyspace cardinality tends to push the effective sample rate lower than you want, since Refinery tries to give every key at least some representation."

**→ Slide 42 — Recap**
"Common values get sampled less, rare or meaningful ones get sampled more, automatically and continuously, preserving coverage across your app without hand-written rules for every case."

**Bridge:** "Let's replace your deterministic catchall with exactly this kind of sampler."

## Instruqt Challenge 3: Adding a Dynamic Sampler (~35 min)

**→ Slide 43 — Instruqt Challenge 3: Adding a Dynamic Sampler**

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Replace the deterministic catchall with `EMADynamicSampler` (`GoalSampleRate` + `FieldList`)
- Verify the sampler is hitting its goal using `meta.refinery.sample_key` in Usage Mode

**Opening question:** "You're about to replace a flat `SampleRate: 10` catchall with a dynamic sampler aiming for the same goal of 10. What would you expect to look basically the same, and what would you expect to actually change?"
- Listen for: overall volume should look similar since the goal rate matches; what changes is that low-volume traffic gets sampled less aggressively than high-volume traffic, instead of everything getting the same flat rate.

**Visual:** Refinery Sample Application tab → `refinery_configs/rules.yaml` and `docker-compose.yaml`, then Terminal, then Honeycomb.

**What to say:**
- "Replace the catchall rule with `EMADynamicSampler`, `GoalSampleRate: 10`, `FieldList: [app.function]`. Keep the 500-status rule as-is."
- "Add a third load generator (`loadgen3`) to `docker-compose.yaml`, duplicating an existing loadgen service block."
- "Create `loadgen_configs/loadgen_config3.yaml` via `touch` in the Terminal tab, **then switch to the Code Editor tab and click the file nav panel's reload icon** — a new file created externally via Terminal doesn't show up in the Code Editor's file tree until you manually refresh it. Paste in the provided config."
- "`./run` to start all three loadgens plus the collector and Refinery."
- "Explore in Honeycomb: you may still be in Usage Mode from the end of Challenge 2. Switch back to Normal Mode first. `WHERE app.function exists` / `GROUP BY app.function` should still be set from Challenge 2; if not, set them back. Four functions should now be visible: two shared by all loadgens (highest volume), one shared by two (mid volume), one only from `loadgen3` (lowest volume)."
- "In Usage Mode, grouping by `meta.refinery.reason` with `AVG(Sample Rate)` shows the dynamic rule landing close to its goal of 10. This works well because `app.function` alone is moderate cardinality."
- "Investigate the keyspace: filter to the dynamic rule specifically, group by `meta.refinery.sample_key`, visualize `SUM(Sample Rate)`. The least-busy function (`skin`) sent 6,784 events at an average sample rate of 4.5 (~30,000 spans); the busiest (`drawer`) sent about 118,000 spans, despite the volume gap, the relative sample size stayed close, because low-volume keys got sampled more gently and high-volume keys more aggressively."

**While circulating:**
- Created `loadgen_config3.yaml` via `touch` but can't find it in the Code Editor? Remind them to click the file nav panel's reload icon — this doesn't auto-refresh.
- Still in Usage Mode from last challenge, or seeing an unexpected `WHERE`/`GROUP BY`? Query state carries over between challenges since this is one continuous session; walk them back to Normal Mode and the right `app.function` grouping.
- Seeing only two or three `app.function` values instead of four? Give it the full two minutes for `loadgen3` traffic to start landing before troubleshooting further.
- Remind anyone reaching for `./stop` that this environment needs to keep running into Challenge 4.

**Closing question:** "This hit your goal sample rate almost exactly. What field did you use for the keyspace, and roughly how many unique values does it have?"
- Listen for: `app.function`, and a small number of unique values (four, in this pipeline); set up the contrast for what happens when that number gets much bigger.

**Checkpoint:** `rules.yaml` keeps the 500-status rule and replaces the catchall with `EMADynamicSampler` (`GoalSampleRate: 10`, `FieldList: [app.function]`); `loadgen3` is running alongside `loadgen1`/`loadgen2`; grouping by `app.function` shows four distinct values; Usage Mode shows the dynamic rule near its goal of 10, with high-volume keys sampled more aggressively than the lowest-volume one.

**Bridge:** "Now let's talk about what happens when that number of unique values gets big."

## Sampling and Fidelity Tradeoffs (Lecture, 10 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Explain why high-cardinality fields undermine dynamic sampling, and when full-fidelity (no sampling) or targeted tail-sampling rules are the better call instead

**→ Slide 44 — Sampling and Fidelity Tradeoffs** 
"Dynamic sampling sounds powerful, but what happens when the field you care about has really high cardinality? Take something like `user_id`. Even fifty thousand users is an enormous number of unique values to manage in a dynamic sampling keyspace."

**→ Slide 45 — High Cardinality Isn't Always a Problem**
"Here's the key: high-cardinality data is often already well-represented in sampled data, as long as you're not undersampling. A ten-second slice might miss most users; a one-minute slice probably won't, within a small margin of error. So you don't need to base dynamic sampling on high-cardinality fields like `user_id` — doing so mostly just causes the sampler to keep far more traffic than you intended, without actually reducing volume."

**→ Slide 46 — Choosing FieldList Fields**
"Do use mid-cardinality fields, like region and status code, that represent the type of traffic you want distinctly sampled. Don't use high-cardinality fields like user ID or session ID... that causes the dynamic sampler to keep far more traffic than intended, defeating the point of sampling."

**→ Slide 47 — What If You Really Need High Fidelity?**
"What if you genuinely need high fidelity for a high-cardinality field  for say, payment transactions? Sampling is a cost-saving tradeoff. If the value of full fidelity outweighs the cost, don't sample that data at all. The better question isn't 'do I need high fidelity everywhere,' it's 'do I need it for this specific type of traffic'. Payment traffic is high-value, because that's where customers give you money."

**→ Slide 48 — Strategic Fidelity**
"That's where targeted tail-sampling rules come in: rules that keep payment traffic completely (or nearly so), while dynamic sampling handles the rest, where lower fidelity is an acceptable tradeoff."

**→ Slide 49 — Recap**
"High-cardinality fields don't always need to drive dynamic sampling; time usually captures enough variety without the cost. Where you truly need high fidelity, don't sample, or write a targeted rule instead. Strategic sampling means dynamic for general traffic, high-fidelity only where the business value justifies it."

**Bridge:** "Let's go break it on purpose."

## Instruqt Challenge 4: High Cardinality in Dynamic Samplers (~35 min)

**→ Slide 50 — Instruqt Challenge 4: High Cardinality in Dynamic Samplers**

**Learning objectives:** By the end of this challenge, attendees will be able to:
- Add a high-cardinality field (`app.user_id`) to the `FieldList` and observe the sampler fail to hit its goal (keyspace exceeds the 500-key default max)
- Explain why, and connect it back to the fidelity tradeoffs lecture

**Opening question:** "Based on the last lecture, what do you think happens if we add `app.user_id`, a randomly generated, essentially-unique value, to the `FieldList` you just built?"
- Listen for: predictions that the keyspace will explode and the sampler won't be able to hit its goal rate anymore. 

**Visual:** Refinery Sample Application tab → `refinery_configs/rules.yaml`, then Honeycomb Usage Mode.

**What to say:**
- "Add `app.user_id` to the existing `FieldList` (now `[app.function, app.user_id]`) and `./run` to restart."
- "In Usage Mode, `WHERE app.function exists`, `GROUP BY meta.refinery.reason`, `AVG(Sample Rate)`, `LIMIT 1000`: the dynamic rule now shows a much lower effective sample rate than its goal of 10."
- "Investigate the keyspace: filter to the dynamic rule specifically, group by `meta.refinery.sample_key`. A large number of distinct keys appear; very few reach a sample rate near 10; many are stuck near 1."
- "Visualize keyspace size: `COUNT_DISTINCT(meta.refinery.sample_key)`, no `GROUP BY`. Well over 1,000 unique keys."
- Explain why: "`EMADynamicSampler`'s default maximum keyspace is 500 keys. Past that, new keys evict older ones, and evicted keys reset to `SampleRate: 1` when seen again — constant turnover that keeps the effective rate low and unstable, even though traffic is flowing normally."

**While circulating:**
- Confirm `FieldList` actually has both `app.function` and `app.user_id`, and that they restarted with `./run` after editing, a stale sampler won't show the failure.
- Help them read `COUNT_DISTINCT(meta.refinery.sample_key)` correctly — over 500 is the signal, and they should be seeing well over 1,000.

**Closing question:** "If you genuinely needed per-user visibility here — say, for security monitoring on login failures — would you add `user_id` back into this `FieldList`? What would you do instead?"
- Listen for: no — use a targeted rule (keep or heavily sample that specific high-value traffic) instead of forcing a high-cardinality field into the dynamic sampler's keyspace. Direct callback to the fidelity tradeoffs lecture.

**Checkpoint:** `FieldList` is `[app.function, app.user_id]`; Usage Mode shows the dynamic rule's `AVG(Sample Rate)` well below its goal of 10; grouping by `meta.refinery.sample_key` shows a large number of distinct keys, most stuck near 1; `COUNT_DISTINCT(meta.refinery.sample_key)` shows well over 500; attendees can explain why (keyspace exceeds Refinery's default 500-key limit).

**Bridge:** "So what should a real default rule set actually look like, given everything you've seen today?"

## Sensible Defaults & Wrap-up (Lecture, 20 min)

**Learning objectives:** By the end of this section, attendees will be able to:
- Recall why rule ordering matters (most-specific rules first, catchall last)
- Apply a sensible starting rule set (keep errors/500s, drop health checks, keep long-duration traces, dynamically sample the rest) as a template for their own environment

**→ Slide 51 — Sensible Default Refinery Rules** (title)

**→ Slide 52 — Rules Outlined**
"These rules aim to preserve high-value traces — like errors or 500s — where high cardinality actually matters, while dropping low-value traffic like health checks, and applying sensible dynamic sampling to everything else."
- Keep 500 status codes
- Keep traces where an error field exists
- Drop health checks
- Keep long-duration traces
- Dynamically sample good HTTP traffic (200s–400s)
- Dynamically sample gRPC traffic
- Catchall

**→ Slide 53 — Keep Errors and 500s** 
"500-level errors can have many causes, and full cardinality helps you spot patterns, the same pattern you already wrote in Challenge 2. The error-field rule overlaps with 500-level HTTP errors but also catches gRPC errors and other flagged cases like 404s/403s, a wider net than status code alone."

**→ Slide 54 — Drop Health Checks**
"Successful health-check calls, commonly from load balancers hitting every service every few seconds, are noisy and rarely worth keeping unless something's actually wrong. Adjust the route/path match per environment."

**→ Slide 55 — Keep Long Duration Traces**
"Root span duration as a proxy for the whole trace being slow. Placed after the health-check rule deliberately, a slow-but-successful health check usually isn't worth the event budget. Swap the order if your environment disagrees."

**→ Slide 56 — Dynamically Sample Good HTTP and gRPC Traffic** 
"The HTTP rule handles everything else HTTP, using `root.`-prefixed fields to keep the keyspace to root-span values only, directly applying the cardinality lesson from Challenge 4. The gRPC rule follows the same idea with gRPC-specific fields."

**→ Slide 57 — The Catchall Rule**
"Everything not matched earlier, dynamically sampled with a higher goal rate since it matters less. Notice the `root.` prefix is deliberately dropped here to capture variation across all services involved, not just the root span. Watch for cardinality if you have many microservices."

**→ Slide 58 — Recap**
"Preserve high-value traces where visibility is critical. Drop low-value noise. Apply dynamic sampling based on traffic type and cardinality. Control keyspace growth by scoping `FieldList`s deliberately."

**What to say (wrap-up):** "Today you went from a pipeline, to a rules file that protects your errors, to a dynamic sampler that adapts to real traffic, to watching that same sampler fail once cardinality gets out of hand, and a template rule set that avoids that failure on purpose. Everything here is a starting point, not a finished product; the real tuning happens against your own traffic. For more, the Honeycomb docs are the next stop."

**Bridge:** "Let's wrap it up."

## Q&A / Close (15 min)

**→ Slide 59 — What You Built Today** (workshop recap)

**What to say:**
- Recap the workshop's top-level learning objectives from `workshop/outline.md`, using slide 59's five cards as the on-screen anchor: sampling tradeoffs (head/tail/dynamic), how Refinery ingests/caches/decides, reading and modifying `rules.yaml` with deterministic and dynamic samplers, diagnosing a failing dynamic sampler from a high-cardinality `FieldList`, and applying a sensible default rule set.
- "Keep your Honeycomb API key and this environment's URL; everything you built today stays queryable for 60 days if you want to revisit it."
- **→ Slide 60 — Observability for what comes next.** Thank the room and close on this slide.
