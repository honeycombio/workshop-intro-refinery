# Video Scripts & Lab Guides — Section 2: Getting Started with Refinery

Source: "Scripts - Section 2 (3).pdf"

## Activity 1 — Refinery Architecture (Video)

**Learning Objectives:**
- Learners will explain that Refinery can be deployed in standalone or clustered mode.
- Learners will understand the driving factors behind the need for computational resources like memory, CPU, and Networking when deploying Refinery.

**Script (video narration — matches the storyboard in `refinery-architecture-storyboard.md`):**

[Audio] Let's say you're ready to deploy Honeycomb's Refinery to manage your telemetry sampling. What are your deployment options?
[Visual] Character excited. Title card: "Refinery Architecture"

[Audio] Refinery can be deployed as a standalone binary or as a cluster of nodes. It runs as a container on platforms like Kubernetes or Amazon ECS, or as a binary on a Linux system like an EC2 instance.
[Visual] Diagram: Refinery in standalone and clustered deployment modes. Icons: Docker, Kubernetes, EC2. Architecture note shown: "Honeycomb Refinery - a dynamic, tail sampling proxy service you can run on your own infra." Standalone = "Standalone service (Linux server, k8s pod, EC2 instance, etc)". Clustered = Load Balancer → Refinery nodes (peer traffic between them) + Redis (cluster membership state) → O11y Backend. Resource note: "Memory, CPU and Network throughput requirements — memory based on volume x size of tracing data needed to store in memory; CPU for processing traffic and making sampling decisions and process overhead; Network bandwidth determined by number of peers and volume of data."

[Audio] Here's the most important thing to understand: Refinery makes its best sampling decisions when it can see all of the data in an entire trace, not just individual spans. That means all services that create spans that belong to the same trace must be routed to the same Refinery instance. If you don't do this, you risk keeping part of a trace and dropping the rest—making your observability data incomplete.
[Visual] Diagram showing a full trace sent to a single Refinery instance; contrast with broken trace split across two nodes marked "incomplete."

[Audio] So, why would you run a standalone deployment instead of a clustered deployment? It's really about high availability and horizontal scalability. At a certain volume, it doesn't make sense to keep growing one instance larger and larger to handle that volume. Running Refinery as a standalone instance is ideal for smaller telemetry volumes. But if you're dealing with higher volumes, clustered deployments provide horizontal scalability and high availability. Instead of scaling one instance vertically, you add nodes to distribute the load.
[Visual] Single node vs. multi-node cluster. Labels: "Standalone" → "Simple, lightweight"; "Cluster" → "Scalable, fault-tolerant"

[Audio] Still, clustering isn't always better. In a clustered setup, each Refinery node is only aware of the traffic it receives and can only make sampling decisions on those traces. This means that dynamic samplers can be slightly less effective in a cluster because they see a less complete view of the traffic to build their dynamic sampling keyspace with.
[Visual] Cluster node with limited visibility. Callout: "Local traffic only — less dynamic context"

[Audio] So yes, clustering helps with scale, but dynamic samplers work best when they have a full view of traffic.
[Visual] Character thinking: "Fewer high-powered nodes?" "Depends on how you sample!"

[Audio] That said, dynamic sampling is still the best sampler type for sampling data that changes as your app changes. It adjusts automatically as your app evolves, without needing constant rule updates.
[Visual] Dynamic sampling icon next to app timeline. Arrow showing adjustment to new endpoints/services.

[Audio] Regardless of your deployment choice, you'll need to provision the right computational resources to run Refinery efficiently. That means planning for memory, CPU, and networking bandwidth—especially in a clustered environment. Let's walk through why each one matters.
[Visual] Three icons: Memory, CPU, Network. Title overlay: "Resource Planning"

