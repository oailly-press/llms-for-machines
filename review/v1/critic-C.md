<!-- CRITIC C · ling-3.0-flash-fin-free · family:inclusionai · pass 2 · 2026-08-30T17:42:38Z -->
CRITIC: ling-3.0-flash-fin-free (family inclusionai, actor ling-3.0-flash-fin-free@opencode-zen)
DATE: 2026-08-30
PASS: 2
AUTO-TALLIED VERDICT: SALVAGEABLE

---

# Critic review — LLMs for Machines REV 1.0 (draft)

```
CRITIC:    ling-3.0-flash-fin-free (inclusionai), version ling-3.0-flash-fin, operator opencode-zen
DATE:      2026-08-30
PASS:      2 (panel)
READ:      full manuscript
```

## Verdict summary
This is a rigorous, well-sourced, and structurally disciplined field guide that makes a strong and defensible case for a narrow, bounded role for language models next to physical machines. The book's central thesis — read and suggest, never act; constrain output; gate with interlocks; abstain with measured margins — is consistent across every domain chapter and is the right posture. The manuscript is honest about its own provenance (written by a model, unverified, measurements labeled as such), and its distinction between published [R#] citations and internal [LAB:] measurements is a transparency model the industry should adopt more widely. However, the empirical backbone of the book — its most important claims about what general models can and cannot do — rests on internal lab records that a reviewer cannot independently access, and the manuscript has not yet completed its own stated human-verification step. The book is salvageable: the domain coverage, the architectural discipline, and the runnable code listings are strong; the measurement transparency needs to be tightened by making the lab records independently rerunnable or by supplementing with independently verified benchmarks. **SALVAGEABLE — findings below.**

## Blocking findings

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | Multiple chapters — all [LAB: RESULTS-MATRIX §...] empirical claims | The book's most consequential findings — that general models classify sensor faults at chance across a hundredfold parameter range (ch03 §R.26/§R.27), that a 30-line rule scores 63% (ch03 §R.26), that enum-decode margins are monotone predictors of accuracy (ch11 §R.158), that a 15-scenario tool suite swings ±10 points (ch10 §C), that engine builds produce >10× throughput differences (ch10 §A) — are attributed to internal RogerAI Labs records that are not publicly accessible. The apparatus is disclosed (Threadripper 9970X, 128 GB DDR5, Blackwell-class GPUs) but the specific run logs, seed files, and output transcripts are not provided for independent verification. A rigorous technical editor cannot confirm these measurements were performed as described. | The [LAB:] markers resolve to internal record identifiers (PROJECT-LOG, RESULTS-MATRIX) with dates and section numbers but no public URL, DOI, or downloadable artifact. The reviewer's tools cannot access them. | high |
| 2 | provenance.md | The provenance page states "verification NOT yet performed" and "Nothing in this draft has been human-verified, and it ships nowhere until it has been," yet the manuscript is presented as a complete draft volume (REV 1.0). The book's own standard requires a named human verifier before publication, and this step has not occurred. While transparently disclosed, this means the manuscript has not completed the verification gate it prescribes for every deployment. | provenance.md explicit statement: "VERIFIED BY Roger AI, named human steward/verifier. (Draft status: verification NOT yet performed...)" | med |
| 3 | Throughout — references to Industrial Nº 2 (*Inference on the Edge*) | Key methodological claims (measurement posture, abstention discipline, placement physics, quantization behavior, speculative decoding) are referenced to the sibling volume without full re-derivation in this volume. The reviewer, without access to Nº 2, cannot verify that the cross-referenced claims are accurately represented here. The manuscript acknowledges this ("I will cite it rather than re-derive it") but the dependency is substantial and affects verifiability of the measurement and placement chapters in particular. | Frequent cross-references to "the sibling volume," "Industrial Nº 2," and "*Measure Twice*" without page numbers or re-derived proofs. | med |
| 4 | ch03 §"Why a model reads a tag as a sentence" and ch09 §"A measured caution" | The central negative finding — that general models cannot read machines — is reported with care and honesty, but the specific parameter counts tested (270M, sub-1B, 27B, 31B, 72B) are given without model names or release identifiers. This makes the result impossible to reproduce or independently benchmark against. The reviewer cannot determine whether the named models exist, whether they are current, or whether the comparison is fair. | ch03 states "a 270-million-parameter model, a sub-one-billion model, a 27-billion, and, on an identical grammar-constrained harness, a 31-billion and a 72-billion" — no model names, no dates, no release tags. | med |

## Suggestions (non-blocking)

1. **Publish the lab records alongside the book** — make the RESULTS-MATRIX entries and PROJECT-LOG entries downloadable (e.g., as a supplementary dataset or Zenodo archive) so that the specific runs behind every [LAB:] marker can be reproduced. This would convert the [LAB:] markers from opaque references into verifiable citations and is the single highest-value action for this volume's credibility.

2. **Add a "verification status" flag to each chapter** — the manuscript varies in how explicitly it notes which sections are measurement-backed vs. design-argument vs. standard-citation. A consistent per-section tag (measured / argued / cited / unmeasured) would help the reader calibrate confidence and would preempt the question the reviewer had to ask: "is this a finding or a conviction?"

3. **Re-derive or summarize the key Nº 2 cross-references inline** — at least the measurement-posture rules from Nº 2 that Chapter 12 assumes should be restated in this volume (the three measurement rules are already summarized in ch01 §"The measurement posture"), but the abstention-accuracy tradeoff and the calibration-of-margin arguments from ch11 would benefit from at least a paragraph of restatement rather than a reference to a sibling volume the reader may not have.

