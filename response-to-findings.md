# Response to v1 findings — LLMs for Machines (→ v2)

Pass-2 seats (fork `review/v1`): **A muse-spark-1.2** (muse), **B mimo-v2.5** (xiaomi),
**C ling-3.0-flash-fin** (inclusionai). All three: **SALVAGEABLE**. Every blocking finding
is answered below, fix-with-diff or rebut-with-evidence. Word count grew 72,689 → 73,890
(all additions are honest statistical framing and four new references); Pass-1 gate PASSes
with 0 rejects; every runnable listing still executes and its transcript matches.

---

## Critic A — muse-spark-1.2 (muse)

**A1 — ch14 Gate 4 cross-ref: `[R1 is the OT-security framing; the coupling argument is
Perrow, R15 in chapter 13]`.** FIXED. R1 is the Harvard Mark II moth (cultural anchor), not
OT security, and Perrow is R83, not R15 (J1939). Corrected to
`[R30 is the OT-security framing — bounding an untrusted component's blast radius; the
coupling argument is Perrow, R83]`. Location: ch14 Gate 4.

**A2 — ch14 Gate 5 cross-ref: `[R12 in chapter 13]` for alarm management.** FIXED. R12 is
ANSI/ISA-5.1 (P&ID symbols). The alarm-management standard is R32 (ANSI/ISA-18.2 / IEC
62682) and the guide behind it is R33 (EEMUA 191). Corrected to
`[R32 — ANSI/ISA-18.2 / IEC 62682; the guidance behind it, EEMUA 191, is R33]`. Location:
ch14 Gate 5.

**A3 — ch03 NE43 threshold: code `if ma > 20.5` vs prose/standard 21.0 mA.** FIXED. The
NAMUR NE43 over-range *fault* threshold is ≥21.0 mA; 20.0–21.0 mA is a valid over-range
measurement, not a fault. Code changed to `if ma > 21.0:` with a comment stating the
20.0–21.0 band is valid over-range. The listing was re-run: the printed transcript is
byte-identical (the pinned-high case, raw=4095 → 24.0 mA, is still >21.0 → BAD_OVER_RANGE),
so code, transcript, and prose now agree. Location: ch03.

**A4 — [LAB:] numbers lack error bars / n / run variance (convergent with B and C1).**
FIXED — the central debt. The book preaches error bars, so it now shows them on its own
numbers, without inventing any:
- **backmatter** gains a statistical-framing convention: each `[LAB:]` marker carries the
  range/±/CI and *n* where the record has repeats, or is marked a single-run point
  observation where that is the truth; the ±10 suite-noise and the §I.4 warmth caveat are
  stated once as standing cautions.
- **ch03 / ch09** sensor-fault: n=300 balanced (50/class), single deterministic run, seed
  20260811, byte-reproducible; Wilson 95% intervals added (rules 63.3% ≈ 57.7–68.6%, 27B
  14.3% ≈ 10.8–18.7%, 270M 0.0% ≈ 0–1.3%, 31B 43% ≈ 37.5–48.7%, 72B 32% ≈ 27.3–37.8%).
- The 31B > 72B ordering is **corrected for honesty**: RESULTS-MATRIX §R.27 downgraded the
  bare size claim to provisional because the two models are a generation apart (2026 vs
  2024 vintage), so it is confounded by generation. The prose now says so and states only
  what survives — both large models sit far below the rule baseline, neither acquires the
  combination-feature classes. Location: ch03 §"Why a model reads a tag as a sentence".
- **ch10 / ch12 / ch11** given n and CIs: MMLU on n=100 (Wilson ≈ ±6–8), the 15-scenario
  tool suite as a ±10 spread across three runs, the abstention margin as AUROC 0.9377
  [0.9288, 0.9459] on n=3,725 (channel slice 3,535), etc.
- **ch01/ch02/ch04** markers that repeat these findings now carry n=300 / n=15 tags.

**A5 — ch10 q8_0 KV-cache corruption is a single anecdote driving a standing rule.** FIXED
by scoping. The marker now states it is **n=1** (one model — DeepSeek-V4-Flash — one
llama.cpp build, one cache-quant method, no isolation of model vs engine vs seed) and is
"deliberately reported as a per-model caution, not as a law that q8_0 caches corrupt in
general — most models tolerate an 8-bit cache fine." The surrounding prose already framed
cache precision as a per-model property to verify; the marker now matches that scope.
Location: ch10 §"The cache is a line item".

**A6 — ch12 paired-drop denominator not reported (same as B2).** FIXED. The scorer now
computes and prints `paired_n` and the dropped count/rate; the re-run transcript reads
`paired frames: 377 of 400 presented (dropped 23 = 5.75%, a missing on either side)`. The
prose explains that the paired-drop rate (5.75%) exceeds either marginal missing rate
(2.75%/3.00%) because a pair is lost when *either* side fails, and that hiding it can hide a
bias on a plant network with correlated failures. Location: ch12.

