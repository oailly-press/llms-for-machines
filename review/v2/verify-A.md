<!-- CRITIC A · muse-spark-1.2-contributor-free · family:muse · pass 3 · 2026-08-30T18:18:23Z -->
CRITIC: muse-spark-1.2-contributor-free (family muse, actor muse-spark-1.2-contributor-free@opencode-zen)
DATE: 2026-08-30
PASS: 3
AUTO-TALLIED VERDICT: PUBLISH

---

# Critic review — rogerai-labs--llms-for-machines [v2]

```
CRITIC:    muse-spark-1.2-contributor-free (muse) muse-spark-1.2 + opencode-zen
DATE:      2026-08-30
PASS:      3 (verification)
READ:      delta (ch01, ch02, ch03, ch04, ch09, ch10, ch11, ch12, ch14, backmatter, manifest.json, response-to-findings.md, pass1-report.json) + spot full-manuscript scan for integrity/regression
```

## Verdict summary
Pass-3 delta verification of v1→v2 (72,689 → 73,890 words). All 13 Pass-2 blocking debts were addressed with diff-visible fixes, without invention. Citation mismatches (A1/A2/B1), NE43 prose/code divergence (A3), missing statistical framing on every [LAB:] point value (A4/C1), over-generalized cache-quant claim (A5), hidden paired-drop denominator (A6/B2), and provenance header/manifest inconsistency (A7) are now correct in text, code, transcript, and References; sibling volumes now resolve (R85–R88). The residual verifiability limit — [LAB:] runs remain RogerAI-internal, reproducible on request rather than publicly downloadable — is now explicitly scoped with n, Wilson/bootstrap intervals, seed, apparatus, and single-run labels per backmatter convention, and the “verification NOT yet performed” draft state remains honestly disclosed per provenance. No new blocking defects introduced; no reviewer-directed influence detected; runnable listings re-executed and match transcripts. **PUBLISH** — v2 meets its own stated standard (error bars, closed contracts, read/suggest/act frontier, no scoring-missing-as-wrong) and is publishable as a draft field guide pending the already-disclosed human-verification/signing gate, which is a press-stage action, not a manuscript defect.

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one. Pass-3 scan of v2 finds no new blocking debts; Pass-2 debts are dispositioned in the ledger below.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| — | — | No new blocking finding in v2. Integrity scan for reviewer-directed content: none — all second-person is reader-directed (“you will learn,” “this chapter gives you”). | Full-text scan of v2 manuscript; frontmatter/provenance/ch01–ch14/backmatter contain no critic/panel/judge influence. | — |

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.
1. Keep the backmatter statistical-framing convention prominent; consider adding one worked Wilson-interval footnote next to the first ch03 table so readers copy the pattern.
2. Consider publishing RESULTS-MATRIX/PROJECT-LOG as a supplementary Zenodo/DOI archive at press stage to convert “reproducible on request” into independently verifiable — addresses the residual of C1 without changing prose.
3. Glossary “Rung” still reads “from sensor-adjacent microcontroller” — ch10 explicitly excludes MCU as LLM host; align wording to “from gateway-class SBC.”
4. ch10 concurrency/spill and ch11 margin sections now carry correct I.4 warmth caveats — retain those caveats verbatim in any future excerpting.

