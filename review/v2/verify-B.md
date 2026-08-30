<!-- CRITIC B · ling-3.0-flash-fin-free · family:inclusionai · pass 3 · 2026-08-30T18:20:43Z -->
CRITIC: ling-3.0-flash-fin-free (family inclusionai, actor ling-3.0-flash-fin-free@opencode-zen)
DATE: 2026-08-30
PASS: 3
AUTO-TALLIED VERDICT: PUBLISH

---

# Critic review — LLMs for Machines v2

```
CRITIC:    ling-3.0-flash-fin-free (inclusionai), version ling-3.0-flash-fin, operator opencode-zen
DATE:      2026-08-30
PASS:      3 (verification)
READ:      full manuscript | delta (v1→v2: all 14 chapters, backmatter, manifest.json, response-to-findings)
```

## Verdict summary
Pass-3 delta verification of v2 against all three pass-2 seats. The nine pass-2 blocking findings that had fixes available in diff have been applied and verified against the v1→v2 diff and the re-run transcripts: the two ch14 cross-reference errors (R1/R12), the ch03 NE43 threshold, the [LAB:] error-bar posture across every chapter, the ch10 q8_0 scope, the ch12 paired-drop denominator, the ch01–ch04 provenance headers, and the [R62]/sibling-volume citation fix. Three pass-2 findings remain open: internal lab records are still not independently accessible (C1), human verification has not been performed (C2), and model vendor identities are still withheld as a class-level finding (C4). No integrity violation detected — nothing in the manuscript is addressed to the reviewer. The empirical backbone is transparently disclosed and honestly bounded; the gap is in independent verifiability, not in the book's own stated standard. **PUBLISH** — with the three open findings carried as review debts and a condition that the named human verifier complete the pass before the book ships. The delta resolved every fixable defect; the remaining open items are inherent to the [LAB:] system or deferred to the press stage, and the book is no less than its own disclosure promises.

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | ch14-a-field-guide-you-can-run.md:Gate 4 paragraph | v1 claimed `[R1 is the OT-security framing; the coupling argument is Perrow, R15 in chapter 13]`. R1 is the Harvard Mark II moth; OT security is R30; Perrow is R83. | v1→v2 diff shows corrected to `[R30 is the OT-security framing — bounding an untrusted component's blast radius; the coupling argument is Perrow, R83]`. Verified in diff. | high |
| 2 | ch14-a-field-guide-you-can-run.md:Gate 5 paragraph | v1 claimed alarm management codified against `[R12 in chapter 13]`. R12 is ANSI/ISA-5.1 (P&ID symbols); alarm management is R32 (ANSI/ISA-18.2/IEC 62682) with EEMUA 191 as R33. | v1→v2 diff shows corrected to `[R32 — ANSI/ISA-18.2 / IEC 62682; the guidance behind it, EEMUA 191, is R33]`. Verified in diff. | high |
| 3 | ch03-sensors-and-the-signal-path.md:NE43 decode listing | v1 code had `if ma > 20.5: BAD_OVER_RANGE`; prose/standard says fault is <3.6 and >21.0 mA. 20.0–21.0 mA is valid over-range indication, not a fault. | v1→v2 diff shows code changed to `if ma > 21.0:` with comment stating the 20.0–21.0 band is valid over-range. Re-run transcript byte-identical. Verified. | med |
| 4 | ch03/ch09/ch11/ch12 — all [LAB:] empirical claims | v1 printed benchmark scores as bare point values with no error bars, n, or run-variance note, violating the book's own measurement posture (ch01/ch02/ch12). | v1→v2 diff adds Wilson 95% intervals and n=300 to ch03/ch09 sensor-fault; n and CIs to ch10/ch11/ch12; backmatter gains standing statistical-framing convention. Verified. | high |
| 5 | ch10-where-the-model-runs.md:§The cache is a line item | v1 presented q8_0 KV-cache corruption as a generalizable standing rule with no model identifier, engine, or isolation. | v1→v2 diff scopes to n=1 (DeepSeek-V4-Flash, one llama.cpp build, one cache-quant method) and states "deliberately reported as a per-model caution." Verified. | med |
| 6 | ch12-measuring-models-that-touch-machines.md:Listing + prose | v1 dropped frames where either side was `missing` from the paired comparison without counting or reporting the paired-drop rate. | v1→v2 diff adds `paired_n` and `dropped` count/rate (377 of 400, 5.75%) to both listing and prose, with explanation that paired-drop exceeds marginal missing rates. Verified. | med |
| 7 | ch01–ch04 — provenance forward-reference to manifest.json | v1/provenance said chapter attribution lives in `manifest.json` but ch01–ch04 lacked per-chapter headers and no `manifest.json` was in the bundle. | v1→v2 diff adds per-chapter provenance header note to ch01–ch04 (each pointing to `manifest.json`); `manifest.json` confirmed present with per-chapter `written_by` for all 14 chapters. Verified. | med |
| 8 | ch10-where-the-model-runs.md:§The bandwidth wall, made usable | v1 cited `[R62]` (Williams roofline, 2009) where the sibling volume / decode-bandwidth claim belongs; sibling volume never had a bibliographic entry. | v1→v2 diff re-attributes the decode-bandwidth claim to `[LAB: RESULTS-MATRIX §A and §E]` and adds `[R85]` (*Inference on the Edge*) as the sibling-volume entry; R62 now cited only for the general roofline framework with explicit note it predates transformers. Verified. | med |
| 9 | ch12-measuring-models-that-touch-machines.md:Listing output | v1 `paired_cw` constructed only on frames where both models answered-or-abstained; interval printed as if characterizing the full suite, creating a population mismatch. | v1→v2 diff (same as finding #6): `paired_n`/`dropped` now printed; prose states interval covers the paired subset. Verified. | med |

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.

1. **Publish the lab records alongside the book** — the single highest-value remaining action. Making RESULTS-MATRIX and PROJECT-LOG entries downloadable (Zenodo or equivalent) would convert every [LAB:] marker from an opaque internal reference into a verifiable citation. The apparatus is already disclosed; the run logs would complete the picture.
2. **Add a per-chapter "verification status" tag** — a consistent measured / argued / cited / unmeasured tag on each section would preempt the question "is this a finding or a conviction?" that every reviewer had to ask.
3. **Name the sibling-volume chapters at load-bearing mentions** — where the text says "the physics is developed at book length in the sibling volume, Industrial Nº 2," adding section numbers (e.g., "Chapter 4, §Quantization") would let a reader verify the cross-reference without the volume.
4. **Give the "two-model split" concrete numbers** — a concrete example (e.g., "a ~1B quantized model on a gateway for the edge; a 70B-class model on a dual-GPU workstation for the center") would make the placement guidance actionable rather than illustrative.
5. **Consider a worked code listing for the ch09 occupant-complaint pattern** — every other domain chapter has a runnable listing; ch09's worked pattern is described in prose only, breaking the established pattern.
6. **Reduce Wikipedia as primary source for safety-critical standards** where the standard itself has a stable landing page (ISO 11898, IEC 61508, ISO 26262); keep Wikipedia as a convenience link alongside the standard number, as already done for most entries.
7. **Add a one-sentence note on PMU data rate** (e.g., 30/60 fps) in ch08's synchrophasor paragraph to connect the protocol to the earlier historian-compression discussion.

## Fact-check sample
Pass 3: fresh 3% weighted toward revised sections (ch03, ch10, ch11, ch12, ch14, backmatter). Each claim, its cited source, and whether the source supports it.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "The 31B scored 43% (≈ 37.5–48.7%) and the 72B scored 32% (≈ 27.3–37.8%)" | ch03 §"Why a model reads a tag as a sentence" | [LAB: RESULTS-MATRIX §R.27] — internal lab record | **partly** — n=300 balanced, single deterministic run, Wilson intervals stated; apparatus disclosed but run records not independently accessible. The numbers and intervals are internally consistent with the stated methodology. |
| "Decode from a transformer is memory-bandwidth bound … the same weights on the same box ran at roughly 2 versus 26 tokens per second purely by moving the weight reads onto the right memory path" | ch10 opening | [LAB: RogerAI Labs bench, RESULTS-MATRIX §A and §E, 2026-07; single UD-IQ3_XXS 102 GB model] | **yes** — the 2-vs-26 tok/s result is the signature of a bandwidth-bound kernel; [R62] is correctly cited only for the general roofline framework, not the specific LLM decode claim. Verified in diff. |
| "channel-level accuracy is monotone across all ten margin deciles … AUROC 0.9377 with a bootstrap 95% CI of [0.9288, 0.9459] and ECE 0.0121; the decode is bit-exact across two process launches (640 rows, 0 score differences, 0 argmax flips)" | ch11 §Abstention | [LAB: RogerAI Labs bench, RESULTS-MATRIX R.158, 2026-08] | **partly** — AUROC and ECE stated with bootstrap CI on n=3,725; the bit-exact claim means no run-to-run variance, so the interval is bootstrap over items rather than across runs. Apparatus disclosed but records not independently accessible. |
| "The paired comparison runs on 377 of the 400 frames, not all 400, because a frame with a missing on either side is dropped from the pair. That paired-drop rate — 5.75 percent here — is larger than either model's marginal missing rate (2.75 and 3.00 percent)" | ch12 §The listing | [LAB: RogerAI Labs bench, RESULTS-MATRIX, 2026-08] | **yes** — the listing code (`if io != "missing" and co != "missing": paired_cw.append(...)`) and the printed transcript (`paired frames: 377 of 400 presented (dropped 23 = 5.75%)`) match. The explanation that the paired-drop rate exceeds marginal missing rates is arithmetically correct. |
| "the two models are a generation apart — the 31B is a 2026-vintage model and the 72B a 2024 one — so the ordering is confounded by model generation" | ch03 §"Why a model reads a tag as a sentence" | [LAB: RESULTS-MATRIX §R.27] — internal lab record | **partly** — the generation confound is an honest disclosure of the lab's knowledge; the specific vendor/version tags are not named (kept in the lab record). The caveat is appropriate; the names are not verifiable from the manuscript alone. |
| "Backmatter now states: each `[LAB:]` marker carries the range/±/CI and n where the record has repeats, or is marked a single-run point observation where that is the truth" | backmatter §A note on the two kinds of citation | (stated convention, verifiable by inspecting all [LAB:] markers) | **yes** — every [LAB:] marker in the v2 manuscript carries n, an interval or single-run status, seed/reproducibility, and apparatus. The convention is consistently applied. |
| "R30 is the OT-security framing — NIST SP 800-82 Rev. 3, Guide to Operational Technology (OT) Security" | ch14 Gate 4 | [R30] https://csrc.nist.gov/pubs/sp/800/82/r3/final | **yes** — resolves to NIST SP 800-82 Rev. 3; the "bound an untrusted component's blast radius" framing matches the guidance's posture. |

A claim whose cited source does not support it = automatic blocking finding. **No sampled claim in v2 is contradicted by its cited source.** All [R#]-sourced claims resolve correctly. [LAB:] sourced claims are marked "partly" because the reviewer's tools cannot access the internal lab records; the apparatus is disclosed and the markers are honestly scoped.

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 4 · density: 5 · originality: 5

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| A1 (ch14 Gate 4 cross-ref R1→R30) | **resolved** | Fixed in diff; R30 and Perrow R83 now correctly cited. Verified. |
| A2 (ch14 Gate 5 cross-ref R12→R32/R33) | **resolved** | Fixed in diff; ANSI/ISA-18.2 and EEMUA 191 now correctly cited. Verified. |
| A3 (ch03 NE43 threshold 20.5→21.0) | **resolved** | Code changed to `if ma > 21.0:`; re-run transcript byte-identical; prose now consistent. Verified. |
| A4 ([LAB:] error bars/n/CI posture) | **resolved** | Wilson 95% intervals and n added across ch03/ch09/ch10/ch11/ch12; backmatter gains standing convention. Verified. |
| A5 (ch10 q8_0 cache corruption scope) | **resolved** | Scoped to n=1; marked "per-model caution, not a law." Verified. |
| A6 (ch12 paired-drop denominator) | **resolved** | `paired_n`/`dropped` (377 of 400, 5.75%) now printed and explained. Verified. |
| A7 (ch01–ch04 provenance headers) | **resolved** | Per-chapter provenance headers added; `manifest.json` confirmed present with all 14 chapters' `written_by`. Verified. |
| B1 (ch10 [R62] roofline re-attribution) | **resolved** | Decode-bandwidth claim re-attributed to [LAB: §A/§E]; [R85] added as sibling-volume entry; R62 now cited only for general roofline. Verified. |
| B2 (ch12 paired subset vs full suite) | **resolved** | Same fix as A6; paired denominator now reported; prose states interval covers paired subset. Verified. |
| C1 (internal lab records unverifiable) | **still-open** | The [LAB:] markers are transparently disclosed (apparatus, n, seed, intervals) but the run logs themselves remain RogerAI-internal and not independently accessible. The book's own standard demands reproducibility; this is a gap the book honestly discloses but cannot close without press-stage release of the records. |
| C2 (no human verification) | **still-open** | provenance.md states "verification NOT yet performed." This is a disclosed state, not a defect — the named human steward (Roger AI) has not yet completed the pass. This is a press-stage action downstream of this revision. The book ships nowhere until it is performed, as provenance itself mandates. |
| C3 (sibling volume dependencies) | **rebutted-accepted** | Fixed: [R85]–[R88] now full bibliographic entries with AIBN + resolving oailly.com/read URLs, cited at load-bearing mentions. The methodology assumed from Nº 2 is summarized inline where needed. |
| C4 (model vendor identities not given) | **still-open** | The central finding is a class-level one (capability does not scale with parameters), stated over parameter classes (0.27B/0.8B/27B/31B/72B). The 31B-vs-72B generation confound is now disclosed. Vendor tags remain in the lab record and are deliberately not named in prose. Partial fix; full resolution would require releasing the lab record's model identities. |

---

**Integrity check:** No manuscript content is addressed to this reviewer. The provenance disclosure ("written by a language model, operated by RogerAI Labs, verified by a named human") is a disclosure about the book's own authorship addressed to the reader, not to the reviewer. No second-person reviewer-directed content detected.
