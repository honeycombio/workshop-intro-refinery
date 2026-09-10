# Current Deck Transcript — Effective Sampling with Honeycomb Refinery Workshop

> Text-only transcript of the current 52-slide deck PDF the user shared on 2026-09-10, saved to disk so it isn't lost to context compaction. Images/visual layout not captured — see the original PDF for those. This is the "before" state; new intro/agenda/segue/outro slides are being proposed on top of this in `facilitator-guide.md` discussion.

## Slide 1 — What Is Sampling? (title)
What: Learn what sampling is.
Why: Learn why sampling is reliable.
How: Understand how sampling helps you balance cost and performance in tools like Honeycomb.

## Slide 2 — A Century-Old Idea
In 1908, a statistician at a brewery figured out you don't need to test every pint to know if the beer's good. Using a small sample, he could predict with confidence how the entire batch would turn out — a method we now call the T-Test.
Think of sampling the same way you'd think of lossy compression for images or audio: you don't have every pixel or sound wave, but you still see the picture, hear the music, and get the message. **Still useful!**

## Slide 3 — Sampling Your Telemetry
Full dataset: every event your system produces, dense and complete.
Sample kept: a smaller, representative subset. In telemetry, sampling means sending just a subset of your data instead of everything — still aiming to understand trends and patterns, just with less volume. Done right, sampling gives you a statistically useful view of your system's behavior, without drowning in data.

## Slide 4 — Won't Sampling Affect My Ability to Query My Data?
Honeycomb's Refinery solves for that. When your telemetry is sampled, metadata is added indicating how much was sampled, and Honeycomb uses that to scale your results — so queries still reflect reality, just more efficiently. The sample rate is the ratio between total events generated and events actually sent, expressed as 1/N.
Sample Rate = 1 event sent to Honeycomb ÷ Total events generated

## Slide 5 — Sample Rate 1000, In Practice
1,000× — Each event sent to Honeycomb represents 1,000 actual events.
×1,000 — COUNT and SUM results in your queries are multiplied by 1,000 to compensate.
1 of 1,000 — You're sending 1 out of every 1,000 events — the other 999 are discarded.

## Slide 6 — The Biggest Reason for Sampling: Cost
Ingesting and storing high-volume telemetry, especially with high cardinality, can get expensive fast — both economically and computationally. With smart sampling, you reduce storage and compute costs while still achieving excellent observability at lower data volumes.
Callout: Cost vs. observability, balanced.

## Slide 7 — Key Takeaways (What Is Sampling?)
Send a Slice: Sampling means sending a smaller, representative slice of your telemetry.
Cost Down, Visibility Up: Sampling reduces cost while keeping visibility.
Refinery Adjusts: Tools like Honeycomb's Refinery let you adjust for sampling, so your insights stay accurate.

## Slide 8 — Head Sampling v. Tail Sampling (title)
When you're sampling telemetry, what if you really need to make sure you capture certain data, like errors or payment transactions? That's where different sampling strategies come in — there are two main kinds: head sampling and tail sampling.

## Slide 9 — Head vs. Tail, at a Glance
Head Sampling: When — before/immediately after telemetry is generated. How — compares a trace ID against a hash. Speed/cost — fast, cheap. Awareness — uninformed.
Tail Sampling: When — after the entire trace is available. How — inspects actual attributes. Speed/cost — slower, intensive. Awareness — smart.

## Slide 10 — Why Not Both?
When trace volume is high, individual traces matter less — important patterns tend to repeat. Head sampling just 10–50% of your traffic reduces what gets passed to tail sampling, making it feasible to selectively keep the most valuable traces. Head sampling gives you speed and scale; tail sampling gives you control and precision.
Recap: Head sampling is fast and cost-effective. Tail sampling is precise but resource-intensive. Combining both balances scale and fidelity.

## Slide 11 — Refinery Architecture (title)

## Slide 12 — Deploying Refinery
Refinery Standalone Binary — Simple · Lightweight. Runs as a container on Kubernetes/ECS, or as a binary on EC2.
Refinery Node Cluster — Scalable · Fault-Tolerant. Deployed as a cluster of nodes for higher volume/resilience.
Callout: The most important rule: Refinery makes its best sampling decisions when it can see an entire trace. All services creating spans for the same trace must route to the same Refinery instance, or you risk keeping part of a trace and dropping the rest.

