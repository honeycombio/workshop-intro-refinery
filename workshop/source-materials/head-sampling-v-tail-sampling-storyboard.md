# Head Sampling v. Tail Sampling (Storyboard)

Source: "Head Sampling v. Tail Sampling Storyboard.pdf" — Section 1, Activity 2

## Panel 1
**Audio:** "When you're sampling telemetry, what if you really need to make sure you capture certain data, like errors or payment transactions? That's where different types of sampling strategies come into play. There are two main kinds: head sampling and tail sampling."
**Visual:** Character (Engie) thought bubble: "I want to sample my telemetry data, but I need to keep error data. How can I do that?"

## Panel 2
**Audio:** "Let's start with head sampling. This approach makes sampling decisions early either before the telemetry is generated, or by quickly comparing something like a trace ID–which ensures all spans in a trace stay together–against a hash to determine if all spans in that trace should be sampled or dropped. It's fast and computationally cheap. That's because the decision logic is simple and happens immediately—no need to store or process any trace data first."
**Visual:** Trace IDs move through a "HEAD SAMPLER" box which flashes "KEEP" and lets ~50% through randomly.

## Panel 3
**Audio:** "On the other end, we have tail sampling. This method makes sampling decisions after the telemetry has been received, once the entire trace is available. By inspecting the data first, you can ensure that you keep the most critical data, but drop a larger portion of less critical data. This, of course, is much more computationally intensive, because you need to keep the telemetry until a full trace has arrived, and then you need to parse all the attributes in the trace to know how to apply your sampling decisions. With really high volumes of data, tail sampling can be discouragingly expensive on its own."
**Visual:** Traces move through a "TAIL SAMPLER" box showing "ERROR: TRUE / PAYMENT: TRUE" — scans for attributes, keeps those with errors/payment info, drops ~half that don't meet criteria.

## Panel 4
**Audio:** "So how do teams balance the two? Head sampling is fast but uninformed; it doesn't look at the trace content. Tail sampling is expensive but smart; it uses the actual data to guide decisions. Because of that tradeoff, many companies combine both. They'll use head sampling to reduce overall volume, and tail sampling on top of that to retain the most important traces."
**Visual:** Two boxes: "Head Sampling" (labeled "Fast, cheap, and uninformed") vs. "Tail Sampling" (labeled "Expensive, smart"). Character with thought bubble: "Looks like we could use both!"

## Panel 5
**Audio:** "Here's the key idea: when you have high trace volume, individual traces become less critical. That's because any important pattern is likely to repeat. So by head sampling just 10 to 50 percent of your traffic, you reduce what gets passed to tail sampling, making it feasible to analyze and selectively keep the most valuable traces. That's the tradeoff. Head sampling gives you speed and scale. Tail sampling gives you control and precision. Together, they help ensure you get the data you care about, without blowing up cost or infrastructure."
**Visual:** Trace IDs move in bulk through head sampler (flashing keep/drop quickly), then a smaller volume moves more slowly through the tail sampler (reads attributes to decide keep/drop). End result: fewer traces with higher proportion of wanted attributes.

## Panel 6 (Recap)
**Audio:** "To recap: Head sampling is fast and cost-effective, making quick decisions before or immediately after telemetry is generated. Tail sampling inspects full trace data after it's collected, allowing precise retention of important traces—but it's significantly more resource-intensive. Combining both allows teams to balance scale and fidelity: head sampling limits volume, while tail sampling preserves critical data like errors or payment traces."
**Visual:** Standard recap slide.