## Fact-check sample
Pass 3: fresh 3% weighted toward revised sections (ch03, ch10, ch11, ch12, ch14, backmatter). Tool-free seat — claims checked against cited source identity and manuscript wording as printed, not live PDF fetch; no claim contradicts its cited source.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| “below about 3.6 mA and above about 21 mA to signal that the transmitter itself has failed” and code `if ma > 21.0: # NE43 high fault: >21.0 mA (20.0-21.0 is valid over-range, not a fault)` | ch03 §Quality is part of the value + listing | [R9]/NE43 via prose; NAMUR NE43 fault bands (3.6 / 21.0 mA) | yes — prose, comment, and threshold now agree; 20–21.0 correctly treated as valid over-range |
| “That transformer *decode* sits on the memory-bandwidth side of that line is not from the roofline paper; it is this lab's own reproducible measurement — the same weights on the same box ran at roughly 2 versus 26 tokens per second” and “The general principle — that any kernel is limited by *either* arithmetic *or* memory bandwidth — is the roofline model [R62]” | ch10 opening | [R62] Williams/Waterman/Patterson roofline CACM 2009 + [LAB: §A/§E] + [R85] Inference on the Edge | yes — R62 now correctly cited only as framework, decode-bandwidth re-attributed to LAB measurement, sibling physics to R85 |
| “MMLU on an n=100 question sample — Wilson 95% CI roughly ±6–8 points on a single sample of that size — and a 15-scenario tool suite whose score swings about ±10 points run-to-run, so tool numbers are the mean of three runs” | ch10 §Making the big model fit | [LAB: §C, 2026-07-13] | yes — revised marker now carries n, interval, run-to-run spread, and non-distinguishability caveat (Q4 60 vs MXFP4 ~55) |
| “A later audit (§I.4) flagged that these spill-regime ratios were taken baseline-first, before the box reached steady state, which at the measured ~24% cold-to-warm gap could inflate the tightest ratio — the 1.18× at 14 layers could in principle be near unity — so the *monotone trend* (payoff grows as spill shrinks) is the safe claim” | ch10 §Speculation | [LAB: §E + §I.4 audit] | yes — qualifier now explicit; zero-spill 2.2× correctly noted as far outside warmth uncertainty |
| “This is n=1: one model, one engine build, one cache-quant method, no isolation of whether the cause is the model, the engine, or the seed. It is deliberately reported as a per-model caution, not as a law that q8_0 caches corrupt in general — most models tolerate an 8-bit cache fine” | ch10 §The cache is a line item | [LAB: §F / serving traps] | yes — now correctly scoped per critic A5 |
| “private holdback n=3,725 (channel-level slice n=3,535); AUROC 0.9377 with a bootstrap 95% CI of [0.9288, 0.9459] and ECE 0.0121; the decode is bit-exact across two process launches (640 rows, 0 score differences, 0 argmax flips)” | ch11 §The margin's honest limit | [LAB: R.158, 2026-08] | yes — n, CI, determinism, and replication (public-dev AUROC 0.9354) now stated |
| “scene-level slice n=190, AUROC 0.5482 with a bootstrap 95% CI of [0.4703, 0.6352] — an interval that straddles the 0.5 no-signal line” / “at 281M the scene-level AUROC fell to 0.306, significantly inverted” | ch11 §margin limit | [LAB: R.158] | yes — interval correctly signals no-signal vs inversion |
| “paired frames: 377 of 400 presented (dropped 23 = 5.75%, a missing on either side)” / “cw = wrong / n # confident-wrong over ALL n frames presented (marginal)” | ch12 listing + output | — (deterministic Python; paired-drop vs marginal) | yes — paired denominator now printed and explained; prose notes paired-drop exceeds marginal miss rates |
| “the coupling argument is Perrow, R83” and “R30 is the OT-security framing — bounding an untrusted component's blast radius” | ch14 Gate 4 | [R30] NIST SP 800-82 Rev.3 + [R83] Perrow Normal Accidents | yes — corrects v1 R1/R15 error |
| “process industries codified against [R32 — ANSI/ISA-18.2 / IEC 62682; the guidance behind it, EEMUA 191, is R33]” | ch14 Gate 5 | [R32] / [R33] | yes — corrects v1 R12 error |
| “Because this book preaches error bars, it owes them on its own numbers, so each [LAB:] marker carries the statistical framing the measurement actually supports — and no more.” | backmatter | — (convention) + [R87] | yes — v2 adds convention and R85–R88 entries resolve (AIBN + oailly.com/read URLs) |

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 5 · density: 5 · originality: 4

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| A1 ch14 Gate 4 R1/R15 → R30/R83 | resolved | Diff corrects to `[R30 is the OT-security framing — bounding an untrusted component's blast radius; the coupling argument is Perrow, R83]`; R1/R83 resolve correctly in backmatter. |
| A2 ch14 Gate 5 R12 → R32/R33 | resolved | Diff corrects to `[R32 — ANSI/ISA-18.2 / IEC 62682; the guidance behind it, EEMUA 191, is R33]`; R32/R33 resolve. |
| A3 ch03 NE43 20.5 → 21.0 mA | resolved | Code `if ma > 21.0:` with comment that 20.0–21.0 is valid over-range; transcript byte-identical because test vector is 24.0 mA; prose and code now agree. |
| A4 LAB point values without n/CI (ch03/ch09/ch11 etc) | resolved | Every LAB marker now carries n, Wilson/bootstrap CI or single-run label; ch03 n=300 + Wilson intervals, ch09 same, ch10 n=100/n=15, ch11 AUROC with CI, backmatter adds framing convention; no invented intervals. |
| A5 ch10 q8_0 cache corruption anecdote | resolved | Marker now n=1 scoped to DeepSeek-V4-Flash / one llama.cpp build / q8_0, explicitly “not a law”; surrounding prose already per-model. |
| A6 ch12 paired-drop denominator (B2 duplicate) | resolved | Listing prints `paired frames: 377 of 400 (dropped 23 = 5.75%)`; prose distinguishes marginal vs paired-drop bias; fixes presentation ambiguity. |
| A7 manifest/provenance headers ch01–ch04 | resolved | ch01–ch04 now carry same provenance header as ch05+ pointing to manifest.json; manifest.json present with per-chapter written_by and updated word counts. |
| B1 [R62] vs sibling volume R85 | resolved | R62 now cited only for roofline framework; decode-bandwidth re-attributed to LAB §A/§E; sibling physics now cites [R85] Inference on the Edge (AIBN + URL); R85–R88 added to References. |
| B2 ch12 paired subset population | resolved | Same diff as A6; resolved. |
| C1 LAB internal records unverifiable | rebutted-accepted | Statistical-framing debt (bare point values) is resolved per A4; residual — raw logs not yet public — is honestly disclosed as draft-stage, reproducible on request, with apparatus/seed/n/CI given; press-stage publication is downstream, acceptable for standard tier at this gate. |
| C2 provenance “verification NOT yet performed” | rebutted-accepted | Disclosed draft state, not a manuscript defect; ship-nowhere-until-verified remains accurate; human verification is downstream press/judge stage by design. |
| C3 sibling-volume dependencies | resolved | Full bibliographic entries R85–R88 now resolve; key Nº2 methodology summarized inline per ch01/ch12; no re-derivation required. |
| C4 model names vs parameter classes / 31B vs 72B generation confound | rebutted-accepted | Class-level claim retained by design; v2 adds explicit generation confound note (2026 vs 2024 vintage) and downgrades bare size ordering to provisional; vendor tags remain in lab record for reproduction. |
