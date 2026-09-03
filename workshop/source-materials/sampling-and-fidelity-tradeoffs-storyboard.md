# Sampling and Fidelity Tradeoffs (Storyboard)

Source: "Sampling and Fidelity Tradeoffs Storyboard.pdf" — Section 1, Activity 4

## Panel 1
**Audio:** "Dynamic sampling sounds powerful, but what happens when the field you care about has really high cardinality?"
**Visual:** Character thought bubble: "What about user ID?"

## Panel 2 (header mislabeled "What is Dynamic Sampling?")
**Audio:** "Let's take something like user_id. Even if your system only sees fifty thousand users, that's still an enormous number of unique values to try and manage in a dynamic sampling keyspace."
**Visual:** User_ID keys (Key A, B, C... I, J, F, D, G, H) rapidly filling the screen, giving a sense of overwhelm.

## Panel 3 (header mislabeled)
**Audio:** "Here's the key: high cardinality data is often already well-represented in sampled data, as long as you're not undersampling. For example, if you have five thousand active users in a ten-minute window, and you look at sampled data over a ten-second slice, you might not see all users."
**Visual:** Dot-scatter chart: "10 minutes" window (dense, many colored dots) vs. "10 seconds" slice (mostly white/unsampled area with a few colored dots visible).

## Panel 4 (header mislabeled)
**Audio:** "But over a one-minute slice, you probably will, with a small margin of error. Remember–the goal of sampling is to keep slices of your telemetry data that are important to you, even though some of it will get dropped. With the right setup, the data you lose is balanced out by the data that is kept–within a small margin of error."
**Visual:** Dot-scatter chart: "1 minute" window showing much better color/dot coverage than the 10-second slice.

## Panel 5 (header mislabeled)
**Audio:** "This means you don't need to base your dynamic sampling on high-cardinality fields like user_id. Instead, focus on fields that represent the type of traffic you want to be distinctively sampled, not on the highest fidelity or the highest cardinality fields, because that will cause the dynamic sampler to keep a lot more of your traffic, and you won't reduce the volume of your telemetry at all."
**Visual:** Checklist slide — "Do: Use mid-cardinality fields (region, status code)" / "Don't: Use high-cardinality fields (user ID, session ID)"

## Panel 6 (header mislabeled)
**Audio:** "Now, what if you really do need high fidelity for a high cardinality field? Maybe you're tracking payment transactions or login failures for security monitoring."
**Visual:** Character thought bubble: "What if I need high fidelity?"

## Panel 7 (header mislabeled)
**Audio:** "Here's the reality: sampling is a cost-saving tradeoff. If the value of full fidelity outweighs the cost of storing and processing that data, then don't sample it. This can be the right answer if and when your telemetry data has enough value. Dynamic sampling isn't magic. It can't give you both low volume and high fidelity for every high cardinality field at the same time."
**Visual:** Seesaw balancing "Cost" and "Fidelity," labeled "Dynamic Sampling" at the fulcrum.

## Panel 8 (header mislabeled)
**Audio:** "But here's the better question: Do you need high fidelity everywhere? Or only for certain types of traffic, like payment processing, because that's where customers give you money?"
**Visual:** "Strategic Fidelity" — two boxes: "Low-value paths (less critical traffic)" and "High-value paths (payments)."

## Panel 9 (header mislabeled)
**Audio:** "This is where tail sampling comes in! Rather than just relying on dynamically sampling all of your traffic, create rules that ensure specific traffic - like your payment processing - get sampled really heavily, or kept completely, and then use the dynamic sampler for the telemetry that has a less stringent need for fidelity of high-cardinality fields. That way, you balance cost and fidelity with intent."
**Visual:** "Strategic Fidelity" flow: "High-value paths (payments)" → Tail sampling; "Low-value paths (less critical traffic)" → Dynamic sampling; both → "Balanced sample set."

## Panel 10 (Recap, header mislabeled)
**Audio:** "To recap: High-cardinality fields like user_id don't always need to be used in dynamic sampling; sampling over time typically captures enough variety without adding extra cost. If you truly need high fidelity for specific traffic, like payments or security events, consider not sampling that data or applying targeted tail sampling rules. Strategic sampling means using dynamic sampling for general traffic and reserving high-fidelity approaches only where the business value justifies the cost."
**Visual:** Standard recap slide.

**Note:** Every panel from #2 onward carries a "What is Dynamic Sampling?" header (copy-paste artifact) instead of "Sampling and Fidelity Tradeoffs" — cosmetic, flagged in `visual-asset-inventory` memory. This storyboard's canonical high-value-traffic example is **payment transactions** (as confirmed for the workshop deck), whereas the written script (`scripts-section-1.md`) uses an "online dating app profile-write" example instead — the script's example should be rewritten to match when drafting this section.
