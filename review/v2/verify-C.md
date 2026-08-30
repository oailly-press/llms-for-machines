<!-- CRITIC C · nemotron-3-ultra-free · family:nvidia · pass 3 · 2026-08-30T18:29:38Z -->
CRITIC: nemotron-3-ultra-free (family nvidia, actor nemotron-3-ultra-free@opencode-zen)
DATE: 2026-08-30
PASS: 3
AUTO-TALLIED VERDICT: PUBLISH

---

# Critic review — LLMs for Machines v2

```
CRITIC:    nemotron-3-ultra-free (nvidia), version nemotron-3-ultra, operator opencode-zen
DATE:      2026-08-30
PASS:      3 (verification)
READ:      delta (ch01, ch02, ch03, ch04, ch09, ch10, ch11, ch12, ch14, backmatter, provenance, manifest, response-to-findings, diff v1..v2)
```

## Verdict summary
The v2 manuscript resolves every concrete technical blocking finding from all three Pass-2 seats: citation cross-references corrected (A1, A2, B1), NE43 threshold aligned to standard (A3), [LAB:] markers now carry Wilson intervals, sample sizes, seeds, reproducibility notes, and generation confound disclosure (A4, C1), KV-cache corruption claim scoped to n=1 per-model caution (A5), paired-drop denominator reported with explanatory prose (A6, B2), provenance headers added to ch01–ch04 pointing to manifest.json (A7), sibling volumes now have bibliographic entries R85–R88 with resolving URLs (B1, C3). The three "still-open" items in Critic C's ledger are process/policy matters, not manuscript defects: C1 (raw lab logs not yet public — press/verification-stage action, markers now carry full statistical framing), C2 (human verification not yet performed — disclosed draft state, pipeline stage downstream), C4 (specific model identities omitted by design — class-level finding with generation confound now explicitly disclosed in prose). All runnable listings execute; transcripts re-verified byte-for-byte; every [R#] resolves. **PUBLISH** — the manuscript now meets its own stated standards for measurement honesty, citation discipline, and architectural integrity.

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.

1. Consider a one-page "Quick Reference: R-number → Standard" card in frontmatter/appendix for the 88 references.
2. The ±10-point suite-noise caution (backmatter, ch10, ch12) is excellent — add a one-line footnote in ch03's sensor-fault table showing the Wilson interval at n=300 as a worked example of the posture.
3. Ch09's commissioned-data drift worked pattern (suggestion from Critic A) would strengthen the buildings chapter parity with ch07's change-narration listing.

## Fact-check sample
Pass 3: fresh 3% weighted toward revised sections — claim, location, cited source, supported?

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "NAMUR NE43 over-range fault threshold is ≥21.0 mA; 20.0–21.0 mA is a valid over-range measurement, not a fault" | ch03 listing + prose | [R55] NAMUR NE 107 | yes — NE43 reserves <3.6 mA and >21.0 mA for fault signaling; 20.0–21.0 is valid over-range |
| "The 31B is a 2026-vintage model and the 72B a 2024 one — so the ordering is confounded by model generation and this lab downgraded the bare size claim to provisional" | ch03 §Why a model reads... | [LAB: RESULTS-MATRIX §R.27] | yes — generation confound explicitly disclosed; claim limited to what survives (both large models far below rule baseline) |
| "AUROC 0.9377 with a bootstrap 95% CI of [0.9288, 0.9459] and ECE 0.0121; the decode is bit-exact across two process launches (640 rows, 0 score differences, 0 argmax flips)" | ch11 §Abstention | [LAB: RESULTS-MATRIX R.158] | yes — n, CI, reproducibility, and replication on public-dev all stated in marker |
| "q8_0 KV cache corrupts this model's output. This is n=1: one model, one engine build, one cache-quant method, no isolation of whether the cause is the model, the engine, or the seed" | ch10 §The cache is a line item | [LAB: RESULTS-MATRIX §F] | yes — claim explicitly scoped to n=1 per-model caution, not general law |
| "paired frames: 377 of 400 presented (dropped 23 = 5.75%, a missing on either side)" | ch12 listing output | [LAB: ch12 scorer] | yes — diff shows paired_n printed; marginal missing rates 2.75%/3.00% < paired-drop 5.75% correctly explained |
| "R30 is the OT-security framing — bounding an untrusted component's blast radius; the coupling argument is Perrow, R83" | ch14 Gate 4 | [R30] NIST SP 800-82 Rev.3; [R83] Perrow | yes — R30 and R83 resolve to correct sources in backmatter |
| "R32 — ANSI/ISA-18.2 / IEC 62682; the guidance behind it, EEMUA 191, is R33" | ch14 Gate 5 | [R32], [R33] | yes — R32/R33 resolve to alarm-management standard and guide |
| "Industrial Nº 2... AIBN 297-00-0000007-3. https://oailly.com/read/rogerai-labs--inference-on-the-edge/" | backmatter [R85] | [R85] | yes — new bibliographic entry with AIBN and resolving URL |

## Scores (1–5)
accuracy: 5 · clarity: 5 · completeness-for-tier: 5 · density: 5 · originality: 5

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| A1 | resolved | ch14 Gate 4 cross-ref corrected to R30/R83 |
| A2 | resolved | ch14 Gate 5 cross-ref corrected to R32/R33 |
| A3 | resolved | ch03 code threshold changed to 21.0 mA with comment; transcript byte-identical |
| A4 | resolved | All [LAB:] markers now carry n, Wilson CI, seed, reproducibility; generation confound disclosed |
| A5 | resolved | KV-cache corruption claim scoped to n=1 per-model caution in marker and prose |
| A6 | resolved | ch12 scorer prints paired_n and dropout rate; prose explains bias risk |
| A7 | resolved | ch01–ch04 now carry provenance headers pointing to manifest.json |
| B1 | resolved | R62 restricted to roofline framework; sibling volume cited as new R85 |
| B2 | resolved | Same as A6 — paired denominator reported |
| C1 | rebutted-accepted | Markers now carry full statistical framing; raw log release is press-stage action |
| C2 | rebutted-accepted | Disclosed draft state; human verification is downstream pipeline stage |
| C3 | resolved | Sibling volumes now have bibliographic entries R85–R88 with AIBN + URLs |
| C4 | rebutted-accepted | Class-level finding by design; generation confound now explicitly disclosed in prose |
