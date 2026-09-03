# Refinery Configuration and Rules (Slide Deck)

Source: "Activity 3: Refinery Configuration and Rules Files.pdf" — Section 2, Activity 3 deck

Title: **Refinery configuration and rules** — Important configuration options, and an introduction to Refinery's rule mechanism

## Section: Refinery configuration

### Slide: Sample configuration options
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

### Slide: Networking for receiving telemetry
```yaml
Network:
  ListenAddr: "0.0.0.0:8080"
  PeerListenAddr: "0.0.0.0:8081"
GRPCServerParameters:
  Enabled: true
  ListenAddr: "0.0.0.0:9090"
```
Refinery listens on three ports:
- HTTP Traffic (`Network.ListenAddr`)
- GRPC Traffic — in the `GRPCServerParameters` section, not enabled by default
- Peer Traffic — only needed with clustered Refinery

### Slide: Storing, queuing, and buffering
```yaml
Collection:
  AvailableMemory: 4GiB
  MaxMemoryPercentage: 75
  IncomingQueueSize: 30_000
  PeerQueueSize: 30_000
BufferSizes:
  UpstreamBufferSize: 10_000
  PeerBufferSize: 100_000
```
- `AvailableMemory`: amount of memory allocated to the Refinery process; should be lower than total system memory.
- `MaxMemoryPercentage`: amount of `AvailableMemory` for the Trace Cache.
- The Queues (Incoming and Peer): number of spans queued before being processed into the trace cache; if full, Refinery drops spans from upstream or peers.
- The Buffers (Incoming and Upstream): number of spans buffered for sending to peers or Honeycomb; if full, Refinery starts blocking threads and slows down.

### Slide: Processing traces
```yaml
Traces:
  SendDelay: 2s
  TraceTimeout: 60s
  SpanLimit: 2_500
```
- `SendDelay`: after a root span arrives, wait this long for async/late spans before making a sampling decision.
- `TraceTimeout`: if the root span hasn't arrived but it's been this long since the first span arrived, make a trace decision with available spans.
- `SpanLimit`: if the root span hasn't arrived but span volume for the trace exceeds this threshold, make a trace decision with available spans.

### Slide: Monitoring and debugging Refinery
```yaml
RefineryTelemetry:
  AddRuleReasonToTrace: true
Debugging:
  AdditionalErrorFields:
    - service.name
    - trace.trace_id
Logger:
  Type: stdout
  Level: warn
OTelMetrics:
  Enabled: true
  Dataset: refinery-metrics
  APIKey: {HNY_API_KEY}
```
- `RefineryTelemetry`: adds metadata to spans to show how Refinery processed the data and which rule was used.
- `Debugging.AdditionalErrorFields`: if Refinery drops spans with an error, add fields from the dropped span to help debug.
- `Logger`: send logs to stdout or honeycomb.
- `OTelMetrics`: Refinery can send metrics about its functioning to Honeycomb.

## Section: Refinery Rules

### Slide: Sample Refinery rules
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

### Slide: Separating rules by Honeycomb environment
```yaml
RulesVersion: 2
Samplers:
  __default__:
    DeterministicSampler:
      SampleRate: 10
  refinery-academy:
    RulesBasedSampler:
      Rules:
        - …
```
- `RulesVersion`: can only be 2 right now, must exist to be validated.
- `Samplers`: grouped by Honeycomb environment name (must match exactly, including spaces/symbols/capitalization), or `__default__` for traffic not matching a named environment. If an environment is listed under `Samplers`, its traffic only follows that environment's rules — `__default__` doesn't apply to it.

### Slide: RulesBasedSampler allows intentional sampling
The `Rules` section is an ordered array. Refinery compares trace data against rules in listed order; once a trace matches, it follows that rule's sampling instructions and is not tested against further rules. Rules should have conditions, except optionally the last one (a rule with no conditions matches everything — should be the catchall, and there should only be one such rule, placed last).

### Slide: Understanding Conditions
```yaml
refinery-academy:
  RulesBasedSampler:
    Rules:
      - Name: Keep all when status_code >= 500
        Scope: span
        Conditions:
          - Field: http.response.status_code
            Operator: ">="
            Value: 500
            Datatype: int
        SampleRate: 1
```
- Conditions list is an array; ALL conditions must be true for a match (AND logic).
- Conditions target a specific span attribute field (or `Fields` for an array of field names).
- `Operator` values: `=`, `!=`, `<`, `<=`, `>`, `>=`, `starts-with`, `contains`, `does-not-contain`, `in`, `not-in`, `exists`, `not-exists`, `has-root-span`, `matches`.
- `Value` is what the operator assesses against; `exists`/`not-exists` don't use a value.
- `Datatype` casts field values (`string`, `int`, `float`, `bool`) — useful when instrumentation sometimes sends ints as strings.

### Slide: Samplers for Rules
```yaml
refinery-academy:
  RulesBasedSampler:
    Rules:
      - Name: Keep all when status_code >= 500
        # rule conditions
        SampleRate: 1
      - Name: Drop all health-checks
        # rule conditions
        Drop: true
      - Name: Sample the rest dynamicallly
        Sampler:
          {Sampler Name}:
            {Sampler Configuration}
```
- Deterministic sampling: add `SampleRate` (integer) to a rule — Refinery keeps 1 trace per `{value}` received. `SampleRate: 1` keeps all.
- `Drop: true` drops all matching traces (defaults to `false` if unset).
- Dynamic sampler via the `Sampler` field — most common are `EMADynamicSampler` and `EMAThroughputSampler`.

### Slide: The recommended dynamic sampler
```yaml
Rules:
  - Name: Keep all when status_code >= 500
    # rule configuration
  - Name: Drop all health-checks
    # rule configuration
  - Name: Sample the rest dynamicallly
    Sampler:
      EMADynamicSampler:
        GoalSampleRate: 10
        FieldList:
          - app.function
          - app.endpoint
```
`EMADynamicSampler` (EMA = Exponential Moving Average) regularly re-evaluates traffic to adjust sample rates applied to the keyspace. The keyspace = unique values across all spans of a trace for the fields in `FieldList`. High keyspace cardinality usually means a lower-than-desired effective sample rate, because Refinery tries to sample every key at least a little. `GoalSampleRate` is the target *average* SampleRate across all traffic through the sampler — high-volume keys get assigned much higher SampleRate than the goal, low-volume keys much lower, averaging out near the goal.

### Closing slide: Please read the docs
"While this should get you started, Refinery configuration and rules are complex and there are a lot more details in our documentation at: https://docs.honeycomb.io"
