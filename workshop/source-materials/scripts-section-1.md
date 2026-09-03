# Video Scripts — Section 1: Understanding Sampling

Source: "Scripts - Section 1.pdf"

## Activity 1 — What Is Sampling? (Video)

**Learning Objectives:**
1. Learners will be able to define sampling.
2. Learners will be able to articulate the benefits and risks of cost vs. sampling.
3. Learners will be able to explain the effects of sampling on cardinality.

**Script:**

[Audio] In 1908, a statistician at a brewery figured out something remarkable: you don't need to test every pint to know if the beer's good. Using a small sample, he could predict with confidence how the entire batch would turn out. That method—what we now call the T-Test—proved that a well-chosen slice of data can tell the whole story. In this video, we'll bring that same principle to your telemetry data. You'll learn what sampling is, why it's reliable, and how it helps you balance visibility with cost and performance in tools like Honeycomb.
[Visual] Historical sketch of a brewing scene fading into a stylized chart. Overlay text: "Small sample, big confidence." Transition to title card: "What Is Sampling?"

[Audio] You might've heard that teams—maybe even your own—are "doing sampling" with their telemetry data. But what does that actually mean?
[Visual] Title card "What Is Sampling?" with subtle animation of data flowing into a funnel.

[Audio] Think of it like lossy compression for images or audio. You don't have every pixel or sound wave, but you still see the picture, hear the music, and get the message.
[Visual] Side-by-side of original vs. compressed image. Audio wave with subtle drop in detail. "Still useful!" stamp.

[Audio] In telemetry, sampling means sending just a subset of your data instead of everything. You're still aiming to understand trends and patterns—just with less volume. Done right, sampling gives you a statistically useful view of your system's behavior, without drowning in data.
[Visual] Overlaid telemetry events, with a few "selected" by a highlight or filter animation.

[Audio] Now, you might wonder: doesn't sampling make my data less reliable? Won't sampling affect my ability to accurately query my data in Honeycomb? Honeycomb's Refinery tool solves for that. Refinery supports multiple sampling types, chosen based on the situation, context, and desired outcomes. When your telemetry is sampled, metadata is added to indicate how much data was sampled, and Honeycomb can use that to scale your results. That means your queries still reflect reality, just more efficiently. Refinery does this by applying sample rates to your data. The sample rate is a measure of how much your telemetry data has been reduced before being sent to Honeycomb. It represents the ratio between the total number of events generated and the number of events actually sent to Honeycomb for storage and analysis, expressed in a 1/N format. For example, if your sample rate is 1000, it means that: each event sent to Honeycomb represents 1,000 actual events that occurred in your system (that will be reflected in your Honeycomb queries where COUNT, SUM, etc. results will be multiplied by 1,000); in other words, you're only sending 1 out of every 1,000 events, and the other 999 are discarded.
[Visual] Graph showing extrapolated trend from sampled data. "Multiplier" icons to show adjusted values.

[Audio] The biggest reason for sampling is cost. Ingesting and storing high-volume telemetry, especially with high cardinality, can get expensive fast, both economically and computationally. With smart sampling, you reduce both storage and compute costs. And with the right setup, you still achieve excellent observability, even at lower data volumes.
[Visual] Cost vs. performance scale. Icons for compute, storage, and observability.

[Audio] Let's recap: Sampling means sending a smaller, representative slice of your telemetry. It reduces cost while keeping visibility. Tools like Honeycomb's Refinery let you adjust for sampling, so your insights stay accurate.
[Visual] Three-point checklist with icons.

*(See `workshop/assets/what-is-sampling-frames/` for stills extracted from the final rendered video, since no storyboard exists for this activity — the final video adds a "Learning Objectives" opening slide, a "Key Takeaways" closing slide, and a bee/garden metaphor sequence not described in this written script.)*

---

## Activity 2 — Head Sampling v. Tail Sampling: When to Sample (Video)

**Learning Objectives:**
1. Learners will be able to delineate the differences between head sampling and tail sampling.

**Script:**

[Audio] When you're sampling telemetry, what if you really need to make sure you capture certain data, like errors or payment transactions? That's where different types of sampling strategies come into play. There are two main kinds: head sampling and tail sampling.
[Visual] Character at their desk, puzzled. "What if I need to keep error or payment data?" in a thought bubble. Title card: "Two Types of Sampling: Head vs. Tail"

[Audio] Let's start with head sampling. This approach makes sampling decisions early either before the telemetry is generated, or by quickly comparing something like a trace ID–which ensures all spans in a trace stay together–against a hash to determine if all spans in that trace should be sampled or dropped. It's fast and computationally cheap. That's because the decision logic is simple and happens immediately—no need to store or process any trace data first.
[Visual] Diagram of head sampling: incoming requests, quick decision path with a hash, "Keep"/"Drop" arrows, labels "Fast," "Low compute cost."