[Audio] First up: Memory. Memory is the workhorse of Refinery. Since spans in a trace may not arrive all at once, Refinery holds spans in memory until it has enough to make a decision. It also needs memory for internal queues, storing spans during ingest, between nodes, and before forwarding telemetry onward. Recall that tail sampling is resource intensive because you have to store all spans in a trace long enough to make a decision, and then quickly drop or send the telemetry on to its next endpoint. For Refinery to do this, it requires CPU, Memory, and Networking bandwidth (particularly in clustered mode). The most important resource for Refinery is probably memory.
[Visual] Memory diagram with span buffer queues. Callouts: "Holding incomplete traces" and "Queue management." Sketch: Memory box → "Queues (ingest, peer routing, upstream sending)," "Telemetry Cache (where your span data is stored)," "Decision Caches (keep track of sampling kept/dropped decisions)"

[Audio] Next is CPU. CPU handles ingesting telemetry from your app and making sampling decisions to determine if a trace is kept or dropped. Refinery is multithreaded and can use multiple cores, but it doesn't scale endlessly. If you see ingest queues backing up while your CPU isn't fully used, that's a sign to add more nodes, not just more cores.
[Visual] Processor graphic with usage graph. Labels: "Multithreaded" and "Watch ingest queue behavior." Sketch: CPU box → "Ingest," "Sampling Decisions"

[Audio] Finally, network bandwidth. Bandwidth matters in two ways: First, your Refinery instance needs the bandwidth to ingest telemetry from your services, which can be a lot. Second, Refinery nodes in a cluster also need network bandwidth between each other to forward parts of traces to the correct node so the whole trace can be processed together.
[Visual] Two arrows: inbound telemetry, internal cluster traffic. Overlay: "Don't underestimate internal network usage." Sketch: Network box → "Ingest Traffic," "Peer Traffic"

[Audio] Let's recap. Refinery can be deployed as a container or binary, either as a single node or in a clustered architecture. For low-volume telemetry traffic, a single-node standalone is simple and effective. For high-volume workloads, clustering helps—but requires thoughtful memory, CPU, and network planning. And remember: no matter your setup, dynamic sampling and trace completeness require getting the full trace to a single node.
[Visual] Character summarizing at a whiteboard: Binary or container / Standalone for low volume / Clustered for scale / Plan for memory, CPU, and bandwidth.

**Note:** an alternate "dialogue" version of this script also exists, framed as a conversation between characters "Engie" and "HoneyBot" covering the same content beats (deployment options, trace-routing requirement, standalone vs. cluster tradeoff, dynamic sampler limitation in clusters, memory/CPU/network resource planning) ending with Engie summarizing and HoneyBot confirming.

---

## Activity 2 — How Refinery Processes Data (Video with Slide Deck)

**Format:** Video with deck (denser content)

**Learning Objectives:**
- Summarize how Refinery caches data before making decisions.
- Understand what triggers the trace decision in Refinery (root span arrival + SendDelay wait; Trace Timeout; Span Limit).
- Explain how Refinery uses all the data in a trace when making a trace decision.

**Deck: "How Refinery Processes Data"**

**Slide 1 (Intro):** "Let's dive a little bit deeper on how Refinery processes data, from ingest, to the trace cache, to making sampling decisions."

**Slide 2 (Ingesting data, title):** "Let's start with how Refinery receives data."

**Slide 3 (Refinery can ingest OTLP and Honeycomb events):** Refinery accepts two types of traffic:
- **OTLP** (OpenTelemetry Protocol) via standard endpoints. GRPC listens on a separate port from HTTP. `/v1/traces` & `/v1/logs` for sending OTLP trace data (current SDKs/Collectors append this automatically). Metrics or other data types not accepted — Refinery only accepts traces and logs that include trace information.
- **Honeycomb events** — JSON objects that include a trace ID, sent individually (`/1/events/{datasetSlug}`) or in batches (`/1/batch/{datasetSlug}`).

**Slide 4 (Ingested data goes where?):** When Refinery ingests data, it places it in an ingest queue for light processing (e.g. checking if spans belong to a trace that already had a decision), then spans go to the Trace Cache.
Diagram: Span A/B/C/D → Ingest Queue → Ingest → Trace Cache.

