# Challenge 1: Explore Your Refinery Pipeline

Your sandbox is already running a full telemetry pipeline: two load generators (`loadgen1`, `loadgen2`) sending synthetic traffic through an OpenTelemetry Collector, into Honeycomb Refinery, and on to Honeycomb. You won't build any of this yourself — your job in this challenge is to understand what's already running before you start tuning it. One thing you *do* need to do first: the pipeline ships with a placeholder Honeycomb API key, so it can't actually send data to your Honeycomb environment until you add your own.

1. Select the [button label="Terminal"](tab-1) tab.
2. Check what's running:
```bash
docker compose ps -a
```
You should see all four services listed: `loadgen1`, `loadgen2`, `otel-collector`, and `refinery`. `otel-collector` and `refinery` should show `Up` — they run continuously. `loadgen1` and `loadgen2` generate a fixed ~2-minute burst of traffic and then exit on their own, so depending on timing you might see them as `Up` or as `Exited (0)` — both are expected. If you want a fresh burst of traffic running right now, use `./run` to restart just the load generators.

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

## Create your own Honeycomb environment

Use your own personal Honeycomb account for this workshop, not a shared team account. That way there's no mix-up over who owns what data or environment.

1. Select the [button label="Honeycomb"](tab-2) tab.
2. If you don't already have a personal Honeycomb account, sign up for free at [ui.honeycomb.io/signup](https://ui.honeycomb.io/signup) (US) or [ui.eu1.honeycomb.io/signup](https://ui.eu1.honeycomb.io/signup) (EU) — signup automatically creates your own team with you as its Team Owner. Already have a personal account? Just log in instead.
3. Create a dedicated environment for this workshop: select **Environments** (top-left) → **Manage Environments** → **Create Environment**. Name it `refinery-workshop`, then select **Create Environment**.
4. Honeycomb generates an API key for the new environment automatically. Select **View API Keys** on the environment to find it. Copy it now; you'll need it in the next step.

## Add your Honeycomb API key

The pipeline can't reach your new environment yet; it's using a placeholder key.

1. Select the [button label="Refinery Sample Application"](tab-0) tab.
2. Open `.env` in the repo root.
3. Replace `<YOUR_HONEYCOMB_API_KEY>` with the API key you just copied.
4. Select the [button label="Terminal"](tab-1) tab and restart the pipeline so it picks up the new key — editing `.env` alone doesn't do anything until the containers restart:
```bash
./stop && ./run
```
5. Give it about 30 seconds to come back up and start sending fresh traffic.

## Confirm data is flowing in Honeycomb

1. Select the [button label="Honeycomb"](tab-2) tab.
2. Set the time range to the last 10 minutes.
3. Run a query:
   - **WHERE:** `app.function exists`
   - **GROUP BY:** `app.function`, `app.endpoint`
4. You should see three distinct `app.function` values, and many more `app.endpoint` values.
5. Now remove `app.endpoint` from **GROUP BY** and rerun — this simplified view is what you'll come back to in later challenges to see how your rule changes affect the data.

> [!IMPORTANT]
> If you don't see any data, go back to the [button label="Terminal"](tab-1) tab and run `docker compose logs otel-collector` and `docker compose logs refinery` to check for errors before re-checking Honeycomb. A `401 response for AuthInfo request` or `check your API key` error almost always means `.env` still has the placeholder key, or the pipeline wasn't restarted after you edited it — double-check `.env`, then run `./stop && ./run` again.

## Success criteria

- `.env` contains your own Honeycomb API key, not the placeholder
- All four services appear via `docker compose ps -a`, with `otel-collector` and `refinery` showing `Up` (`loadgen1`/`loadgen2` may show `Up` or `Exited (0)` depending on timing — both are fine)
- You can state what sampler and `SampleRate` are currently configured in `refinery_configs/rules.yaml`
- A Honeycomb query grouped by `app.function` shows three distinct values with live data in the last 10 minutes
