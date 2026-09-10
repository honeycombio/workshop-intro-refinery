# Final Deck Transcript — Effective Sampling with Honeycomb Refinery Workshop (60 slides)

> Text-only transcript of the final 60-slide deck PDF the user shared on 2026-09-10 (`Effective Sampling with Honeycomb Refinery - Workshop.pptx`), built via Claude Desktop's `/honeycomb-slides` skill from `honeycomb-slides-prompt.md`. This is the version `facilitator-guide.md`'s slide numbers are synced against. Saved to disk immediately per standing practice. Compare against `current-deck-transcript.md` (the 52-slide "before" version) to see exactly what changed — several original slides were lightly edited/condensed and gained images beyond the 7 new slides that were actually requested; see the "Known deck deviations" note at the top of `facilitator-guide.md` for the specific list.

1. Effective Sampling with Honeycomb Refinery (title, dark ring background)
2. Today's Agenda — 6-item bulleted list (Sampling Fundamentals, Rules-Based Sampling, Dynamic Sampling, Cardinality & Limits, Sensible Defaults & Wrap-up, Q&A)
3. Instruqt Lab — "[link here]" placeholder (dark ring background) — **not part of the original request; needs a real link pasted in or removal**
4. What Is Sampling? — title + What/Why/How objectives cards
5. A Century-Old Idea — brewery photo + lossy-compression image comparison; condensed text (dropped explicit "T-Test" naming and the "predict with confidence" sentence versus the pre-edit version)
6. Sampling Your Telemetry — dot-scatter images added; dropped one explanatory sentence versus pre-edit version
7. Won't Sampling Affect My Ability to Query My Data? — same content as pre-edit, reformatted with bold emphasis
8. Sample Rate 1000, In Practice — unchanged (3-card layout)
9. The Biggest Reason for Sampling: Cost — unchanged
10. Key Takeaways (recap) — unchanged
11. Head Sampling v. Tail Sampling (title)
12. Head vs. Tail, at a Glance — unchanged (2-column comparison)
13. Why Not Both? — unchanged content, "Recap" relabeled "In summary" with bullets
14. Refinery Architecture (title)
15. Deploying Refinery — unchanged, includes the trace-routing-requirement callout
16. Standalone or Clustered? — unchanged
17. Clustering Isn't Always Better — But Dynamic Sampling Still Wins — unchanged
18. Resource Planning — unchanged (Memory/CPU/Network 3-card)
19. Recap (Refinery Architecture) — unchanged
20. How Refinery Processes Data (title)
21. From Ingest to Trace Cache — unchanged (OTLP/Honeycomb events)
22. Caching Until the Trace Is Ready — unchanged (3 triggers)
23. Refinery Only Samples Whole Traces — unchanged
24. Samplers, and the Rules That Combine Them — unchanged (deterministic/dynamic + RulesBasedSampler)
25. Build Rules to Sample Smart — 5-card example rule set; **dropped concrete numbers** on two cards: "Long Duration" lost "(e.g. >5s)", "Normal Traffic" lost "(goal: 1 in 10)"
26. Recap (How Refinery Processes Data) — unchanged
27. **Instruqt Challenge 1: Explore Your Refinery Pipeline** — new segue slide, matches requested brief, callout "Go explore!"
28. Refinery Configuration and Rules Files (title)
29. Sample Configuration Options — unchanged (YAML block)
30. Key Configuration Options — unchanged (4-card: Networking/Memory & Buffering/Monitoring & Debugging/Trace Processing)
31. Refinery Rules — unchanged (sample YAML + side text)
32. How Rules Match — unchanged
33. What a Rule Does With a Match — unchanged
34. Recap — **dropped** the "which we'll cover next" transition cue and the "see docs.honeycomb.io for the full detail" pointer
35. **Instruqt Challenge 2: Using Rules to Change How Data Is Sampled** — new segue slide, matches requested brief, callout "Time to write a rule!"
36. What Is Dynamic Sampling? (title)
37. Sampling That Adjusts Automatically — unchanged
38. Keys, Volume, and the Goal Sample Rate — unchanged (Trace A/B key examples)
39. How Refinery Tracks Frequency — unchanged
40. Rebalancing Without Manual Rules — unchanged
41. The Recommended Dynamic Sampler — unchanged (YAML + side text)
42. Recap — unchanged
43. **Instruqt Challenge 3: Adding a Dynamic Sampler** — new segue slide, matches requested brief, callout "Time to add a sampler!"
44. Sampling and Fidelity Tradeoffs (title)
45. High Cardinality Isn't Always a Problem — unchanged
46. Choosing FieldList Fields — unchanged (Do/Don't)
47. What If You Really Need High Fidelity? — unchanged
48. Strategic Fidelity — unchanged
49. Recap — unchanged
50. **Instruqt Challenge 4: High Cardinality in Dynamic Samplers** — new segue slide, matches requested brief (doesn't spoil the outcome), callout "Watch what happens when you add a field to your FieldList!"
51. Sensible Default Refinery Rules (title)
52. Rules Outlined — unchanged (7-item list, gRPC bullet still correctly reads "< 2")
53. Keep Errors and 500s — unchanged (covers both the 500-status and error-field rules on one slide)
54. Drop Health Checks — unchanged
55. Keep Long Duration Traces — unchanged
56. Dynamically Sample Good HTTP and gRPC Traffic — YAML unchanged (gRPC `Value: 2` confirmed correct) but **the entire explanatory paragraph about the `root.` prefix and cardinality was dropped**
57. The Catchall Rule — unchanged
58. Recap (Sensible Default Refinery Rules) — unchanged (4-card numbered grid)
59. **What You Built Today** — new workshop-level recap slide, matches requested brief (5-card grid)
60. Observability for what comes next (brand closer, unchanged)
