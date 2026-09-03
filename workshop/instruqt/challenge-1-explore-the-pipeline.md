# Challenge 1: Explore Your Refinery Pipeline

Your sandbox is already running a full telemetry pipeline: two load generators (`loadgen1`, `loadgen2`) sending synthetic traffic through an OpenTelemetry Collector, into Honeycomb Refinery, and on to Honeycomb. You won't build any of this yourself — your job in this challenge is to understand what's already running before you start tuning it.

1. Select the [button label="Terminal"](tab-1) tab.
2. Check what's running:
```bash
docker compose ps
```
You should see four services up: `loadgen1`, `loadgen2`, `otel-collector`, and `refinery`.

## How the pieces fit together

- **loadgen** generates repeatable synthetic trace traffic — the same shape of traffic every time you run it, which makes it easy to compare unsampled vs. sampled data.
- **otel-collector** receives that traffic over OTLP and enriches it, splitting the generated URL into `app.function` and `app.endpoint` fields to make later querying and rule-writing easier.
- **Refinery** sits between the collector and Honeycomb. It caches each trace's spans, waits until the trace looks complete, then applies your sampling rules to decide what gets kept.
- Only what Refinery keeps is forwarded on to Honeycomb.

## Read the pipeline configuration

1. Select the [button label="Refinery Sample Application"](tab-0) tab. This is the sample app's code — you'll come back here to make changes in later challenges.
2. Open `docker-compose.yaml`. Find the `refinery` service block. Notice it mounts two files as volumes — a `config.yaml` and a `rules.yaml`.
3. Open `refinery_configs/rules.yaml`. Notice the `__default__` sampler is a `DeterministicSampler` with a `SampleRate`. Whatever number you see here is roughly the fraction of traces Refinery is currently keeping — a `SampleRate` of 10 means it keeps about 1 in every 10 traces.
4. Open `collector_configs/otelcol-config.yaml`. Find the `exporters` section — notice traffic is sent to `refinery`, not directly to Honeycomb.

## Confirm data is flowing in Honeycomb

1. Select the [button label="Honeycomb"](tab-2) tab.
2. Set the time range to the last 10 minutes.
3. Run a query:
   - **WHERE:** `app.function exists`
   - **GROUP BY:** `app.function`, `app.endpoint`
4. You should see three distinct `app.function` values, and many more `app.endpoint` values.
5. Now remove `app.endpoint` from **GROUP BY** and rerun — this simplified view is what you'll come back to in later challenges to see how your rule changes affect the data.

> [!IMPORTANT]
> If you don't see any data, go back to the [button label="Terminal"](tab-1) tab and run `docker compose logs otel-collector` and `docker compose logs refinery` to check for errors before re-checking Honeycomb.

## Success criteria

- All four services (`loadgen1`, `loadgen2`, `otel-collector`, `refinery`) show as running via `docker compose ps`
- You can state what sampler and `SampleRate` are currently configured in `refinery_configs/rules.yaml`
- A Honeycomb query grouped by `app.function` shows three distinct values with live data in the last 10 minutes