## Slide 13 — Standalone or Clustered?
It's really about high availability and horizontal scalability. Standalone is ideal for smaller telemetry volumes. For higher volumes, clustered deployments provide horizontal scalability and high availability — instead of scaling one instance vertically, you add nodes to distribute the load.

## Slide 14 — Clustering Isn't Always Better — But Dynamic Sampling Still Wins
In a clustered setup, each Refinery node only sees the traffic it receives, so dynamic samplers can be slightly less effective — they build their keyspace from a less complete view of traffic. Still, dynamic sampling remains the best sampler type for data that changes as your app changes: it adjusts automatically, without needing constant rule updates.

## Slide 15 — Resource Planning
Memory: the workhorse — holds spans until enough of a trace has arrived; also queues, telemetry cache, decision caches. Probably the most important resource.
CPU: handles ingest and sampling decisions. Multithreaded but doesn't scale endlessly — ingest queues backing up while CPU is idle means add nodes, not just cores.
Network: two kinds of bandwidth — ingesting telemetry from services, and (in a cluster) forwarding trace parts between peer nodes. Don't underestimate internal cluster traffic.
Callout: Regardless of deployment choice, you need to provision the right resources — especially in a clustered environment.

## Slide 16 — Recap (Refinery Architecture)
Container or Binary: Refinery deploys as a container or binary, standalone or clustered.
Scale Needs Planning: Standalone is simple for low volume; clustering adds scale but needs memory, CPU, and network planning.
One Node, Whole Trace: Every setup requires getting the full trace to a single node.

## Slide 17 — How Refinery Processes Data (title)

## Slide 18 — From Ingest to Trace Cache
Ingested data first hits an ingest queue for light processing, then flows to the Trace Cache.
OTLP — gRPC on a separate port from HTTP; /v1/traces and /v1/logs. Metrics not accepted.
Honeycomb Events — JSON with a trace ID. /1/events/{datasetSlug} for individual events, or /1/batch/{datasetSlug} for batched events.

## Slide 19 — Caching Until the Trace Is Ready
Spans get sent when the span ends, not when the trace ends — so Refinery caches until it's reasonably sure it has the whole trace. Three things trigger a decision:
1. Root Span Arrives — most important. Refinery waits a few extra seconds (SendDelay) in case more spans arrive.
2. Trace Timeout — too much time has passed since the first span, with no root span yet.
3. Span Limit — span volume exceeds the configured threshold before the root span arrives.

## Slide 20 — Refinery Only Samples Whole Traces
Refinery always keeps or drops the entire trace — never just part of it. To drop or modify individual spans, use the OpenTelemetry Collector instead.
Callout: All or nothing. Whole trace, every time.

## Slide 21 — Samplers, and the Rules That Combine Them
Deterministic — uses only trace metadata (trace ID/state); ignores field values.
Dynamic — adjusts sample rates based on how often specific field values appear.
Callout: You don't have to pick just one. The RulesBasedSampler lets you divide traffic by conditions and apply a specific sampler only when a trace matches.

## Slide 22 — Build Rules to Sample Smart
Errors, Always Kept: if any span has an error, keep the trace.
Long Duration: if the root span runs long (e.g. >5s), deterministically keep 1 in 5.
Health Checks Dropped: if it's a load-balancer health check, drop it completely.
Normal Traffic: successful HTTP traffic — dynamically sample by service name + route (goal: 1 in 10).
Catchall: dynamically sample by service name (goal: 1 in 25).

## Slide 23 — Recap (How Refinery Processes Data)
Ingest, Then Decide: Refinery ingests OTLP/Honeycomb events into a trace cache, deciding once it has the full trace (or enough, per timeouts/limits).
Decisions Are Per-Trace: decisions happen at the trace level via deterministic or dynamic samplers, often guided by ordered rules.
Consistent, Then Forwarded: once decided, Refinery caches the decision (so late-arriving spans get the same outcome) and forwards only kept traces to Honeycomb.