[Audio] On the other end, we have tail sampling. This method makes sampling decisions after the telemetry has been received, once the entire trace is available. By inspecting the data first, you can ensure that you keep the most critical data, but drop a larger portion of less critical data. This, of course, is much more computationally intensive, because you need to keep the telemetry until a full trace has arrived, and then you need to parse all the attributes in the trace to know how to apply your sampling decisions. With really high volumes of data, tail sampling can be discouragingly expensive on its own.
[Visual] Diagram of tail sampling: all traces stored first, full trace visual shown, attributes scanned (e.g. "error=true," "payment=true"), keep/drop decision applied after analysis, labels "Attribute-aware," "Delayed decision," "Higher cost."

[Audio] So how do teams balance the two? Head sampling is fast but uninformed; it doesn't look at the trace content. Tail sampling is expensive but smart; it uses the actual data to guide decisions. Because of that tradeoff, many companies combine both. They'll use head sampling to reduce overall volume, and tail sampling on top of that to retain the most important traces.
[Visual] Side-by-side diagram: head sampling labeled "Fast," tail sampling labeled "Detailed, costly," flowchart of combined strategy, text "Balanced sampling strategy = Efficiency + Fidelity."

[Audio] Here's the key idea: when you have high trace volume, individual traces become less critical. That's because any important pattern is likely to repeat. So by head sampling just 10 to 50 percent of your traffic, you reduce what gets passed to tail sampling, making it feasible to analyze and selectively keep the most valuable traces.
[Visual] Animated funnel: large volume of traces → head sampling reduces volume → tail sampling filters for "important" traces → smaller set of meaningful traces labeled "error," "slow query," "payment."

[Audio] That's the tradeoff. Head sampling gives you speed and scale. Tail sampling gives you control and precision. Together, they help ensure you get the data you care about, without blowing up cost or infrastructure. To recap: Head sampling is fast and cost-effective, making quick decisions before or immediately after telemetry is generated. Tail sampling inspects full trace data after it's collected, allowing precise retention of important traces—but it's significantly more resource-intensive. Combining both allows teams to balance scale and fidelity: head sampling limits volume, while tail sampling preserves critical data like errors or payment traces.
[Visual] Wrap-up slide: "Head sampling: Fast, probabilistic" / "Tail sampling: Precise, compute-intensive" / "Combined: Scalable, insightful observability"

---

## Activity 3 — What is Dynamic Sampling? (Video)

**Learning Objectives:**
1. Learners will understand the basics of how dynamic sampling works.

**Script:**

[Audio] What if the telemetry you care about isn't easy to describe with simple rules? What if it doesn't fall into large buckets I can define, like "the trace has an error"? Maybe you're not just looking for errors or specific transaction types; you want to make sure that all parts of your application, even the quiet ones, are properly represented in your data.
[Visual] Character curious, thinking: "What if I want to proportionately represent different regions of my app?" Overlay: "When simple rules aren't enough…"

[Audio] This is where dynamic sampling comes in! Honeycomb Refinery supports dynamic sampling strategies that adjust sample rates automatically, based on the observed frequency of specific field values in your traces.
[Visual] Chart labeled "Dynamic Sampling," zoom on a dataset where certain field values are more common than others.

[Audio] Let's say you want to dynamically sample based on HTTP status codes. We would want to sample 200s less frequently than 500s because they're a lot less common. Honeycomb Refinery's dynamic samplers can assess the recent volume of traffic by creating keys based on the values of selected attribute fields, and then based on the volume of those keys, adjust a sample rate appropriate for the different traffic to meet an overall goal sample rate. The goal sample rate is the average frequency at which Refinery aims to send telemetry data to Honeycomb. For example, a goal sample rate of 1000 means Refinery tries to send about 1 out of every 1,000 events. Dynamic sampling adjusts the sampling rate for different types of events to collectively meet this target—it doesn't mean every event is sampled exactly 1-in-1000. High-frequency, less critical events–like 200s–might be sampled less frequently, while rare or significant events–like 500s–might be sampled more frequently or even sent in full.
[Visual] High-frequency 200s with lower sampling vs. rare 500s with higher sampling; arrows "keep less" vs "keep more." Example of two keys in a keyspace for a trace with 200 status codes vs. one with a 500.

