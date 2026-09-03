# Lab Guides — Section 3: Tuning Your Refinery Rules

Source: "Scripts - Section 3.pdf"

## Activity 1 — Using Rules to Change How Data Is Sampled (Hands-on)

**Repository:** Academy-Intro-Refinery, branch `section3_activity1` (end state after s2a5 → s3a1)

**Learning Objectives:**
- Update the Refinery rules to make sure Refinery keeps traffic of a certain value.
- See how traffic changes through normal and Usage Mode queries in Honeycomb.

### Script

**Update the `rules.yaml` File** — add a rule for `>=500` status codes plus a catchall with sample rate 10:
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
        - Name: Sample the rest
          SampleRate: 10
```
The catchall rule has no conditions, so all non-matching traces land there and Refinery keeps 10% of them.

**Run the Environment:**
```bash
./run
```

**Create a Calculated Field to Flag 500 Errors** — in the Query Builder GROUP BY, define a calculated field named `if 500 true`:
```
IF(
  EXISTS(
    $http.response.status_code
  ),
  GTE(
    $http.response.status_code,
    500
  )
)
```
Returns `true` for traces where `http.response.status_code >= 500`.

**Visualize the Differences:**
- Time range last 10 minutes. VISUALIZE: `COUNT`, `HEATMAP(duration_ms)`. WHERE: `app.function exists`. GROUP BY: the calculated field above.
- Expect a clear, consistent appearance of 500s (unlike prior deterministic sampling where some were missing/skewed) and a smoother, better-populated duration heatmap for 500 errors.

**Compare With Deterministic Sampling:** previously (deterministic sampler across all traffic) some 500s were missed and duration heatmaps were incomplete/distorted for low-volume-but-important traces. Now, every 500-status trace is preserved and visualizations are more accurate.

**Use Refinery Meta Fields to Inspect Rule Behavior:**
1. Zoom into the same 10-minute period (custom absolute time range).
2. GROUP BY `meta.refinery.reason` (remove `HEATMAP(duration_ms)`) — shows which rule each trace matched (e.g. `Keep all when http.response.status_code >= 500` vs. `Sample the rest`).
3. Add `http.response.status_code` to GROUP BY to confirm 500s landed in the correct rule and observe the traffic split between intentional and catch-all rules.

**Anomaly Detection Based on Rule Matches:** hover/select a point on the error-trace line, choose "Detect Anomalies." Use `app.` in the search to focus on specific attributes; hover bar charts to see what could be triggering errors — helps identify which rules capture which anomalies and whether specific error types/latencies link to certain rules.

**Explore Sample Rate Behavior in Usage Mode:**
1. Switch to Usage Mode (add `/usage/` before `/result/` in the URL, without losing the query).
2. GROUP BY `app.function`. VISUALIZE: add `AVG(Sample Rate)`. Zoom to the same 10-minute period (absolute time).
3. Observations: `clock` and `bulb` functions had no errors, so their sample rate is flat at 10. `drawer` had errors during a couple of segments and sent SampleRate-1 data mixed with SampleRate-10 non-error data, blending its average sample rate to ~9.4 over the period (visibly dropping during the error spike).
4. Presence of 500s on a given function lowers that function's apparent average sample rate.

**Shut Down the Environment:**
```bash
./stop
```

**Recap:** Used Refinery rules to intentionally preserve important traffic; created rules for status-code 500s; verified behavior using meta fields; explored effects in heatmaps and Usage Mode; used BubbleUp to dig deeper.

---

## Activity 2 — Adding a Dynamic Sampler (Hands-on)

**Repository:** Academy-Intro-Refinery, branch `section3_activity2` (end state after s3a1 → s3a2)

**Learning Objectives:**
- Understand how to configure the `EMADynamicSampler`.
- Explain how the keyspace is generated in the `EMADynamicSampler` and show the keys in Honeycomb queries.

**Note:** assumes completion of Activity 1.

### Script

**Update the Rules to Use the `EMADynamicSampler`** — replace the deterministic catch-all rule with a dynamic rule; keep the status-500 rule as-is:
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
Note: the `FieldList` defines the keyspace — here, `app.function` generates unique sampling keys; key volume determines each key's sample rate, aiming for the overall target average.

**Add the Third Load Generator** — duplicate a `loadgen` service block in `docker-compose.yaml`:
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

**Create the Third LoadGen Configuration:**
```bash
touch loadgen_configs/loadgen_config3.yaml
```
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
  # simulate URLs for 10 services, each with 10 endpoints on the root span
  0.url.full: /u4,5
  # generate status codes where 5% are 400s and .1% are 500s on the root span
  0.http.response.status_code: /st5,0.1
  # generate a hex ID as a user_id on the root span
  0.app.user_id: /sx24
```
This adds a new `app.function` value once processed by the collector — increasing cardinality; one function appears only in `loadgen3`, others are shared across two or all three loadgens.

**Start the Environment:**
```bash
./run
```

**Explore the Data in Honeycomb:** after ~2 minutes, group by `app.function` — should see four total functions: two shared by all loadgens (highest volume), one shared by two loadgens (mid volume), one only from `loadgen3` (lowest volume). Overall traffic/function frequency is higher with the third loadgen present.

**Inspect Sampling Behavior in Usage Mode (By Rule):**
1. Switch to Usage Mode. WHERE `app.function exists`. GROUP BY `meta.refinery.reason`. VISUALIZE: add `AVG(Sample Rate)`. Zoom to the active time window.
2. Interpretation: this hits the sample goal — works well because only one moderate-cardinality field (`app.function`) is used. Adding more fields would increase keyspace size and make the goal harder to hit.

