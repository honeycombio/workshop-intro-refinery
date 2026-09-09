# Challenge 4: High Cardinality in Dynamic Samplers

This challenge builds directly on Challenge 3 — your `rules.yaml` should have the `EMADynamicSampler` with `GoalSampleRate: 10` and `FieldList: [app.function]`. You're about to deliberately break it, on purpose — watching a dynamic sampler fail is the fastest way to understand why field choice matters.

## Update the rules to introduce high cardinality

1. Select the [button label="Refinery Sample Application"](tab-0) tab.
2. Open `refinery_configs/rules.yaml`. Add one more field to the `FieldList`:
```yaml
                - app.user_id
```
3. This combines function and user ID values into each key. Because `app.user_id` is a randomly generated 24-character hex string, it introduces very high cardinality.

## Run the environment

1. Select the [button label="Terminal"](tab-1) tab:
```bash
./run
```

## Explore the data in Honeycomb in Usage Mode

1. Select the [button label="Honeycomb"](tab-2) tab. Give the system about two minutes to begin emitting traces.
2. Make sure you're in Usage Mode — modify the URL by adding `/usage/` before `/result/` without losing your existing query.
3. Set the time range to the last 10 minutes, then select the area on the graph where you see activity and select **Zoom in**.
4. Add `app.function exists` to the **WHERE** clause. Add `meta.refinery.reason` to **GROUP BY**.
5. Add `AVG(Sample Rate)` to **VISUALIZE**, alongside `COUNT`.
6. Increase the **LIMIT** to 1000.
7. You should observe: the "sample the rest dynamically" rule shows a much lower effective sample rate than the goal of 10.

## Investigate the keyspace

1. Add `meta.refinery.reason = rules/trace/Sample the rest dynamically:emadynamic` to the **WHERE** clause. Add `meta.refinery.sample_key` to **GROUP BY**, and remove `meta.refinery.reason` from **GROUP BY**.
2. You should observe:
   - A large number of distinct keys — this is due to the high cardinality.
   - Very few keys reach a sample rate near 10.
   - Many keys stuck at very low sample rates, close to 1.

## Visualize the keyspace size

1. Add `COUNT_DISTINCT(meta.refinery.sample_key)` to **VISUALIZE**, removing other values from **VISUALIZE**. Remove any values from **GROUP BY**. This shows you the size of the keyspace.
2. You should observe:
   - There are well over 1,000 unique keys.
   - Sample rates stay low because Refinery is constantly encountering new keys.
3. Keep in mind: heatmaps or counts grouped by sample key may appear unstable or incomplete, and despite some stable-looking data, you are not hitting your goal sample rate across the dataset.

## Why the sample rate fails

- The `EMADynamicSampler` uses an Exponential Moving Average to adjust sampling rates over time.
- The default maximum keyspace size is **500 keys**.
- If more than 500 unique keys are seen, new keys replace older ones, and evicted keys get a default `SampleRate` of 1 when seen again.

This constant turnover makes the effective sample rate spike and drop, especially for long-tail or rarely seen keys. Even though traffic continues to flow, Refinery can't maintain a representative sample across all keys.

## Recommendations: use FieldList wisely

- Be cautious about adding high-cardinality fields to `FieldList`.
- `app.user_id` is almost always unique — making it a poor fit for dynamic sampling.
- Try to limit the `FieldList` to fields with manageable cardinality (e.g., function names, region, service type).
- Try to capture the user intent so that different types of behaviors have representation.
- Use the queries above to measure sampler effectiveness and debug sampling behavior.

## Success criteria

- `refinery_configs/rules.yaml`'s `EMADynamicSampler` has `FieldList: [app.function, app.user_id]`
- In Usage Mode, the dynamic rule's `AVG(Sample Rate)` is well below the goal of 10
- Grouping by `meta.refinery.sample_key` shows a large number of distinct keys, most stuck near a sample rate of 1
- `COUNT_DISTINCT(meta.refinery.sample_key)` shows well over 500 unique keys
- You can explain why: the keyspace exceeds Refinery's default 500-key limit, so keys get evicted and reset before the sampler can settle on a stable rate for them