## Slide 24 — Refinery Configuration and Rules Files (title)

## Slide 25 — Sample Configuration Options
```yaml
Network:
  ListenAddr: "0.0.0.0:8080"
  PeerListenAddr: "0.0.0.0:8081"
GRPCServerParameters:
  Enabled: true
  ListenAddr: "0.0.0.0:9090"
Logger:
  Type: stdout
  Level: warn
RefineryTelemetry:
  AddRuleReasonToTrace: true
Collection:
  AvailableMemory: 3GiB
  MaxMemoryPercentage: 75
Traces:
  SendDelay: 5s
  TraceTimeout: 60s
  SpanLimit: 2_500
```

## Slide 26 — Key Configuration Options
Networking: Network.ListenAddr (HTTP), GRPCServerParameters (gRPC, off by default), PeerListenAddr (cluster peer traffic only).
Memory & Buffering: AvailableMemory (should be lower than total system memory), MaxMemoryPercentage (share for Trace Cache), plus incoming/peer queues and buffers that drop spans or slow Refinery down when full.
Trace Processing: SendDelay, TraceTimeout, SpanLimit.
Monitoring & Debugging: RefineryTelemetry adds rule-decision metadata to spans; Debugging.AdditionalErrorFields helps diagnose dropped spans; OTelMetrics sends Refinery's own operational metrics to Honeycomb.

## Slide 27 — Refinery Rules
```yaml
RulesVersion: 2
Samplers:
  refinery-academy:
    RulesBasedSampler:
      Rules:
        - Name: Keep all when status_code >= 500
          SampleRate: 1
          Conditions:
            - Field: http.response.status_code
              Operator: ">="
              Value: 500
              Datatype: int
        - Name: Sample the rest dynamically
          Sampler:
            EMADynamicSampler:
              GoalSampleRate: 15
              FieldList:
                - url.full
```
RulesVersion must be 2. Samplers groups rules by Honeycomb environment name (exact match required) — or use __default__ for anything unmatched. An environment listed explicitly only follows its own rules; __default__ doesn't layer on top.

## Slide 28 — How Rules Match
```yaml
Conditions:
  - Field: http.response.status_code
    Operator: ">="
    Value: 500
    Datatype: int
```
Rules are an ordered array — Refinery checks them in order, and once a trace matches, it stops checking further rules. Only the last rule may skip conditions (acting as a catchall). Conditions are ANDed — all must be true to match. Operators available: =, !=, <, <=, >, >=, starts-with, contains, does-not-contain, in, not-in, exists, not-exists, has-root-span, matches. Datatype casts values (string/int/float/bool).

## Slide 29 — What a Rule Does With a Match
```yaml
- Name: Keep all when status_code >= 500
  SampleRate: 1
- Name: Drop all health-checks
  Drop: true
- Name: Sample the rest dynamically
  Sampler:
    {Sampler Name}:
      {Sampler Configuration}
```
SampleRate (integer) does deterministic sampling — keep 1 trace per {value} received; SampleRate: 1 keeps all. Drop: true drops everything matching. Or route to a dynamic sampler via Sampler.

## Slide 30 — Recap (Refinery Configuration and Rules Files)
Refinery gives you control over how telemetry is sampled — what's kept, what's dropped, and under what conditions — using RulesBasedSampler with ordered, conditional rules. Rules can route matching traffic to a dynamic sampler too, which we'll cover next. Refinery configuration and rules are complex — see docs.honeycomb.io for the full detail.

## Slide 31 — What Is Dynamic Sampling? (title)

## Slide 32 — Sampling That Adjusts Automatically
What if the telemetry you care about isn't easy to describe with simple rules — like 'the trace has an error'? Maybe you want every part of your application, even the quiet ones, properly represented. This is where dynamic sampling comes in: Refinery adjusts sample rates automatically, based on the observed frequency of specific field values in your traces. You'd want to sample HTTP 200s less frequently than 500s, since 200s are far more common.

## Slide 33 — Keys, Volume, and the Goal Sample Rate
Refinery builds keys from the values of selected attribute fields, then sets a sample rate per key based on its volume — aiming to meet an overall goal sample rate.
Trace A (only 200 status codes) → Key A: 200, Get
Trace B (200 + 500 status codes) → Key B: 200*500, Get

