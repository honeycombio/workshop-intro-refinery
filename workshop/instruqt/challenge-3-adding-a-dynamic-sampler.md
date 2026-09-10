# Challenge 3: Adding a Dynamic Sampler

This challenge builds directly on Challenge 2. Your `rules.yaml` should still have the rule keeping all `http.response.status_code >= 500` traces, with a deterministic catchall for everything else. You'll replace that catchall with a dynamic sampler, then add a third load generator to see how it responds to a new, lower-volume function.

## Update the rules to use the EMADynamicSampler

1. Select the [button label="Refinery Sample Application"](tab-0) tab.
2. Open `refinery_configs/rules.yaml`. Keep the status-code rule that captures all HTTP responses with `status_code >= 500` and a `SampleRate` of 1. Replace the deterministic catch-all rule with a new dynamic rule:
```yaml
RulesVersion: 2
Samplers:
  __default__:
    RulesBasedSampler:
      Rules:
        - Name: Keep all when http.response.status_code >= 500
          SampleRate: 1
          Conditions:
            - Field: http.response.status_code
              Operator: ">="
              Value: 500
              Datatype: int
        - Name: Sample the rest dynamically
          Sampler:
            EMADynamicSampler:
              GoalSampleRate: 10
              FieldList:
                - app.function
```
3. The `FieldList` defines the keyspace for the sampler. Here, `app.function` is used to generate unique sampling keys. The volume of each key determines how much Refinery samples it, aiming to reach the overall target average sample rate (`GoalSampleRate: 10`).

## Add the third load generator

1. Still in [button label="Refinery Sample Application"](tab-0), open `docker-compose.yaml` and add a new service, duplicating one of the existing `loadgen` definitions:
```yaml
  loadgen3:
    image: ghcr.io/honeycombio/loadgen/loadgen
    env_file:
      - .env
    volumes:
      - ./loadgen_configs/loadgen_config3.yaml:/etc/loadgen/config.yaml
    command:
      ["--config=/etc/loadgen/config.yaml"]
    networks:
      - honeycomb
    depends_on:
      - otel-collector
      - refinery
```

## Create the third loadgen configuration

1. Select the [button label="Terminal"](tab-1) tab and create the new config file:
```bash
touch loadgen_configs/loadgen_config3.yaml
```
2. Select the [button label="Refinery Sample Application"](tab-0) tab. The new file won't show up in the file tree yet. Select the reload icon at the top of the file navigation panel. 
3. Open `loadgen_configs/loadgen_config3.yaml`. Paste in:
```yaml
telemetry:
  host: "otel-collector:4317"
  dataset: loadgen-data
  insecure: true
format:
  depth: 5
  nspans: 10
  tracetime: 1s
quantity:
  tps: 1000
  runtime: 120s
  ramptime: 1s
output:
  sender: otel
  protocol: grpc
global:
  loglevel: warn
fields:
  # simulate URLs for 10 services, each of which has 10 endpoints on the root span
  0.url.full: /u4,5
  # generate status codes where 5% are 400s and .1% are 500s on the root span
  0.http.response.status_code: /st5,0.1
  # generate a hex ID as a user_id on the root span
  0.app.user_id: /sx24
```
4. This will result in a new `app.function` value once processed by the collector. This increases cardinality and ensures that one function only appears in `loadgen3`, while the others are shared across two or all three loadgens.

## Start the environment

1. Select the [button label="Terminal"](tab-1) tab:
```bash
./run
```
2. Make sure all three loadgens, the collector, and Refinery are running.

## Explore the data in Honeycomb

1. Select the [button label="Honeycomb"](tab-2) tab. Give the system about two minutes to begin emitting traces; the data should reflect the new traffic from `loadgen3`.
2. You may still be in Usage Mode from the end of Challenge 2. Switch back to Normal Mode (remove `/usage/` from the URL, or select **Return to Normal Mode**).
3. Set the time range to the last 10 minutes.
4. **WHERE** should still be `app.function exists`; **GROUP BY** should still be `app.function` from Challenge 2. If either got changed, set them back.
5. You should now observe four total functions:
   - Two functions shared by all loadgens (highest volume)
   - One function shared by two loadgens (mid volume)
   - One function only from `loadgen3` (lowest volume)
   
The presence of a third loadgen means overall traffic — and function frequency — is higher.

## Inspect sampling behavior in Usage Mode (by rule)

1. Change the visualization mode to Usage Mode — modify the URL by adding `/usage/` before `/result/` without losing your existing query.
2. Change the **WHERE** clause to `app.function exists`. Change **GROUP BY** to `meta.refinery.reason`. Add `AVG(Sample Rate)` to **VISUALIZE**.
3. Rerun this query. Keep the time range within the same 10-minute period. Select the area on the graph where you see activity, then select **Zoom in** (you're now viewing a custom absolute time range).
4. Interpret the results. This hits your sample goal! This approach works well because you're only using one field with moderate cardinality (`app.function`). If you added more fields to the `FieldList`, the number of unique keys (keyspace size) would increase, making it harder to hit the goal sample rate.

## Investigate the keyspace

1. Add `meta.refinery.reason = rules/trace/Sample the rest dynamically:emadynamic` to the **WHERE** clause. This is the rule we care about.
2. Change **GROUP BY** to `meta.refinery.sample_key`.
3. Add `SUM(Sample Rate)` to **VISUALIZE** — this shows what the count would be in normal mode, so you can compare adjusted count to unadjusted count in the same view.

A bit of quick math shows that the least busy service, `skin`, sent 6,784 events with an average sample rate of 4.5, which maps to about 30,000 spans. The busiest service, `drawer`, sent about 118,000 spans. Despite being nearly 4 times the number of events, the relative sample size was close.

High-volume keys were sampled more aggressively, meaning their sample rates were higher than the target while the lowest-volume key, which made up about one-third of the traffic, was sampled less frequently and received a lower-than-goal sample rate. This dynamic adjustment worked effectively because the overall cardinality of the keyspace was relatively low.

## Success criteria

- `refinery_configs/rules.yaml` keeps the `status_code >= 500` rule and replaces the deterministic catchall with an `EMADynamicSampler` (`GoalSampleRate: 10`, `FieldList: [app.function]`)
- `loadgen3` and `loadgen_configs/loadgen_config3.yaml` exist and are running alongside `loadgen1`/`loadgen2`
- Grouping by `app.function` in Honeycomb shows four distinct values
- In Usage Mode, grouping by `meta.refinery.reason` with `AVG(Sample Rate)` shows the dynamic rule landing close to its goal of 10
- In Usage Mode, grouping by `meta.refinery.sample_key` with `SUM(Sample Rate)` shows high-volume keys sampled more aggressively than the lowest-volume key
