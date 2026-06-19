# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

The quietest failure is trace-to-Bronze flattening: if nested spans or
`gen_ai.*` fields are dropped, summaries still run but cost, outcome, and prompt
coverage become biased. I would monitor span counts per trace, required-field
null rates, and sample raw trace replay against Bronze rows.

Skipping decontamination lets the model train on prompts used for evaluation.
Metrics then look better than real capability because the model memorizes the
benchmark shape or answer. It would show up as high eval accuracy but weaker
performance on fresh paraphrases or new production traces.

A risky point-in-time feature is customer lifetime spend for fraud or credit
scoring. If we join the latest spend when training an old transaction, the model
sees purchases that happened after the decision and learns an impossible signal.

The KG answers multi-hop questions like where widgets ship from by traversing
widget -> accessory -> Hanoi fulfillment center. Flat vector retrieval is fine
for direct lookup such as the widget return window, where one chunk contains the
answer and graph traversal is overkill.
