# LLMs for Machines — outline v1 (2026-08-29)

**Series:** O'AILLY Industrial Nº 3 · **Cover:** circuit moth, violet #A78BFA · **Tier:** comprehensive
**Status:** DRAFTING — sequencing hold lifted (Nº 1 and Nº 2 published 2026-08-28)

**Thesis:** A century of digitizing machines gave us protocols, historians, and SCADA —
not language. Language models can finally *read* that world, but only if they stay local
enough to reach it, constrained enough not to invent it, and honest enough to abstain
when the signal does not support a claim. This is the broad field guide. Industrial Nº 1
earned the plant-floor argument; Nº 2 earned the physics of local inference. Nº 3 widens
both into sensors, vehicles, robots, grids, buildings, and the quiet equipment majority.

**Evidence discipline:** every factual claim resolves to a citable public source or a lab
marker. Lab-specific numbers carry `[LAB: …]` and are labeled unmeasured when not
re-verified in this manuscript. Founder biography is never invented. Pico ≠ MCU. No
IEC 61508 certification claims. No vehicle/robot control claims.

**Audience:** engineers, integrators, and operators who put software next to machines in
any domain — and need a field guide, not a vendor deck.

## Part I — The meeting

1. **Machines Without Language** — the book-shaped hole; what Nº 1 and Nº 2 earned; audience; hard boundaries; moth as thesis.
2. **What a Language Model Is Allowed to Touch** — tokens/context/sampling for machine owners; authority frontier; schemas over chat.
3. **Sensors and the Signal Path** — phenomenon → bytes; units; calibration; missingness; why clean JSON lies.
4. **Protocols as Text** — Modbus, J1939, OPC UA, Sparkplug, syslog, Prometheus as languages; typed assertions over hex dumps.

## Part II — Domains

5. **Plants and Brownfield** — widening Nº 1; historians, air gaps, CMMS, politics of change.
6. **Vehicles and Mobile Equipment** — J1939 vocabulary; telematics; cab vs rack; diagnose ≠ drive.
7. **Robots and Workcells** — cells, pendants, WMS; language helps procedures, not motion plans.
8. **Grids, Energy, and Continuous Process** — slow dynamics; alarm floods; regional blast radius.
9. **Buildings, Facilities, and the Quiet Majority** — BACnet world; volume market is quiet failure.

## Part III — Practice

10. **Where the Model Runs** — MCU≠LLM through plant datacenter; residency; version pin; cost.
11. **Authority, Interlocks, and Abstention** — who may act; silence as feature; escalation packets.
12. **Measuring Models That Touch Machines** — OT-shaped evals; holdout; promotion packets.
13. **Failure Modes the Physical World Invents** — clock skew, sticky bits, power events, trust death.
14. **A Field Guide You Can Run** — printable checklist; never-claim list; what the series still refuses.

## Back matter

- Provenance page (models, sources, verifier, trail, C2PA)
- Glossary (≥40 terms across domains)
- References (every URL/ISBN/DOI resolves at Pass 1)
- Errata & retractions policy

## Open dependencies

1. Named-human verification pass before pipeline entry.
2. Practitioner-interview slots (optional depth); none invented.
3. Online Pass-1 citation fetch after chapters land.
4. Cross-links to published Nº 1 / Nº 2 keep claims inside what those volumes actually say.
