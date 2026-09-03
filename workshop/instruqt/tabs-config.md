# Instruqt Sandbox: Host & Tab Configuration

Reference for the track's sandbox setup — VM host, tabs, and the startup script. Set up via Instruqt's UI (Track → Sandbox); the YAML block below is the equivalent config if the CLI/`track.yml` route is used instead.

## Host

- **Name:** `refinery-workshop-app`
- **Type:** Virtual machine
- **Image:** custom — `effective-sampling-refinery-image` (built per Part 1 of the host-image guide)
- **Machine type:** Large

## Tabs

| Tab | Title | Type | Config |
|---|---|---|---|
| 0 | Refinery Sample Application | Code Editor | host: `refinery-workshop-app`, path: `/root/workshop-intro-refinery` |
| 1 | Terminal | Terminal | host: `refinery-workshop-app` |
| 2 | Honeycomb | Website | `url: https://ui.honeycomb.io`, `new_window: true` (required — Honeycomb sends `X-Frame-Options: DENY`, confirmed via `curl -sI https://ui.honeycomb.io/` on 2026-09-03) |

**Note:** verify the Honeycomb URL region before finalizing — this project has EU-instance awareness from an earlier commit (see git history), so double-check whether learners should be pointed at `ui.honeycomb.io` (US) or `ui.eu1.honeycomb.io` (EU).

No "Service/web application" tab is used — the sample app has no browser-facing UI or port to point one at.

Equivalent YAML (per [Instruqt's challenge-tabs config](https://docs.instruqt.com/tracks/challenges/challenge-tabs)):
```yaml
tabs:
  - title: Refinery Sample Application
    type: code
    hostname: refinery-workshop-app
    path: /root/workshop-intro-refinery
  - title: Terminal
    type: terminal
    hostname: refinery-workshop-app
  - title: Honeycomb
    type: website
    url: https://ui.honeycomb.io
    new_window: true
```

## Setup script

`setup-refinery-workshop-app` (this directory) — follows Instruqt's `setup-<hostname>` naming convention for track lifecycle scripts. Starts the docker-compose stack and waits for `otel-collector` and `refinery` specifically (not the loadgens, which are expected to exit on their own after their traffic burst).

**Unverified:** the `docker compose ps | grep` readiness check should work broadly across Compose v2 versions, but hasn't been tested against whatever Compose version ships with the `instruqt/docker-28-3` base image. Worth a quick manual test in the image-build terminal before relying on it at learner-launch time.
