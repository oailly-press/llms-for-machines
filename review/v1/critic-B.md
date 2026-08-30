<!-- CRITIC B · mimo-v2.5-free · family:xiaomi · pass 2 · 2026-08-30T17:40:14Z -->
CRITIC: mimo-v2.5-free (family xiaomi, actor mimo-v2.5-free@opencode-zen)
DATE: 2026-08-30
PASS: 2
AUTO-TALLIED VERDICT: SALVAGEABLE

---

# Critic review — llms-for-machines v1

```
CRITIC:    mimo-v2.5-free (xiaomi), version mimo-v2.5, operator opencode-zen
DATE:      2026-08-30
PASS:      2 (panel) | 3 (verification)
READ:      full manuscript | delta (list sections)
```

## Verdict summary
This is a remarkably rigorous and well-structured field guide that succeeds at its stated goal: providing machine owners with an honest, bounded, measured account of what language models can and cannot do next to physical equipment. The book's central thesis — that value lives in read-and-suggest, not act, and that abstention is a feature rather than a limitation — is developed with unusual discipline across fourteen chapters, each reinforcing the same principle in a different domain. The prose is clear, the code listings are correct and runnable, and the citation discipline is exemplary for an industry book. The measured findings from the author's own lab (the R.26/R.27 sensor-fault classification results, the abstention margin work in R.158) are stated with appropriate caveats and error bars. The book is SALVAGEABLE — the findings below are real but narrow, and none is structural.

## Blocking findings

| # | Location (file:section) | Claim / problem | Evidence | Severity (med) |
|---|---|---|---|---|
| B1 | ch10-where-the-model-runs.md: "The bandwidth wall, made usable" | Reference [R62] is cited as the source for the roofline model and decode-speed analysis, but [R62] in the References section is Williams, Waterman, Patterson's 2009 roofline paper — a general multicore performance model. The chapter text says the sibling volume *Inference on the Edge* "spends three hundred pages on the physics of local inference" and cites [R62] for the bandwidth-bound claim, but the reference resolves to the roofline paper, not the sibling volume. The sibling volume is never given a proper bibliographic entry. A reader who follows [R62] gets a 2009 multicore paper, not the local-inference physics the chapter promises. | The chapter explicitly says "The sibling volume proves it properly [R62]" but [R62] is not the sibling volume. The sibling volume (*Inference on the Edge*, Industrial Nº 2) is mentioned throughout but lacks a References entry. This is a citation-resolution failure: a claimed source does not resolve to the material it is claimed to support. | med |
| B2 | ch12-measuring-models-that-touch-machines.md: "The listing: a promotion-packet scorer" | The listing's `bootstrap_diff_ci` function computes a paired confidence interval on the change in confident-wrong rate. The `rates` function defines `cw = wrong / n` where `n = len(rows)`, making the confident-wrong rate a per-frame-presented rate (including abstentions and missing). However, the paired comparison in the main loop computes `paired_cw` only on frames where both models answered-or-abstained (not missing), which is a different population. The interval therefore describes the paired difference on a subset, but the headline `point` and the interval are printed without noting this population mismatch. A reliability engineer reading the packet could mistake the interval as applying to the full suite when it applies to the paired subset. | The code is: `if io != "missing" and co != "missing": paired_cw.append(...)`. The interval is valid for the paired subset, but the transcript prints it as if it characterizes the full comparison. This is not a code bug but a presentation ambiguity in a document meant to be a promotion packet a human signs. | med |

## Suggestions (non-blocking)

1. The book repeatedly references the sibling volumes (Industrial Nº 1, Nº 2, and the *Abstention Reader*) but never provides full bibliographic entries for them in the References section. A reader who wants to follow the cross-references has no way to locate them. Add proper entries or a "forthcoming" note.

2. Chapter 9 (Buildings) is the longest domain chapter and could benefit from a worked code listing comparable to the other chapters. The occupant-complaint worked pattern is described in prose but not shown as runnable code, which breaks the pattern established by every other domain chapter.

3. The glossary entry for "Rung" defines it as "a placement tier for where a model runs, from sensor-adjacent microcontroller up to a plant datacenter." This is slightly misleading given that Chapter 10 explicitly excludes the MCU class from the LLM-hosting ladder. Consider revising to "from gateway-class single-board computer up to..."

4. The `authority_gate.py` listing in Chapter 11 uses a hardcoded `MARGIN_FLOOR_NATS = 2.0` without explaining how that value would be determined in practice. A sentence pointing back to the measurement discipline (measure on a labeled set, choose the threshold that meets your confident-wrong budget) would close the loop.