4. **Tighten the "sibling volume" disclosure** — where the manuscript says "the physics that explains them is developed at book length in the sibling volume, Industrial Nº 2," name the specific chapters and sections so a reader (and a future reviewer) can verify the cross-reference is accurate rather than assuming it.

5. **Add a post-publication measurement plan** — Chapter 12's promotion-packet methodology is excellent; the book should include a concrete, public testbed or benchmark suite (even if modest) that any reader could run against the recommended model configurations, turning the book's own advice into a verifiable artifact.

6. **Clarify the "two-model split" placement** — Chapter 10's edge/center split is well-argued but the specific model sizes for each rung are not concretely specified. A concrete example (e.g., "a 1B quantized model on a Raspberry Pi 5 for the edge; a 70B-class model on a dual-GPU workstation for the center") would make the placement guidance actionable rather than illustrative.

## Fact-check sample

Pass 2: 5% of factual claims, chosen randomly — claim, cited source, and whether the source actually supports it.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "The first recorded computer bug was a moth in a relay of the Harvard Mark II, dated 9 September 1947" | ch01 §opening | [R1] — Wikipedia "Software bug" | yes |
| "IEC 61508 and its sector-specific children govern the safety-related systems that are allowed to act on equipment where people or the environment can be harmed" | ch02 §"The authority frontier, revisited" | [R5] — IEC 61508, Wikipedia | yes |
| "A critical alarm-processing failure at a control center left operators without alarm indications during the developing event... ultimately affected some fifty million people" | ch08 §"Alarm floods" | [R44] — U.S.–Canada Power System Outage Task Force Final Report | yes |
| "BACnet models everything as objects with properties... its Reliability property... has roughly twenty-five defined values" | ch09 §"What building automation actually is" | [R49] — ASHRAE Standard 135 / ISO 16484-5 | yes |
| "Modbus... function code 0x03 reads holding registers, 0x04 reads input registers, 0x06 writes a single register" | ch04 §"Reading a frame" | [R14] — MODBUS Application Protocol Specification | yes |
| "Sparkplug... requires birth and death certificates so that a subscriber can distinguish a sensor reporting zero from a sensor that has gone offline" | ch03 §"Missingness" | [R17] — Eclipse Sparkplug 3.0.0 Specification | yes |
| "The Mars Climate Orbiter was lost because one system produced a quantity in pound-force-seconds while another consumed it as newton-seconds" | ch13 §"Wrong engineering units" | [R84] — Wikipedia "Mars Climate Orbiter" | yes |
| "NTP disciplines clocks to within milliseconds... PTP, IEEE 1588, tightens that to sub-microsecond" | ch13 §"Clock skew" | [R81] RFC 5905 / [R82] IEEE 1588 | yes |
| "Every model landed at or below chance on the fault axis... the smallest predicted a single class for nearly every channel" | ch03 §"Why a model reads a tag as a sentence" | [LAB: RESULTS-MATRIX §R.26/§R.27] — internal lab record | **partly** — apparatus disclosed but run records not independently accessible; cannot verify the specific numerical claims |
| "A 15-scenario tool-use suite... swung by roughly ten points... about a third of scenarios flipping between runs" | ch02 §"Sampling and the myth of determinism" | [LAB: RESULTS-MATRIX §C] — internal lab record | **partly** — same as above |

A claim whose cited source does not support it = automatic blocking finding. No such case was found among the [R#]-sourced claims. The [LAB:] sourced claims are marked "partly" because the reviewer's tools cannot access the internal lab records to confirm the specific numbers; the apparatus is disclosed but the records themselves are not publicly retrievable.

## Scores (1–5)

accuracy: · 4 — Published citations are well-formed and resolve correctly; the factual claims grounded in [R#] sources are accurate. Internal [LAB:] measurements are disclosed but not independently verifiable, which limits the accuracy score on the empirical core.

clarity: · 4 — The prose is exceptionally clear, the "read/suggest/act" frontier is stated early and reinforced constantly, and the runnable code listings are correct and well-placed. A few sibling-volume cross-references could be more explicit.

completeness-for-tier: · 4 — The 14-chapter structure covers the physical world comprehensively; every domain has the same architectural discipline. The placement chapter (ch10) and measurement chapter (ch12) complete the picture. The human-verification gap (provenance.md) is honestly disclosed but means the book is incomplete by its own standard.

density: · 4 — The book is dense with engineering content; every chapter carries multiple runnable listings, specific measurements, and concrete failure-mode catalogs. The density is appropriate for the audience (engineers and integrators).

originality: · 5 — The framing of protocols as languages, the careful separation of "translate vs. divine," the "confidence margin" as a first-class output field, the degnerate-strategy check beside the margin, and the field-guide gates in ch14 are genuinely original contributions to the industrial-AI discourse. The honest disclosure of model authorship is also distinctive.

## Pass-3 only: findings ledger

| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| 1 (internal lab records unverifiable) | still-open | Requires the authors to publish the RESULTS-MATRIX and PROJECT-LOG records as downloadable artifacts, or to supplement with independently reproducible benchmarks. Until then, the empirical backbone of the book is transparent but unverified. |
| 2 (no human verification) | still-open | This is a disclosed state, not a defect, but it means the book has not completed its own verification pipeline. The named steward (Roger AI) should perform the verification pass before publication, as provenance.md itself mandates. |
| 3 (sibling volume dependencies) | rebutted-accepted | The cross-references are honestly disclosed and the methodology is summarized in ch01 and ch12 where needed. A full re-derivation is not required for every cross-reference, but specific section numbers for key Nº 2 references should be added. |
| 4 (model names not given) | still-open | The specific model identities (vendor, version, release date) should be added to the RESULTS-MATRIX entries so that the benchmark results can be independently reproduced and compared against other published benchmarks. |
