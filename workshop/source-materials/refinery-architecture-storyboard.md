# Refinery Architecture (Storyboard)

Source: "Refinery Architecture Storyboard.pdf" — Section 2, Activity 1

## Panel 1
**Audio:** "Let's say you're ready to deploy Honeycomb's Refinery to manage your telemetry sampling. What are your deployment options?"
**Visual:** Character, excited, thought bubble: "I'm ready to deploy Honeycomb's Refinery! What are my deployment options?"

## Panel 2
**Audio:** "Refinery can be deployed as a standalone binary or as a cluster of nodes. It runs as a container on platforms like Kubernetes or Amazon ECS, or as a binary on a Linux system like an EC2 instance."
**Visual:** Two-panel diagram: "Refinery Standalone Binary" vs. "Refinery Node Cluster" (5 nodes in a dashed boundary).

## Panel 3
**Audio:** "Here's the most important thing to understand: Refinery makes its best sampling decisions when it can see all of the data in an entire trace, not just individual spans. That means all services that create spans that belong to the same trace must be routed to the same Refinery instance. If you don't do this, you risk keeping part of a trace and dropping the rest—making your observability data incomplete."
**Visual:** Full trace enters the Refinery Standalone Binary; incomplete traces enter different Refinery cluster nodes (marked "Incomplete Trace").

## Panel 4
**Audio:** "So, why would you run a clustered deployment instead of a standalone deployment? It's really about high availability and horizontal scalability. At a certain volume, it doesn't make sense to keep growing one instance larger and larger to handle that volume. Running Refinery as a standalone instance is ideal for smaller telemetry volumes. But if you're dealing with higher volumes, clustered deployments provide horizontal scalability and high availability. Instead of scaling one instance vertically, you add nodes to distribute the load."
**Visual:** "Refinery Standalone Binary" labeled Simple/Lightweight (with flower/bee icons) vs. "Refinery Node Cluster" labeled Scalable/Fault-Tolerant (more flowers).

*Note: the corresponding written script phrases this the opposite way — "why would you run a standalone deployment instead of a clustered deployment" — same content, reversed framing between script and storyboard.*

## Panel 5 (header mislabeled "What is Dynamic Sampling?" — copy-paste artifact, should read "Refinery Architecture")
**Audio:** "Still, clustering isn't always better. In a clustered setup, each Refinery node is only aware of the traffic it receives and can only make sampling decisions on those traces. This means that dynamic samplers can be slightly less effective in a cluster because they see a less complete view of the traffic to build their dynamic sampling keyspace with. So yes, clustering helps with scale, but dynamic samplers work best when they have a full view of traffic."
**Visual:** "Refinery Node Cluster" diagram — "Each node sees local traffic only" / "Less dynamic context" callouts.

## Panel 6 (header also mislabeled "What is Dynamic Sampling?")
**Audio:** "That said, dynamic sampling is still the best sampler type for sampling data that changes as your app changes. It adjusts automatically as your app evolves, without needing constant rule updates."
**Visual:** Seesaw labeled "Dynamic Sampling" balancing "Sample Rate A" and "Sample Rate B" — zoom in on nodes to show dynamic sampling occurring, then a specific node showing real-time sample rate adjustment.

## Panel 7 (header mislabeled)
**Audio:** "Regardless of your deployment choice, you'll need to provision the right computational resources to run Refinery efficiently. That means planning for memory, CPU, and networking bandwidth—especially in a clustered environment. Let's walk through why each one matters."
**Visual:** "Resource Planning" title with three boxes: Memory, CPU, Network.

## Panel 8 (header mislabeled)
**Audio:** "First up: Memory. Memory is the workhorse of Refinery. Since spans in a trace may not arrive all at once, Refinery holds spans in memory until it has enough to make a decision. It also needs memory for internal queues, storing spans during ingest, between nodes, and before forwarding telemetry onward. Recall that tail sampling is resource intensive because you have to store all spans in a trace long enough to make a decision, and then quickly drop or send the telemetry on to its next endpoint. For Refinery to do this, it requires CPU, Memory, and Networking bandwidth (particularly in clustered mode). The most important resource for Refinery is probably memory."
**Visual:** "Resource Planning" — Memory highlighted, sub-items "Holding incomplete traces" / "Queue management"; CPU and Network shown dimmed below.

## Panel 9 (header mislabeled)
**Audio:** "Next is CPU. CPU handles ingesting telemetry from your app and making sampling decisions to determine if a trace is kept or dropped. Refinery is multithreaded and can use multiple cores, but it doesn't scale endlessly. If you see ingest queues backing up while your CPU isn't fully used, that's a sign to add more nodes, not just more cores."
**Visual:** "Resource Planning" — CPU highlighted, sub-items "Multithreaded" / "Watch ingest queue behavior"; Memory and Network dimmed below.

## Panel 10 (header mislabeled)
**Audio:** "Finally, network bandwidth. Bandwidth matters in two ways: First, your Refinery instance needs the bandwidth to ingest telemetry from your services, which can be a lot. Second, Refinery nodes in a cluster also need network bandwidth between each other to forward parts of traces to the correct node so the whole trace can be processed together."
**Visual:** "Resource Planning" — Network highlighted, showing "Inbound Telemetry" arrows and "Internal cluster traffic" with callout "Don't underestimate internal network usage!"; Memory and CPU dimmed below.

*Note: Panels 5–10 all carry a "What is Dynamic Sampling?" header (copy-paste artifact) instead of "Refinery Architecture" — cosmetic, flagged in `visual-asset-inventory` memory.*
