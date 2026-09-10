# Challenge 2: Using Rules to Change How Data Is Sampled

Right now Refinery treats every trace the same way: a flat `DeterministicSampler` keeping 1 in 10, no matter what's in the trace. That means a real error could just as easily get dropped as a routine successful request. In this challenge, you'll write a rule that guarantees errors are never lost, while everything else keeps sampling at the same rate as before.

## Update your rules.yaml

1. Select the [button label="Refinery Sample Application"](tab-0) tab.
2. Open `refinery_configs/rules.yaml` and replace its contents with:
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
3. Rules are checked top to bottom, and the first match wins. This adds a targeted rule *ahead of* the catchall: any trace whose root span has `http.response.status_code >= 500` is now kept 100% of the time (`SampleRate: 1`). Everything else still falls through to the second rule and gets sampled at the same 1-in-10 rate as before.

> [!IMPORTANT]
> YAML is picky about indentation. `Conditions` and its `-` list items need to line up exactly as shown, or Refinery will fail to load the file. If something doesn't work after this step, double-check your indentation first.

## Restart the pipeline

1. Select the [button label="Terminal"](tab-1) tab.
2. Restart so Refinery picks up the new rules:
```bash
./run
```

## Create a calculated field to flag 500 errors

1. Select the [button label="Honeycomb"](tab-2) tab.
2. In the Query Builder's **GROUP BY**, define a new calculated field named `if 500 true`:
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
This returns `true` for any trace where `http.response.status_code >= 500`.

## Visualize the difference

1. Set the time range to the last 10 minutes.
2. **VISUALIZE:** `COUNT`, `HEATMAP(duration_ms)`
3. **WHERE:** `app.function exists`
4. **GROUP BY:** remove `app.function` (left over from Challenge 1) and replace it with the `if 500 true` calculated field you just created
5. Compare this to what you saw in Challenge 1 with the flat deterministic sampler: 500-status traces should now show up consistently every time they occur, and the duration heatmap for them should look smoother and more complete, not randomly missing data the way a flat 1-in-10 sample could.

## Confirm which rule matched, using Refinery's meta fields

1. Zoom into the same active window: select the area on the graph where you see traffic, then select **Zoom in** (you're now viewing a fixed absolute time range instead of a rolling "last 10 minutes").
2. Remove the `if 500 true` calculated field from **GROUP BY** and replace it with `meta.refinery.reason`. Remove `HEATMAP(duration_ms)` from **VISUALIZE** (keep `COUNT`).
3. You should see two values, one for each rule (`Keep all when http.response.status_code >= 500` and `Sample the rest`).
4. Add `http.response.status_code` to **GROUP BY** as well, to confirm the 500-status traces are landing under the "Keep all…" rule specifically, not the catchall.

## BubbleUp outliers based on rule matches

1. Hover over and select a point on the line representing traces with errors, then select **BubbleUp Outliers**.
2. Use `app.` in the search box to focus on specific attributes.
3. Hover over the bar charts to see what could be triggering errors.

This helps identify which rules are capturing outliers, and whether specific error types or latencies are linked to certain rules.

## Check the sample rate in Usage Mode

1. Go back to the **Overview** tab.
2. Switch to Usage Mode: add `/usage/` before `/result/` in the URL without losing your query.
3. Remove `meta.refinery.reason` and `http.response.status_code` from **GROUP BY** (left over from the last step) and replace them with `app.function`.
4. Add `AVG(Sample Rate)` to **VISUALIZE**.
5. Keep the time range within the same 10-minute period. Select the area on the graph where you see activity, then select **Zoom in**.
6. You should see each function's average sample rate in the visualization. `clock` and `bulb` had no errors, so their sample rate is flat at 10. `drawer` had errors during a couple of segments and sent `SampleRate: 1` data alongside the `SampleRate: 10` non-error data. This blends the sample rate for that function to about 9.4 over the period, and you can see it visibly drop during the error spike. Note how the presence of 500s affects downstream fields this way: since all the 500s are on the `drawer` function, that function is the one that appears to have a lower average sample rate.

## Success criteria

- `refinery_configs/rules.yaml` has two rules: a `SampleRate: 1` rule scoped to `http.response.status_code >= 500`, followed by a catchall at `SampleRate: 10`
- A Honeycomb query grouped by `meta.refinery.reason` shows both rule names, with 500-status traces landing under the "Keep all…" rule
- In Usage Mode, `drawer` shows a blended `AVG(Sample Rate)` around 9.4, while `clock` and `bulb` stay flat at 10
