# What is Dynamic Sampling? (Storyboard)

Source: "What is Dynamic Sampling_ Storyboard.pdf" — Section 1, Activity 3

## Panel 1
**Audio:** "What if the telemetry you care about isn't easy to describe with simple rules? What if it doesn't fall into large buckets I can define, like 'the trace has an error'? Maybe you're not just looking for errors or specific transaction types; you want to make sure that all parts of your application, even the quiet ones, are properly represented in your data."
**Visual:** Character thought bubble: "What if I want to proportionally represent different regions of my app?"

## Panel 2
**Audio:** "This is where dynamic sampling comes in! Honeycomb Refinery supports dynamic sampling strategies that adjust sample rates automatically, based on the observed frequency of specific field values in your traces. Let's say you want to dynamically sample based on HTTP status codes. We would want to sample 200s less frequently than 500s because they're a lot less common."
**Visual:** A Honeycomb events table showing `trace.trace_id` / `http.status_code` columns, with 200 values highlighted first, then a 500 value highlighted.

## Panel 3
**Audio:** "Honeycomb Refinery's dynamic samplers can assess the recent volume of traffic by creating keys based on the values of selected attribute fields, and then based on the volume of those keys, create a sample rate appropriate for the different traffic to meet an overall goal sample rate."
**Visual:** Two stacks: "Traces with 200 HTTP status code" (large stack, "keep less") vs. "Traces with 500 HTTP status code" (small stack, "keep more").

## Panel 4
**Audio:** "Here's an example of what two keys in a keyspace might look like if we have one trace with just 200 status codes, and another with a 500 status code in the trace."
**Visual:** "Trace A" (details) → "Key A: 200, Get"; "Trace B" (details) → "Key B: 200*500, Get".

## Panel 5
**Audio:** "Here's how that works under the hood. Refinery creates keys based on specific field values in each trace. These keys represent different segments of your traffic. Refinery keeps a running count of how often each key shows up, and uses that to calculate dynamic sample rates in real time. For high-frequency keys like status_code=200, the sample rate is set high—so only a small percentage is kept. For low-frequency keys like status_code=500, the sample rate is lower—so more of that rare data is retained."
**Visual:** "Key A: 200, Get" counter = 120 ("High frequency key, Most traces dropped") vs. "Key B: 200*500, Get" counter = 20 ("Low frequency key, Most traces retained").

## Panel 6
**Audio:** "And here's the best part: Refinery does this continuously. As your traffic changes, the sampling logic adapts too. So if a previously rare scenario becomes more common, or new patterns emerge, the sample rates rebalance automatically, without any manual rule-writing. This makes it possible to preserve coverage across your entire application, even when traffic patterns shift, without having to create complex custom rules for every possible case."
**Visual:** Key A counter now 330 (labels swap to "Low frequency key, Most traces retained"), Key B counter now 500 (swaps to "High frequency key, Most traces dropped") — demonstrating the dynamic rebalancing.

## Panel 7 (Recap)
**Audio:** "With dynamic sampling, Honeycomb Refinery lets you keep the rare, meaningful traces and scale down the noise automatically. To recap: Dynamic sampling adjusts sample rates in real time based on how frequently specific field values appear in your trace data. Common values (like HTTP 200s) are sampled less, while rare or meaningful ones (like 500s) are sampled more—automatically and continuously. This approach preserves meaningful coverage across your app without manual rules, helping you keep rare signals and reduce noise at scale."
**Visual:** Standard recap slide.