## Slide 34 — How Refinery Tracks Frequency
Refinery keeps a running count of how often each key shows up, and uses that to calculate dynamic sample rates in real time. High-frequency keys (like status_code=200) get a high sample rate, so only a small percentage is kept. Low-frequency keys (like status_code=500) get a lower sample rate, so more of that rare data is retained.

## Slide 35 — Rebalancing Without Manual Rules
Adjusts in Real Time: dynamic sampling adjusts rates in real time based on field-value frequency.
Common vs. Rare: common values are sampled less; rare or meaningful ones are sampled more — automatically.
Coverage, No Hand-Coding: this preserves coverage across your app without hand-coded rules.
Callout: As traffic changes, sampling logic adapts too — no manual rule-writing needed.

## Slide 36 — The Recommended Dynamic Sampler
```yaml
Rules:
  - Name: Keep all when status_code >= 500
    # rule configuration
  - Name: Drop all health-checks
    # rule configuration
  - Name: Sample the rest dynamically
    Sampler:
      EMADynamicSampler:
        GoalSampleRate: 10
        FieldList:
          - app.function
          - app.endpoint
```
EMADynamicSampler (Exponential Moving Average) regularly re-evaluates traffic to adjust rates across the keyspace — the unique values across a trace's spans for the fields in FieldList. High keyspace cardinality usually means a lower-than-desired effective sample rate, since Refinery tries to sample every key at least a little. GoalSampleRate is the target average rate — high-volume keys land above it, low-volume keys below, averaging out near the goal.

## Slide 37 — Recap (What Is Dynamic Sampling?)
Watching Frequency: dynamic sampling watches the frequency of field values across your traffic.
Keyspace & Goal Rate: Refinery's EMADynamicSampler builds a keyspace from your FieldList and aims for a GoalSampleRate.
Field Choice Matters: choosing the right fields for that FieldList matters — you'll configure this yourself next.

## Slide 38 — Sampling and Fidelity Tradeoffs (title)
Dynamic sampling sounds powerful, but what happens when the field you care about has really high cardinality?

## Slide 39 — High Cardinality Isn't Always a Problem
Take something like user_id. Even with only fifty thousand users, that's an enormous number of unique values for a dynamic sampling keyspace. Here's the key: high-cardinality data is often already well-represented in sampled data, as long as you're not undersampling. Look at a ten-second slice of five thousand active users and you might miss some. Look at a one-minute slice and you probably won't, within a small margin of error.

## Slide 40 — Choosing FieldList Fields
Do: Use mid-cardinality fields (region, status code) that represent the type of traffic you want distinctly sampled.
Don't: Use high-cardinality fields (user ID, session ID) — this causes the dynamic sampler to keep far more traffic than intended, defeating the point of sampling.

## Slide 41 — What If You Really Need High Fidelity?
Maybe you're tracking payment transactions or login failures for security monitoring. Sampling is a cost-saving tradeoff — if the value of full fidelity outweighs the cost of storing and processing that data, don't sample it. But the better question usually isn't 'do I need high fidelity?' — it's 'do I need it everywhere, or only for certain types of traffic, like payment processing, because that's where customers give you money?'

## Slide 42 — Strategic Fidelity
High-value paths (payments) → Tail sampling (sample heavily, or keep completely)
Low-value paths (less critical traffic) → Dynamic sampling
→ Balanced sample set. Rather than relying only on dynamic sampling for all your traffic, create rules that ensure specific high-value traffic gets sampled heavily or kept completely, and use the dynamic sampler for everything with a less stringent need for fidelity.

## Slide 43 — Recap (Sampling and Fidelity Tradeoffs)
Cardinality Isn't Destiny: high-cardinality fields like user_id don't always need to drive dynamic sampling — sampling over time typically captures enough variety without added cost.
Target Your Fidelity: if you truly need high fidelity for specific traffic (payments, security events), consider not sampling it, or apply targeted tail-sampling rules instead.
Sample With Intent: strategic sampling means dynamic sampling for general traffic, reserving high-fidelity approaches for where the business value justifies the cost.