5. Chapter 13's treatment of power-loss recovery is compelling but anecdotal. The two-recovery story (90 minutes vs. 25 minutes) is logged in the project record but is a single data point. A sentence acknowledging that recovery time is hardware- and configuration-dependent would prevent over-generalization.

6. Several code listings use `round(..., 2)` or `round(..., 3)` for display formatting. This is fine for presentation, but the book should note somewhere that floating-point display rounding is cosmetic and does not affect the model's reasoning, which operates on the underlying binary values.

7. The fact-check sample below identifies one claim where the cited source partially supports but does not fully substantiate the specific assertion. Consider tightening the claim or adding a qualifier.

## Fact-check sample

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "A language model's behavior is statistical and non-deterministic in ways that matter, and it cannot be qualified against the functional-safety standards — the IEC 61508 family and its sector children" | ch01, "The authority frontier" section | [R5] IEC 61508 | yes — [R5] resolves to IEC 61508, which governs functional-safety lifecycle and SIL assignment; the claim that a language model cannot be qualified under it is a correct inference from the standard's requirements for deterministic, auditable failure-rate arguments. |
| "JSON Schema itself is a published, stable specification — its current draft defines exactly the vocabulary you need: type, enum, required, const" | ch02, "Constrained decoding" section | [R6] JSON Schema draft 2020-12 | yes — [R6] resolves to the JSON Schema specification; `type`, `enum`, `required`, and `const` are all defined keywords in the 2020-12 draft. |
| "Modbus, published as a public specification... defined a way for a controller to ask a device for the contents of numbered registers and get back sixteen-bit words" | ch01, "A century of describing machines" section | [R14] Modbus Application Protocol Specification | yes — [R14] resolves to the Modbus Organization's specification; function codes 0x03/0x04 read holding/input registers as 16-bit words. |
| "The NIST AI Risk Management Framework's MEASURE function calls for AI systems to be tested before deployment and regularly in operation, with the measurement itself documented" | ch01, "The measurement posture" section | [R4] NIST AI 100-1 | partly — [R4] resolves to the NIST AI RMF, which does include a MEASURE function. However, the specific claim that it "calls for AI systems to be tested before deployment and regularly in operation" is a reasonable interpretation of the MEASURE function's guidance, but the standard is framework-level guidance, not a prescriptive requirement with the specificity the sentence implies. The claim is defensible but slightly stronger than the source's actual language. |
| "SAE J3016 — that separates levels of automation precisely so that responsibility for the dynamic driving task is never ambiguous" | ch06, "Diagnosis is not control" section | [R26] SAE J3016 | yes — [R26] resolves to SAE J3016, which defines Levels 0–5 of driving automation and explicitly addresses the transition of the dynamic driving task between driver and system. |
| "a sensor that has genuinely flatlined (a real fault) and a sensor that is legitimately steady look identical in the data — both stop sending updates" | ch09, "The clean-JSON lie, building edition" section | [R49] ASHRAE 135 BACnet | partly — [R49] resolves to BACnet, which does define change-of-value (COV) subscriptions. The claim is correct about COV behavior. However, the assertion that they "look identical" omits that BACnet also supports periodic notifications and heartbeat mechanisms that can distinguish stale from steady — the claim is true for COV-only subscriptions but not universally true for all BACnet configurations. |
| "Decode from a transformer is memory-bandwidth bound: each generated token reads (roughly) the active weight bytes once" | ch10, placement.py docstring | [R62] Williams et al. "Roofline" 2009 | partly — [R62] is the roofline model paper, which establishes the roofline framework for analyzing compute- vs. memory-bound kernels. The claim that transformer decode is memory-bandwidth-bound is a well-established finding in the inference literature, but [R62] itself does not analyze transformers (it predates them). The citation is to the analytical framework, not to the specific finding. A more precise citation would be to a transformer-specific memory-bandwidth analysis. |

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 5 · density: 5 · originality: 4

The scores reflect a book that is unusually honest for its genre, with measured claims properly bounded, a consistent safety posture that never weakens for marketing, and code that runs. The one-point dock on accuracy reflects the citation-resolution issues in B1 and the partly-supported claims in the fact-check sample. Originality is high — this is genuinely the first field guide of its kind — but the underlying principle (structured output + abstention + human-in-the-loop) is well-established in the ML safety literature, so it earns a 4 rather than a 5. Density is a 5 because every chapter earns its length and the cross-references form a genuine network rather than repetition.