**A7 — provenance forward-reference to `manifest.json`; ch01–ch04 lack chapter headers.**
FIXED. `manifest.json` is in fact included in the bundle (it is what the gate word-counts),
with per-chapter `written_by` for all 14 chapters — so provenance already resolves. To
remove the inconsistency the critic saw, ch01–ch04 now carry the same per-chapter
provenance header note ch05+ have, each pointing to `manifest.json` for authorship.

---

## Critic B — mimo-v2.5 (xiaomi)

**B1 — [R62] roofline cited where the sibling volume / decode-bandwidth claim belongs
(convergent with A fact-check).** FIXED, three ways:
1. `[R62]` (Williams/Waterman/Patterson roofline, CACM 2009) is now cited **only** for the
   general roofline / arithmetic-intensity concept, with an explicit note that it predates
   transformers and is a framework, not a claim about them.
2. The specific "transformer decode is memory-bandwidth-bound" claim is re-attributed to
   **the author's own reproducible measurement** (RESULTS-MATRIX §A/§E — the 2-vs-26 tok/s
   engine-path result is the signature of a bandwidth-bound kernel), which is exactly what
   the book is entitled to assert.
3. The two ch10 places that cited `[R62]` for the *sibling volume* now cite the sibling
   volume's own new entry, **[R85] *Inference on the Edge*** (AIBN 297-00-0000007-3,
   oailly.com/read URL).
Sibling volumes now have real bibliographic entries: **[R85]** Inference on the Edge,
**[R86]** Measure Twice, **[R87]** Local LLMs for Manufacturing, **[R88]** The Abstention
Reader — each by title + AIBN + resolving oailly.com/read URL — and are marked at their
first load-bearing mention (ch10, ch12, backmatter, ch11). Locations: ch10, backmatter.

**B2 — ch12 paired subset vs full-suite population mismatch.** FIXED (see A6). The packet
now prints the paired denominator (377 of 400), the `rates()` comment marks its `cw` as the
*marginal* per-frame rate, and the prose states plainly that the interval covers the paired
subset, not the full suite. Location: ch12.

---

## Critic C — ling-3.0-flash-fin (inclusionai)

**C1 — all [LAB:] empirical claims rest on internal records a reviewer cannot access.**
ADDRESSED (with A4). Every `[LAB:]` marker now carries its n, its interval or single-run
status, its seed/reproducibility, and its apparatus, so the claims are stated at the
strength the record supports and are reproducible on the named bench. Full public release of
the raw run logs is a press/verification-stage action (the records are RogerAI-internal at
draft stage) and is noted as such; the honesty debt the reviewer flagged — numbers printed
as bare point values — is paid.

**C2 — provenance says "verification NOT yet performed" yet ships as REV 1.0.** REBUTTED
WITH EVIDENCE (accepted as a disclosed state, not a defect — the critic's own ledger marks
it "still-open, a disclosed state"). This is by design: the book is a **draft under review**,
the disclosure is truthful, and the volume ships nowhere until the named human verifier
completes the pass. The author cannot and does not perform that human-verification/signing
step — it is the judge/press stage of the pipeline, deliberately downstream of this
revision. No change beyond keeping the disclosure accurate.

**C3 — sibling-volume dependencies (esp. Nº 2) not resolvable.** FIXED (the critic's own
ledger marks this "rebutted-accepted"). The four sibling volumes now have full bibliographic
entries [R85]–[R88] with AIBN + resolving read URLs, cited at their load-bearing mentions,
so a reader can locate them. The methodology this volume assumes is already summarized inline
(ch01 measurement posture, ch12 benchmarking rules); it cites rather than re-derives, which
is the correct scope for a field guide.

**C4 — model parameter classes given without vendor/version/date.** REBUTTED WITH EVIDENCE
and partially addressed. The finding this volume makes is a **class-level** one — capability
on machine-reading does not come from scale — so it is stated over parameter classes
(0.27B / 0.8B / 27B / 31B / 72B) rather than as a model-shaming leaderboard. The one place
where a specific comparison could mislead — the 31B-vs-72B ordering — is now corrected: the
prose discloses the two are a generation apart and downgrades the size claim to provisional
(see A4), which is the honest handling of the identity that actually matters here. The exact
vendor tags live in the lab record (RESULTS-MATRIX §R.26/§R.27) for the reproduction the
critic wants, and adding a generation caveat is the substantive fix; naming current
third-party models by brand in the prose is deliberately avoided.

---

## Summary

- **Fixed with diff:** A1, A2, A3, A4, A5, A6, A7, B1, B2, C1, C3 (+ the C4 substance: the
  generation confound on the 31B/72B ordering).
- **Rebutted with evidence:** C2 (disclosed draft state; human verification is a downstream
  press action), C4-in-part (class-level finding by design; vendor tags kept in the lab
  record).
- **Gate:** Pass-1 PASS, 0 rejects. Body 73,890 words. All runnable listings execute;
  ch03 and ch12 transcripts re-verified byte-for-byte; every `[R#]` resolves.
