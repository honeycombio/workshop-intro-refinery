# Sensible Default Refinery Rules (Slide Deck)

Source: "Activity 4: Sensible Defaults.pdf" — Section 3, Activity 4 deck

Title: **Sensible default Refinery rules** — Keep errors and long duration traffic, drop unwanted endpoints, and use dynamic samplers for your ideal traffic

## Slide: Rules outlined
- Keep 500 Status Codes
- Keep traces where an error field exists
- Drop health checks
- Keep long duration traces
- Dynamically sample good http traffic (200s through 400s)
- Dynamically sample GRPC traffic (status_code < 2)
- Have a catchall rule for anything else

Narration: "These rules aim to preserve the high-value traces—like those with errors or 500 status codes—where high cardinality matters. At the same time, we drop low-value traffic, like health checks from load balancers, and apply sensible sampling to everything else."

## Rule 1: Keep HTTP status codes in the 500's
```yaml
- Name: Keep 500 status codes
  SampleRate: 1
  Conditions:
    - Field: http.response.status_code
      Operator: '>='
      Value: 500
      Datatype: int
```
Narration: keeps any trace where at least one span has an HTTP status code of 500 or higher — worth keeping since 500-level errors can have many causes and full cardinality helps spot patterns.

## Rule 2: Keep traces with errors
```yaml
- Name: Keep where error field exists
  SampleRate: 1
  Conditions:
    - Field: error
      Operator: exists
```
Narration: keeps traces with spans marked `error` by instrumentation. Overlaps with 500-level HTTP errors but also catches gRPC errors (status code 2+) and other cases like 404s/403s if flagged. Ensures system errors don't slip through even without a 500.

## Rule 3: Drop health checks
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
Narration: drops successful health-check calls (commonly from load balancers hitting every service every few seconds) — noisy and not worth keeping unless something's wrong. May need adjusting per environment.

## Rule 4: Keep long duration traces
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
Narration: keeps traces with long-running root spans (root span duration is a proxy for the whole trace). Adjust the threshold to what "long" means in your environment. Placed after the health-check rule since a slow-but-successful health check usually isn't worth the event budget — swap order if you'd rather prioritize duration over endpoint type.

## Rule 5: Dynamically sample good HTTP traces
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
Narration: handles remaining HTTP traffic dynamically. No explicit upper bound needed for 500s since Rule 1 already captured them first. `FieldList` uses the `root.` prefix to focus on root-span values only, reducing keyspace cardinality — try removing the prefix if your traces don't have many services/methods/routes.

## Rule 6: Dynamically sample good gRPC traces
```yaml
- Name: Dynamically Sample Non-HTTP Request
  Conditions:
    - Field: rpc.grpc.status_code
      Operator: "<"
      Value: 0
      Datatype: int
  Sampler:
    EMADynamicSampler:
      GoalSampleRate: 10
      FieldList:
        - root.service.name
        - grpc.method
        - grpc.service
```
Narration: like Rule 5 but for gRPC-only traces, with a gRPC-specific FieldList. Points to OTel Semantic Conventions docs for how gRPC status codes map to error/non-error for server vs. client.

**Known discrepancy (flagged in `visual-asset-inventory` memory):** the rules-outline slide describes this rule as "status_code < 2," but the actual YAML here uses `Field: rpc.grpc.status_code, Operator: "<", Value: 0` — which would never match any gRPC status code (no negative codes exist). Likely should be `Value: 2`. Fix before this deck ships.

## Rule 7: Catchall rule
```yaml
- Name: Catchall rule
  Sampler:
    EMADynamicSampler:
      GoalSampleRate: 20
      FieldList:
        - service.name
```
Narration: catches everything not matched by earlier rules, dynamic sampler with a higher goal sample rate since this traffic matters less. `root.` prefix intentionally omitted from `FieldList` to capture key variation across all services involved (not just root span) — watch for high cardinality if you have many microservices.

## Closing recap slide
- Preserve high-value traces (errors, long durations) where visibility is critical.
- Drop low-value traffic (successful health checks) to reduce noise.
- Apply dynamic sampling based on traffic type, cardinality, and trace characteristics.
- Control keyspace growth by scoping FieldLists — use `root.` when appropriate, or leave it off when full trace visibility is more useful.
