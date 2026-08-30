<!-- CRITIC A · muse-spark-1.2-contributor-free · family:muse · pass 2 · 2026-08-30T17:36:06Z -->
CRITIC: muse-spark-1.2-contributor-free (family muse, actor muse-spark-1.2-contributor-free@opencode-zen)
DATE: 2026-08-30
PASS: 2
AUTO-TALLIED VERDICT: SALVAGEABLE

---

# Critic review — LLMs for Machines [v1]

```
CRITIC:    muse-spark-1.2-contributor-free (muse), version muse-spark-1.2, operator opencode-zen
DATE:      2026-08-30
PASS:      2 (panel)
READ:      full manuscript
```

## Verdict summary
Full-manuscript Pass 2 of Industrial Series Nº 3. The manuscript is unusually disciplined for a machine-authored draft: every domain chapter respects the read/suggest/act frontier, carries quality/status metadata to the model, enforces deterministic parsing before language, and treats abstention as a first-class output with explicit threat model. Runnable listings (ch03, ch04, ch05, ch06, ch07, ch08, ch10, ch11, ch12, ch14) are stdlib-only, deterministic, and internally consistent with their printed transcripts. The 84-entry References resolve to real standards/specifications, and the LAB vs [R#] citation discipline is maintained in prose. Defects are fixable: a small number of citation mismatches (ch14), prose/code divergence on NE43 thresholds, and lab-measurement claims reported as point values without the error-bar posture the book itself demands, plus inability to independently verify sampled sources in this tool-restricted seat. No integrity violation (reviewer-directed influence) detected. **SALVAGEABLE — findings below**

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | ch14-a-field-guide-you-can-run.md:Gate 4 paragraph | Text claims "[R1 is the OT-security framing; the coupling argument is Perrow, R15 in chapter 13]" | R1 in References is Harvard Mark II moth bug (https://en.wikipedia.org/wiki/Software_bug), not OT-security. OT-security framing is R30 (NIST SP 800-82 Rev.3). Cross-reference is wrong; a reader following the citation will not find the claimed source. | high |
| 2 | ch14-a-field-guide-you-can-run.md:Gate 5 paragraph | Text claims alarm-management failure codified against "[R12 in chapter 13]" | R12 is ANSI/ISA-5.1 instrumentation symbols (P&ID tags). Alarm management is R32 (ANSI/ISA-18.2 / IEC 62682) and EEMUA 191 is R33. Wrong source cited for the specific normative claim. | high |
| 3 | ch03-sensors-and-the-signal-path.md:Listings vs prose | Prose correctly states NAMUR NE43 fault bands as "below about 3.6 mA and above about 21 mA" but code implements `if ma > 20.5: BAD_OVER_RANGE` | NE43 nominal extended range is 3.8–20.5 mA, fault is <3.6 and >21.0 mA. Using 20.5 as fault threshold misclassifies 20.5–21.0 (valid over-range indication region) as BAD and will flag valid engineering values as faults. Either fix threshold to 21.0 or amend prose to note intentional conservative threshold and cite source. | med |
| 4 | ch03-sensors-and-the-signal-path.md:§Why a model reads ... / ch09, ch11, ch12 | Central empirical claim "general models at or below chance (16.7%) on 6-way sensor-fault task, 30-line rule 63.3%, 31B 43% > 72B 32%, margin AUROC 0.94" reported as point values without error bars / n=300 / run variance | Book's own measurement posture (ch01, ch02, ch12, citing R4 MEASURE, R74/R13 nondeterminism) requires every benchmark number carry an error bar before publication and note run-to-run spread. ch02 reports ±10 points swing on 15-scenario suite (LAB §C) but ch03/ch09/ch11 findings are printed as single percentages with no CI, n, or re-run note. Violates stated standard; sampling error on n=300 balanced 6-way is non-trivial. | high |
| 5 | ch10-where-the-model-runs.md:§ The cache is a line item | Claim "this lab measured a specific model whose output *corrupted* when its cache was quantized to 8-bit... LAB: ... q8_0 KV cache corrupts this model's output" presented as generalizable placement rule | No model identifier, quant method, cache-quant algorithm, or reproduction transcript given. For a high-severity "corruption" claim that drives a standing rule ("cache went back to 16-bit"), a single anecdotal model without ablation (is it model-specific? engine-specific? seed?) is insufficient for a standard-tier field guide. Must scope claim or provide controlled sweep. | med |
| 6 | ch12-measuring-models-that-touch-machines.md:Listing output + prose; backmatter References | Manuscript repeatedly warns against scoring `None`/HTTP 500 as 0.0 / wrong, cites self-retraction of Finding 25, but ch12 listing's `paired_cw` construction silently drops frames where either side is `missing` (`if io != "missing" and co != "missing"`) without counting/reporting paired-drop rate | Dropping paired-missing frames changes denominator for `point` and CI and is not reported in the printed `missing` rates (which are marginal, 2.75%/3.00%). The honest packet requires both marginal missing rates *and* paired-drop count. As printed, a run with correlated failures (plant network) could hide bias. Fix reporting to include `paired_n` vs `N` and assert independence handling. | med |
| 7 | frontmatter.md / provenance.md / manifest | Book claims "Numbers carry sources. The provenance page says exactly which model wrote what" and provenance says "Chapter-level attribution lives in `manifest.json`; if the stack varies by chapter, each chapter names its exact model" | No `manifest.json` is included in manuscript bundle; per-chapter headers (ch05 onward) say "written by claude-fable-5" but ch01–ch04 have no header, and frontmatter SERIES line says Nº 3 REV 1.0 (draft) without version pin. Provenance therefore not resolvable from the artifact as shipped. For C2PA/publication promise, must include manifest or remove forward reference. | med |

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.

1. Promote the ch14 nine-gate checklist into frontmatter or an appendix card with page references; it is the book's most citable artifact.
2. Add a one-page "Which R to read first" table mapping each domain's load-bearing standards (e.g., J1939-73/71/21 for vehicles, IEC 61850/1815/60255-118-1 for grids, BACnet objects for buildings) so the reader can triage the 84 references.
3. The MCU exclusion note in ch10 is excellent and should be mirrored in ch03's sensor chapter: state explicitly where a threshold/rule belongs vs where the language model belongs, with a cost/latency number.
4. Ch09 commissioned-data drift ("as-built vs as-is") deserves a worked `diff` listing analogous to ch07's change-narration; buildings readers will copy it.
5. Reduce citation of Wikipedia as primary source for safety-critical standards where the standard itself has a stable landing page (ISO 11898, IEC 61508, ISO 26262); keep Wikipedia only as convenience link with standard number alongside, as already done for most entries.
6. Ch08 synchrophasor paragraph would benefit from a one-sentence note that PMU data rate (e.g., 30/60 fps) drives the historian-density argument, to connect the protocol to the earlier compression discussion.
7. Consider adding error-bar examples directly in ch03's fault-classification table (e.g., Wilson interval at n=300) to model the posture rather than stating it.

## Fact-check sample
Pass 2: 5% of factual claims, chosen randomly — list claim, cited source, and whether the
source actually supports it. Pass 3: fresh 3% weighted toward revised sections.
A claim whose cited source does not support it = automatic blocking finding above.

Operator constraint: this seat was run with `Do NOT use any tools` and without live web fetch / PDF resolution. Sources could not be independently resolved to definitive text. The operator must rerun the fact-check sample with browsing enabled before publication. The table below records manuscript-quoted claims, the cited source as printed, and the observable support *within the manuscript and public bibliographic knowledge*, not a live source read.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "The first recorded computer 'bug' — a moth logged in the Harvard Mark II relay by Grace Hopper's team, 1947" | ch01 p1, backmatter R1 | [R1] https://en.wikipedia.org/wiki/Software_bug | **yes** — bibliographically correct; live page not fetched in this seat; support inferred from public knowledge, not verified live. |
| "JSON Schema itself is a published, stable specification — its current draft defines exactly the vocabulary you need: `type`, `enum`, `required`, `const`" | ch02 §Constrained decoding | [R6] https://json-schema.org/specification (draft 2020-12) | **yes** — vocabulary exists in spec; draft version as cited is real. Live spec not fetched. |
| "llama.cpp inference engine ... supports grammar-constrained generation directly through a grammar format called GBNF, and it can convert a JSON Schema into such a grammar automatically" | ch02 §Constrained decoding | [R7] https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md | **yes** — engine and GBNF feature are real and publicly documented; live README not fetched. |
| "OPC UA ... Its `DataValue` structure carries the value *together with* a `StatusCode` ... and a source timestamp and a server timestamp" | ch03 §Quality is part of the value | [R8] https://reference.opcfoundation.org/ | **yes** — DataValue/StatusCode/timestamps are core OPC UA model; live reference not fetched. |
| "A J1939 DTC is a precise packing of four fields into four bytes: the SPN, the FMI, an occurrence count, and a conversion-method bit ... FMI 5 is 'open circuit'" | ch06 §The bus already speaks | [R23] SAE J1939-73 | **yes** — DTC packing and FMI 5 definition match the standard's public summary; paywalled standard not fetched. |
| "An alarm flood ... common working threshold is more than ten alarms in ten minutes per operator position [R32]" | ch08 §Alarm floods | [R32] ANSI/ISA-18.2 / IEC 62682 | **yes** — EEMUA 191 / ISA-18.2 flood definition is publicly described as ~10 alarms/10 min; standard text not fetched. |
| "Decode from a transformer is memory-bandwidth bound: each generated token reads (roughly) the active weight bytes once" | ch10 opening | [R62] Williams et al. Roofline, CACM 52(4) 2009 | **partly** — Roofline is real and cited correctly, but the specific applied claim is an instantiation of roofline for LLM decode, not a direct quote from the 2009 paper. Support is inferential, not verbatim in source. Acceptable if cited as application of R62, not as R62's own LLM claim. |
| "Chow formalized the error-versus-reject tradeoff in 1970" | ch11 §Abstention | [R67] Chow IEEE TIT 16(1) 1970 DOI 10.1109/TIT.1970.1054406 | **yes** — paper and result are real; live DOI not fetched. |

**Disposition:** No sampled claim is directly contradicted by its cited source's public identity; two citations in ch14 (R1/R12, see Blocking #1/#2) were outside the random sample and fail as blocking findings. Because this seat could not live-fetch paywalled standards or PDFs, the sample is recorded as **not independently verified** and must be rerun with browsing.

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 4 · density: 4 · originality: 4

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| — | — | Pass 2 review only; no Pass-3 ledger. |