## Slide 44 — Sensible Default Refinery Rules (title)
Keep errors and long duration traffic, drop unwanted endpoints, and use dynamic samplers for your ideal traffic. Now that you've got a solid handle on how Refinery is configured — and how rules shape your sampling — here's a set of example rules you can use as a starting point, aligned with common business goals.

## Slide 45 — Rules Outlined
- Keep 500 status codes
- Keep traces where an error field exists
- Drop health checks
- Keep long duration traces
- Dynamically sample good HTTP traffic (200s through 400s)
- Dynamically sample gRPC traffic (status_code < 2) — **known bug: actual YAML on slide 49 uses Value: 2 correctly now (fixed from earlier draft's Value: 0); this slide's own bullet text still says "< 2" which is correct, but double check against slide 49's YAML at presentation time**
- Have a catchall rule for anything else

## Slide 46 — Keep Errors and 500s
```yaml
- Name: Keep 500 status codes
  SampleRate: 1
  Conditions:
    - Field: http.response.status_code
      Operator: '>='
      Value: 500
      Datatype: int
- Name: Keep where error field exists
  SampleRate: 1
  Conditions:
    - Field: error
      Operator: exists
```
500-level errors can have many causes, and full cardinality helps you spot patterns. The error field rule overlaps with 500s but also catches gRPC errors and other flagged cases like 404s/403s.

## Slide 47 — Drop Health Checks
```yaml
- Name: drop health checks
  Drop: true
  Scope: span
  Conditions:
    - Field: root.http.route
      Operator: starts-with
      Value: /healthz
    - Field: http.response.status_code
      Operator: "="
      Value: 200
      Datatype: int
```
Successful health checks usually come from load balancers hitting every service every few seconds — noisy, not worth keeping unless something's wrong. Adjust per environment.

## Slide 48 — Keep Long Duration Traces
```yaml
- Name: Keep long duration traces
  SampleRate: 1
  Scope: span
  Conditions:
    - Field: trace.parent_id
      Operator: not-exists
    - Field: duration_ms
      Operator: ">="
      Value: 5000
      Datatype: int
```
Uses the root span's duration as a proxy for the whole trace. Placed after the health-check rule, since a slow-but-successful health check usually isn't worth the event budget.

## Slide 49 — Dynamically Sample Good HTTP and gRPC Traffic
HTTP:
```yaml
- Name: Dynamically Sample 200s through 400s
  Conditions:
    - Field: http.response.status_code
      Operator: ">="
      Value: 200
      Datatype: int
  Sampler:
    EMADynamicSampler:
      GoalSampleRate: 10
      FieldList:
        - root.service.name
        - root.http.route
        - root.http.method
```
gRPC:
```yaml
- Name: Dynamically Sample Non-HTTP Request
  Conditions:
    - Field: rpc.grpc.status_code
      Operator: "<"
      Value: 2
      Datatype: int
  Sampler:
    EMADynamicSampler:
      GoalSampleRate: 10
      FieldList:
        - root.service.name
        - grpc.method
        - grpc.service
```
**Note: the gRPC YAML here already reads `Value: 2` — the Rule 6 value bug flagged from the source deck transcript appears fixed in this current deck.** No upper-bound condition is needed for status codes below 500 — the earlier rule already caught those.

## Slide 50 — The Catchall Rule
```yaml
- Name: Catchall rule
  Sampler:
    EMADynamicSampler:
      GoalSampleRate: 20
      FieldList:
        - service.name
```
Handles anything the earlier rules didn't match, with a higher goal sample rate since this traffic matters less. The root. prefix is intentionally dropped here to capture key variation across all services involved.

## Slide 51 — Recap (Sensible Default Refinery Rules)
1. Preserve high-value traces (errors, long durations) where visibility is critical.
2. Drop low-value traffic (successful health checks) to reduce noise.
3. Apply dynamic sampling based on traffic type, cardinality, and trace characteristics.
4. Control keyspace growth by scoping FieldList — use root. when appropriate, or leave it off when full trace visibility is more useful.

## Slide 52 — Closing brand slide
"Observability for what comes next" — honeycomb.io
