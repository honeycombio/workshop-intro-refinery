# Effective Sampling with Honeycomb Refinery — Sample Application

***This is a demo app, don't run it in production!***

This is the sample application for Honeycomb's **Effective Sampling with Honeycomb Refinery** workshop. It generates synthetic trace data through a small pipeline — two load generators, an OpenTelemetry Collector, and Honeycomb Refinery — so workshop participants can see the effects of sampling on their data in Honeycomb.

## Introduction

Hello! Welcome to the **Effective Sampling with Honeycomb Refinery** workshop's sample application.

1. This repository provides a sample pipeline of applications, an OpenTelemetry Collector, and an instance of Refinery used for sampling.
2. The pipeline here is already fully wired together on the `main` branch — Refinery is running from the start, with a sensible default sampling rule already in place. Workshop participants spend their time tuning Refinery's sampling rules, not building the pipeline itself.

## Running the application

Once you run the application, you can send traces to Honeycomb.

### Local development setup

You also have the option to run this application locally.

First, clone this repository.

```bash
git clone https://github.com/honeycombio/workshop-intro-refinery.git
```

Install Docker: <https://docs.docker.com/get-docker/>

Update the `.env` file with your Honeycomb API key:

```bash
HONEYCOMB_API_KEY="your-api-key"
```

If you don't have an API key handy, here is the [documentation](https://docs.honeycomb.io/get-started/configure/environments/manage-api-keys/#create-api-key).

### Run the app

`./run`

(This will run `docker compose` in daemon mode.)

After making changes to a service, you can tell it to rebuild just that one:

`./run [ otel-collector | loadgen[123] | refinery ]`

Note: this workshop assumes you're using Linux or macOS. If you're using Windows, use WSL.

### Stop the app

`./stop`