**Investigate the Keyspace:**
1. WHERE: add `meta.refinery.reason = rules/trace/Sample the rest dynamically:emadynamic`. GROUP BY: `meta.refinery.sample_key`. VISUALIZE: add `SUM(Sample Rate)` (to compare adjusted vs. unadjusted count in one view).
2. Example finding: least-busy key ("skin") sent 6,784 events at avg sample rate 4.5 (~30,000 spans); busiest key ("drawer") sent ~118,000 spans — despite ~4x the event count, relative sample size was close. High-volume keys got sample rates above the goal; the lowest-volume key (~1/3 of traffic) got a lower-than-goal sample rate. This worked because overall keyspace cardinality stayed relatively low.

**Shut Down the Environment:**
```bash
./stop
```

**Recap:** Configured `EMADynamicSampler` to adjust sampling behavior by traffic volume using one moderate-cardinality field (`app.function`); added a third loadgen and observed volume impact on sample rates; validated effectiveness via Usage Mode (`meta.refinery.reason`, `meta.refinery.sample_key`) — high-volume keys got higher rates, low-volume keys got lower rates, maintaining the overall goal. A single moderately-cardinal FieldList let the sampler manage the keyspace efficiently.

---

## Activity 3 — How High Cardinality Affects Dynamic Samplers (Hands-on)

**Repository:** Academy-Intro-Refinery, branch `section3_activity3` (end state after s3a2 → s3a3)

**Learning Objectives:**
- Recognize that high-cardinality fields in the `FieldList` eliminate the effective sample rate of the `EMADynamicSampler`.
- Use Usage Mode queries in Honeycomb to assess the effective sample rate of data sent through an `EMADynamicSampler`.

**Note:** assumes completion of Activity 2.

### Script

**Update the Rules to Introduce High Cardinality** — add to the `FieldList`:
```yaml
                - app.user_id
```
This combines function + user ID values into keys. `app.user_id` is a randomly generated 24-character hex string — very high cardinality.

**Run the Environment:**
```bash
./run
```

**Explore the Data in Honeycomb in Usage Mode:**
1. After ~2 minutes, Usage Mode. Time range last 10 minutes (zoom to activity). WHERE `app.function exists`. GROUP BY `meta.refinery.reason`. VISUALIZE: add `AVG(Sample Rate)`. Limit 1000.
2. Observation: the "sample the rest dynamically" rule shows a **much lower effective sample rate** than the goal of 10.

**Investigate the Keyspace:**
1. WHERE: add `meta.refinery.reason = rules/trace/Sample the rest dynamically:emadynamic`. GROUP BY: `meta.refinery.sample_key` (remove reason from GROUP BY).
2. Observations: a large number of distinct keys (due to high cardinality); very few keys reach a sample rate near 10; many keys stuck near 1.

**Visualize the Keyspace Size:**
1. VISUALIZE: `COUNT_DISTINCT(meta.refinery.sample_key)` only (remove other Visualize/Group By values) — shows keyspace size.
2. Observation: well over 1,000 unique keys; sample rates stay low because Refinery keeps encountering new keys. Heatmaps/counts by sample key may look unstable/incomplete; the dataset overall is **not hitting the goal sample rate**.

**Why the Sample Rate Fails:**
- `EMADynamicSampler` uses an Exponential Moving Average to adjust rates over time.
- Default max keyspace size is **500 keys**.
- Beyond 500 unique keys, new keys replace older ones; evicted keys get a default `SampleRate` of 1 when seen again.
- This constant turnover makes the effective sample rate spike/drop, especially for long-tail/rare keys — Refinery can't maintain a representative sample across all keys even though traffic keeps flowing.

**Recommendations: Use FieldList Wisely**
1. Be cautious adding high-cardinality fields to `FieldList`.
2. `app.user_id` is almost always unique — a poor fit for dynamic sampling.
3. Limit `FieldList` to fields with manageable cardinality (function names, region, service type).
4. Try to capture user intent so different behavior types have representation.
5. Use the queries above to measure sampler effectiveness and debug sampling behavior.

**Shut Down the Environment:**
```bash
./stop
```

**Recap:** Observed how adding a high-cardinality field (`app.user_id`) to the dynamic sampler's `FieldList` overwhelms the sampler's keyspace — resulting in unstable sample rates, lower-than-expected data volume, and inconsistent coverage.

---

## Activity 4 — Sensible Default Refinery Rules (Video)

**Format:** Video

**Learning Objectives:**
- Learners will be reminded of the importance of rule ordering.
- Learners will have some recommendations for starting rules for sampling their application data.

**Deck:** "Activity 4: Sensible Defaults" — full deck transcript in `activity4-sensible-defaults.md`.

Slide 1 narration: "Now that you've got a solid handle on how Refinery is configured—and how rules shape your sampling—we'll walk through a few example rules you can use as a starting point. These will help you build a ruleset that aligns with your business goals."

Slide 2 narration ("Rules outlined"): "These rules aim to preserve the high-value traces—like those with errors or 500 status codes—where high cardinality matters. At the same time, we drop low-value traffic, like health checks from load balancers, and apply sensible sampling to everything else."

(Individual rule narrations match `activity4-sensible-defaults.md` exactly.)

Closing narration: "Together, these rules give you a strong foundation for crafting a sampling strategy that aligns with your environment and business needs."