**Slide 5 (Spans in a trace don't arrive at the same time):** Refinery caches spans because effective tail sampling requires as much of the whole trace as possible. Thousands of spans can arrive over several seconds without completing the trace. Spans get sent when the span ends, not when the trace ends, so Refinery caches until it's reasonably sure it has the whole trace.

**Slide 6 (When telemetry comes out, title):** "Once the trace data has been cached, it needs to leave the cache and get processed."

**Slide 7 (Three reasons for a trace to leave the cache):** Three triggers cause Refinery to take a trace out of cache and decide:
1. **Root span arrives** (most important) — in most configs the root span is the last span sent and runs the full trace duration, so Refinery uses its arrival as the completion signal. Refinery waits a few extra seconds (`SendDelay`) after the root span in case more async/peer spans arrive.
2. **Trace Timeout** — if the root span hasn't arrived but it's been too long since the first span arrived, Refinery decides with what it has.
3. **Span Limit** — if a trace's span count exceeds the configured limit before the root span arrives, Refinery decides with what it has (prevents large traces from overwhelming the cache).

**Slide 8 (Sampling decisions, title):** "Once the trace is out of the cache, Refinery can finally make a sampling decision."

**Slide 9 (Refinery only samples traces):** Refinery always keeps or drops the **entire trace** — never partial spans. It's all-or-nothing; for dropping/modifying individual spans, use the OpenTelemetry Collector instead.

**Slide 10 (Samplers):** Two sampler categories:
- **Deterministic** — uses only trace metadata (trace ID/trace state), no other fields; like probabilistic sampling in the OTel Collector or head sampling in an SDK.
- **Dynamic** — adjusts sample rates based on the values of specific fields in a trace, tracking frequency of value combinations.

**Slide 11 (Refinery also has rules):** You don't have to pick one sampler for all traffic. `RulesBasedSampler` divides traffic by conditions and applies a specific sampler when a trace meets those conditions — e.g. process error traces differently than non-error traces.

**Slide 12 (Build your rules to sample smart!):** Example rule set (in order — first match wins):
1. If any span in a trace has an error, keep the trace.
2. If the root span has a long duration (>5s), deterministically keep 1 in 5.
3. If the trace is a load-balancer health check, drop it completely.
4. If it's normal traffic with successful HTTP status codes, dynamically sample by root span's service name + HTTP route, goal sample rate 1-in-10.
5. Catchall (no conditions): dynamically sample by root span's service name, goal sample rate 1-in-25 — since interesting traffic is already caught by earlier rules.

**Slide 13 (After the decision):** Late-arriving spans (after a decision was made) are handled via two caches — one for dropped traces, one for kept traces — so new spans get the same decision applied without re-entering the trace cache.

**Slide 14 (The path of your data through Refinery, recap):**
- Refinery ingests OTLP traces/logs and Honeycomb events, caching spans until it has the full trace (or enough per timeouts/span limits).
- Sampling decisions are made at the trace level using deterministic or dynamic samplers, often via rules targeting specific traffic patterns.
- Once decided, Refinery caches the decision so late spans are handled consistently; only kept traces are forwarded to Honeycomb.
Diagram: Root Span + Span A-D → Refinery → Cache Until Full Trace Collected → Process Trace Rules → Sample Yes/No → Cache Trace Decision → Honeycomb (or Drop).

---

## Activity 3 — Refinery Configuration and Rules Files (Video with Slide Deck)

See `activity3-refinery-configuration-and-rules-files.md` for the full deck transcript (title slide framing: "Let's explore Refinery's configuration options. These options give you control over how telemetry data is processed before it reaches Honeycomb. At the heart of it is the rules file—it defines your sampling logic: what to keep, what to drop, and under what conditions.").

Additional narration notes not in the deck file:
- Slide 2 narration: "Let's start with the configuration file."
- Slide 3 narration: "Refinery uses YAML as the standard for its configurations. These are some basic, commonly-used options as a sample–we'll dig a little deeper later on."
- Slide 8 narration: "Next, we'll move into configuring our Refinery rules."
- Slide 9 narration: "Refinery rules are also defined in YAML. Here's a sample rules file for Telemetry being sent to our `refinery-academy` environment."
- Closing note (course description, NOT talk track): "Once you feel comfortable with these, we highly recommend going through the documentation as you look to put Refinery into production in your environment."
- Slide 7 (Monitoring/Debugging) also recommends sending Refinery's metrics/logs to Honeycomb to use the **Refinery Operations Board template** (docs: https://docs.honeycomb.io/observe/boards/templates/#refinery-operations).

---

## Activity 4 — Build a Telemetry Pipeline for Testing (Hands-on, guided by video)

**Format:** Hands-on activity, guided by video

**Learning Objectives:**
- Use the `loadgen` tool with docker compose to send data through a collector, building a simple telemetry pipeline for testing.
- Explain how `loadgen` sends repeatable traffic volumes to see the effectiveness of future sampling with Refinery.

**Repository:** Academy-Intro-Refinery — Main Branch

### Script

**Clone the Repository**
```bash
git clone https://github.com/honeycombio/academy-intro-refinery.git
cd academy-intro-refinery
```

**Add Your Honeycomb Ingest Key** — open `.env` in repo root, paste API key:
```
HONEYCOMB_API_KEY=your-api-key-here
```

**What Does `loadgen` Do?**
1. `loadgen` is a small Go binary that generates synthetic telemetry.
2. As long as the configuration stays the same, it produces very similar traffic each time you run it.
3. This consistency makes it ideal for comparing unsampled vs. sampled data in Honeycomb.

**Docker Compose Setup** — `docker-compose.yaml` includes: two `loadgen` containers, an OpenTelemetry Collector, a Honeycomb destination. Traffic sent via OTLP over gRPC.

**`loadgen` Configurations** — each config defines: dataset name, service name, span depth, spans per trace, trace duration, trace rate (e.g. 1,000 traces/sec for 2 minutes).
*Note: `loadgen2` includes three app functions. The third function will have less traffic — this becomes important when analyzing sampling behavior later.*

**OpenTelemetry Collector Config** — `collector_configs/otelcol-config.yaml` extracts `app.function` and `app.endpoint` fields from the generated URL, making it easier to query and define sampling rules in Honeycomb and Refinery.

**Run the Environment:**
```bash
./run
```
Starts both `loadgen` instances, the OTel collector, and routes traffic to Honeycomb.

**Explore the Data in Honeycomb:**
1. Open Honeycomb UI, set time range to last 10 minutes.
2. Query: WHERE `app.function exists`, GROUP BY `app.function, app.endpoint`. Expected: three distinct `app.function` values, many more `app.endpoint` values.
3. Simplify: remove `app.endpoint`, group only by `app.function`.

**Observe Traffic Patterns:** the third function (from `loadgen2`) shows less volume — a controlled baseline to observe how sampling changes the shape of the data.

**Shut Down the Environment:**
```bash
./stop
```

**Recap:** Generated consistent trace traffic using `loadgen`, explored that data in Honeycomb, and prepared the environment to test Refinery's sampling behavior.

---

## Activity 5 — Set Up the Collector and Refinery (Hands-on, guided by video)

**Format:** Hands-on activity, guided by video

**Learning Objectives:**
- Implement a basic Refinery configuration, with a default rule for all data to go through a deterministic sampler.
- See the difference between previous data and new data by running queries in Honeycomb.
- Run queries in Usage Mode in Honeycomb to see volume changes and sample rates.

**Repository:** Academy-Intro-Refinery, branch `section2_activity5` (end-state branch)

### Script

**Set Up the Refinery Configuration Directory**
```bash
mkdir refinery_configs
touch refinery_configs/config.yaml refinery_configs/rules.yaml
```

Paste into `config.yaml`:
```yaml
General:
  ConfigurationVersion: 2
  MinRefineryVersion: v2.0
Network:
  ListenAddr: "0.0.0.0:8080"
  PeerListenAddr: "0.0.0.0:8081"
GRPCServerParameters:
  Enabled: true
  ListenAddr: "0.0.0.0:9090"
RefineryTelemetry:
  AddRuleReasonToTrace: true
  AddSpanCountToRoot: true
  AddHostMetadataToTrace: true
Debugging:
  AdditionalErrorFields:
    - service.name
    - trace.span_id
    - trace.parent_id
    - trace.trace_id
Logger:
  Type: stdout
  Level: warn
LegacyMetrics:
  Enabled: false
OTelMetrics:
  Enabled: false
```

Paste into `rules.yaml`:
```yaml
RulesVersion: 2
Samplers:
  __default__:
    DeterministicSampler:
      SampleRate: 10
```

**Add Refinery to Docker Compose** — new service block after `loadgen`:
```yaml
  refinery:
    image: honeycombio/refinery
    ports:
      - 8080:8080
      - 9090:9090
    entrypoint:
      - "refinery"
    command:
      - "-c"
      - "/etc/refinery/config.yaml"
      - "-r"
      - "/etc/refinery/rules.yaml"
    volumes:
      - ./refinery_configs/config.yaml:/etc/refinery/config.yaml
      - ./refinery_configs/rules.yaml:/etc/refinery/rules.yaml
    networks:
      - honeycomb
```
Add `depends_on: [refinery]` (in addition to existing deps) to `otel-collector`, `loadgen1`, and `loadgen2` service blocks so Refinery starts first.

**Update the OpenTelemetry Collector Configuration** — point the OTLP exporter at Refinery instead of Honeycomb directly:
```yaml
exporters:
  otlp:
    endpoint: refinery:9090
    tls:
      insecure: true
    headers:
      x-honeycomb-team: "${HONEYCOMB_API_KEY}"
  debug:
    verbosity: detailed
```
Keep the transformation logic that splits the URL into `app.function` and `app.endpoint`.

**Run the Environment:**
```bash
./run
```
Full pipeline: loadgen → OpenTelemetry Collector → Refinery → Honeycomb.

**View the Sampled Data in Honeycomb:**
1. Time range: last 10 minutes. WHERE `app.function exists`, GROUP BY `app.function, app.endpoint`. Expect three `app.function` values (one lower traffic), many `app.endpoint` values.
2. Remove `app.endpoint` from grouping, rerun.

**Switch to Usage Mode:** modify the URL by adding `/usage/` before `/result/` (without losing the query). Rerun the same query grouped by `app.function`. Widen time range, select the active area on the graph, click Zoom in (now viewing an absolute custom time range). Usage Mode shows how sampling affected the dataset's completeness/consistency.

**Add Sample Rate to the Visualization:** add `Sample Rate` to the Visualize panel and rerun — shows how each function's data is being sampled and its impact on trace volume/fidelity.

**Analyze Impact on High-Cardinality Fields:** add a `HEATMAP(duration_ms)` visualization. Higher-cardinality fields see more distortion from sampling — don't undersample.
1. Increase the deterministic sampler's `SampleRate` in `rules.yaml` to 100.
2. Restart (`./run`) and re-run the same heatmap query — data becomes noticeably **more distorted** (spiky/inconsistent heatmaps signal detail loss).

**Undersampling guidance:** If analyzing high-cardinality fields or monitoring critical business workflows — do not undersample aggressively; it can hide edge cases, anomalies, or early failure indicators, even if trends emerge over time. Ask: "Do I want to wait until a pattern is obvious, or do I need to catch it as it's happening?"

**Shut Down the Environment:**
```bash
./stop
```

**Recap:** Routed trace traffic through Refinery, applied sampling rules, and visualized the impact in Honeycomb — including how sample rates affect visibility for high-cardinality attributes.