[Audio] Here's how that works under the hood. Refinery creates keys based on specific field values in each trace. These keys represent different segments of your traffic. Refinery keeps a running count of how often each key shows up, and uses that to calculate dynamic sample rates in real time. For high-frequency keys like status_code=200, the sample rate is set high—so only a small percentage is kept. For low-frequency keys like status_code=500, the sample rate is lower—so more of that rare data is retained.
[Visual] Keyspace illustration: Key A: status_code=200, Key B: status_code=500, arrows showing frequency counters increasing over time. Animated funnel per key: wide funnel (200s, most dropped), narrow funnel (500s, most retained). Text: "Sample rate adapts by traffic frequency."

[Audio] And here's the best part: Refinery does this continuously. As your traffic changes, the sampling logic adapts too. So if a previously rare scenario becomes more common, or new patterns emerge, the sample rates rebalance automatically, without any manual rule-writing. This makes it possible to preserve coverage across your entire application, even when traffic patterns shift, without having to create complex custom rules for every possible case.
[Visual] Dashboard showing shifting traffic over time, sample rates adjusting in response. Callout: "Dynamic sampling reacts to change." Character amazed/relieved: "No need to hand-code tail sampling rules."

[Audio] With dynamic sampling, Honeycomb Refinery lets you keep the rare, meaningful traces and scale down the noise automatically. To recap: Dynamic sampling adjusts sample rates in real time based on how frequently specific field values appear in your trace data. Common values (like HTTP 200s) are sampled less, while rare or meaningful ones (like 500s) are sampled more—automatically and continuously. This approach preserves meaningful coverage across your app without manual rules, helping you keep rare signals and reduce noise at scale.

---

## Activity 4 — Sampling and Fidelity Tradeoffs (Video)

**Learning Objectives:**
1. Reiterate to learners that sampling comes at the cost of high fidelity.

**Script:**

[Audio] Dynamic sampling sounds powerful, but what happens when the field you care about has really high cardinality? Let's take something like user_id. Even if your system only sees fifty thousand users, that's still an enormous number of unique values to try and manage in a dynamic sampling keyspace. Here's the key: high cardinality data is often already well-represented in sampled data, as long as you're not undersampling. For example, if you have five thousand active users in a ten-minute window, and you look at sampled data over a ten-second slice, you might not see all users. But over a one-minute slice, you probably will, with a small margin of error. Remember–the goal of sampling is to keep slices of your telemetry data that are important to you, even though some of it will get dropped. With the right setup, the data you lose is balanced out by the data that is kept–within a small margin of error. This means you don't need to base your dynamic sampling on high-cardinality fields like user_id. Instead, focus on fields that represent the type of traffic you want to be distinctively sampled, not on the highest fidelity or the highest cardinality fields, because that will cause the dynamic sampler to keep a lot more of your traffic, and you won't reduce the volume of your telemetry at all.

Now, what if you really do need high fidelity for a high cardinality field? Maybe you're tracking specific user activity or login failures for security monitoring. Here's the reality: sampling is a cost-saving tradeoff. If the value of full fidelity outweighs the cost of storing and processing that data, then don't sample it. This can be the right answer if and when your telemetry data has enough value.

Dynamic sampling isn't magic. It can't give you both low volume and high fidelity for every high cardinality field at the same time. But here's the better question: Do you need high fidelity everywhere? Or only for certain types of traffic? Let's say you're a software engineer who works for an online dating app. You want to see that users' attempts to change their profiles are working as expected. With write operations, there are many different steps before the write starts and at the end of this operation, it must have worked. When you decide if you want to keep or drop traces, you want to keep the traces that tell you whether the write operation worked. This is high value information. In contrast, read operations are far more common and far less impactful. The client should be able to retry these operations without the user even noticing. This is where tail sampling comes in! Rather than just relying on dynamically sampling all of your traffic, you can create rules. A rule that drops lots of traffic can apply to low-value fields. A rule that keeps lots–or all–of traffic can apply to high-value fields. That way, you balance cost and fidelity with intent.

So dynamic sampling is a powerful tool, but it's just one part of the sampling toolbox. When used wisely alongside tail sampling, it helps you reduce volume while keeping the visibility that matters most.

To recap: High-cardinality fields like user_id don't always need to be used in dynamic sampling; sampling over time typically captures enough variety without adding extra cost. If you truly need high fidelity for specific traffic, like specific user activity or security events, consider not sampling that data or applying targeted tail sampling rules. Strategic sampling means using dynamic sampling for general traffic and reserving high-fidelity approaches only where the business value justifies the cost.

*(Note: this written script uses an "online dating app profile-write" example. The storyboard for this activity — `sampling-and-fidelity-tradeoffs-storyboard.md` — uses "payment transactions" instead. Payment transactions was selected as canonical for the workshop deck.)*
