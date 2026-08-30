# Back Matter

## A note on the two kinds of citation in this book

This book cites two kinds of evidence, and keeps them visibly apart.

**Published sources** carry a bracketed `[R#]` marker that resolves in the References
section below. Every such entry names an authoritative source — a standard by its
number, a paper by author and venue (with a DOI or arXiv id), or a specification by its
canonical page — and resolves to a URL or a DOI wherever a stable one exists.

**Lab observations** carry a `[LAB: …]` marker. These are the authors' own reproducible
measurements taken on the RogerAI Labs bench (a Threadripper 9970X workstation, 128 GB
DDR5, Blackwell-class GPUs), logged in the lab's own `PROJECT-LOG` and `RESULTS-MATRIX`
records and reproducible on request. A `[LAB: …]` marker is **not** an external citation
and makes no claim of independent publication; it is a pointer into the authors' own
apparatus, named so a reader can ask for the run. Where the text gives a number without
either marker, the prose labels it unmeasured or refuses the claim. This mirrors the
convention established in Industrial Nº 1, *Local LLMs for Manufacturing*.

## Glossary

- **Abstention**: a model's deliberate refusal to answer when its inputs do not support a
  confident claim; the load-bearing behavior of a model placed next to equipment.
- **Actuator**: any device that converts a command into physical motion or state change; the
  side of the boundary a language model in this book never touches.
- **Air gap**: a physical or logical separation that keeps an operational-technology network
  off any external network.
- **Alarm flood**: a rate of alarms past what an operator can read and act on; the failure
  mode ISA-18.2 exists to prevent.
- **BACnet**: the building-automation protocol standardized as ASHRAE 135 / ISO 16484-5.
- **Blast radius**: the physical and organizational reach of a wrong output — the measure by
  which authority next to a machine is rationed.
- **Brownfield**: an installed base of equipment and software of mixed vintage that already
  runs and cannot be casually changed.
- **CAN bus**: the Controller Area Network data-link layer (ISO 11898) under J1939 and OBD-II.
- **CMMS**: computerized maintenance management system; the system of record for work orders.
- **Constrained decoding**: forcing a model's output to conform to a grammar or schema so it
  can only emit well-formed, typed results.
- **Contamination**: leakage of benchmark items into a model's training data, turning a high
  score into a measure of memorization rather than capability.
- **Historian**: an industrial time-series database storing years of tag values.
- **Holdout**: an evaluation set the model has not been tuned on, directly or by repeated reuse.
- **Interlock**: a deterministic safety mechanism that prevents an unsafe action regardless of
  what any higher-level system requests.
- **J1939**: the SAE family of heavy-duty vehicle networking standards layered on CAN.
- **MoE (mixture of experts)**: a model architecture that routes each token to a subset of
  expert sub-networks; its memory-read pattern dominates local-inference economics.
- **MQTT / Sparkplug**: a publish/subscribe transport (OASIS MQTT) and an industrial payload
  convention (Eclipse Sparkplug) common in telemetry.
- **Modbus**: a register-oriented industrial protocol; a multi-register value is read as a
  sequence, not atomically.
- **OPC UA**: an industrial interoperability standard whose data model carries a value with a
  status code and timestamps rather than a bare number.
- **OT (operational technology)**: the hardware and software that monitors and controls
  physical processes, as distinct from enterprise IT.
- **Oracle check**: verifying a model's output against a deterministic ground truth the system
  already owns.
- **Provenance**: the recorded chain of who or what produced an artifact and how it was verified.
- **Purdue model**: a layered reference architecture for segmenting OT from IT networks.
- **Replay harness**: a test rig that feeds recorded historian data back through a model to
  measure it on real, dated events.
- **Rung**: a placement tier for where a model runs, from sensor-adjacent microcontroller up to
  a plant datacenter or the rare justified cloud path.
- **SCADA**: supervisory control and data acquisition; the operator-facing control layer.
- **Schema**: a declared output shape (types, enums, required fields) that pins what a model may
  emit next to a machine.
- **SIL (safety integrity level)**: a quantified reliability target assigned to a safety
  function under IEC 61508 and its sector standards.
- **SPN/FMI**: the Suspect Parameter Number and Failure Mode Identifier vocabulary of J1939
  diagnostics.
- **Speculative decoding**: drafting several tokens with a small model and verifying them with
  the large one; its payoff on a spilled MoE collapses to a draft length of one.
- **Spill**: the condition where model weights do not fit in fast memory and must be read from
  slower DDR5, making inference bandwidth-bound.
- **Tag**: a named process variable in a historian or control system.
- **TEDS**: a transducer's self-describing electronic datasheet (IEEE 1451.0) carrying units,
  range, and calibration.
- **Telematics**: the off-board reporting of a mobile machine's operating data.

## References

- **[R1]** The first recorded computer "bug" — a moth logged in the Harvard Mark II relay by Grace Hopper's team, 1947 (cultural anchor: machines produce conditions, humans produce the language). https://en.wikipedia.org/wiki/Software_bug
- **[R2]** Situnayake, D. & Plunkett, J., *AI at the Edge*, O'Reilly Media, 2023. ISBN 9781098120191. https://www.oreilly.com/library/view/ai-at-the/9781098120191/
- **[R3]** C2PA Specifications 2.2, Coalition for Content Provenance and Authenticity. https://spec.c2pa.org/specifications/specifications/2.2/index.html
- **[R4]** NIST, AI Risk Management Framework (AI RMF 1.0), NIST AI 100-1. https://www.nist.gov/itl/ai-risk-management-framework (document: https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf)
- **[R5]** IEC 61508, Functional safety of electrical/electronic/programmable electronic safety-related systems. https://en.wikipedia.org/wiki/IEC_61508
- **[R6]** JSON Schema specification (draft 2020-12). https://json-schema.org/specification
- **[R7]** llama.cpp GBNF grammars — grammar-constrained generation and JSON-Schema-to-grammar. https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md
- **[R8]** OPC UA (OPC 10000 series), OPC Foundation online reference — the Variable/DataValue model carrying value, StatusCode, and timestamps. https://reference.opcfoundation.org/
- **[R9]** IEEE Std 1451.0, Smart Transducer Interface / TEDS. https://standards.ieee.org/ieee/1451.0/4162/
- **[R10]** JCGM 100:2008, Guide to the Expression of Uncertainty in Measurement (GUM), BIPM. https://www.bipm.org/documents/20126/2071204/JCGM_100_2008_E.pdf
- **[R11]** JCGM 200:2012, International Vocabulary of Metrology (VIM), BIPM. https://www.bipm.org/documents/20126/2071204/JCGM_200_2012.pdf
- **[R12]** ANSI/ISA-5.1, Instrumentation Symbols and Identification (tag conventions in P&IDs). https://en.wikipedia.org/wiki/Piping_and_instrumentation_diagram
- **[R13]** IEEE Std 754-2019, Floating-Point Arithmetic (float layout; non-associativity of batched sums). https://ieeexplore.ieee.org/document/8766229
- **[R14]** MODBUS Application Protocol Specification (function codes, PDU/MBAP framing, exception responses; multi-register reads are sequential). https://www.modbus.org/modbus-specifications
- **[R15]** SAE J1939, Serial Control and Communications Heavy Duty Vehicle Network — Top Level Document. https://www.sae.org/standards/content/j1939_201808/
- **[R16]** ISO 11898-1, Road vehicles — CAN — data-link layer and physical signalling. https://en.wikipedia.org/wiki/CAN_bus
- **[R17]** Eclipse Sparkplug 3.0.0 Specification (topic namespace, typed metrics, NBIRTH/NDATA/NDEATH lifecycle). https://sparkplug.eclipse.org/specification/version/3.0/documents/sparkplug-specification-3.0.0.pdf
- **[R18]** OASIS MQTT Version 5.0 (PUBLISH, topics, QoS, retained messages). https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- **[R19]** Prometheus exposition formats (HELP/TYPE lines; name{labels} value timestamp samples). https://prometheus.io/docs/instrumenting/exposition_formats/
- **[R20]** OpenMetrics specification (standardized successor to the Prometheus exposition format). https://openmetrics.io/
- **[R21]** RFC 5424, The Syslog Protocol (PRI = facility×8 + severity; structured data). https://datatracker.ietf.org/doc/html/rfc5424
- **[R22]** Kadavath et al., "Language Models (Mostly) Know What They Know," 2022. https://arxiv.org/abs/2207.05221
- **[R23]** SAE J1939-73, Application Layer — Diagnostics (active and previously-active fault reporting). https://www.sae.org/standards/content/j1939/73_201310/
- **[R24]** SAE J1939-71, Vehicle Application Layer (the standard SPN vocabulary). https://www.sae.org/standards/content/j1939/71_202008/
- **[R25]** SAE J1939-21, Data Link Layer (PGNs; transport of multi-frame messages). https://www.sae.org/standards/content/j1939/21_201810/
- **[R26]** SAE J3016, Taxonomy and Definitions for Terms Related to Driving Automation Systems. https://www.sae.org/standards/content/j3016_202104/
- **[R27]** Regulation of automated vehicles — the regulatory posture built on the SAE J3016 automation-level taxonomy. https://en.wikipedia.org/wiki/Regulation_of_self-driving_cars
- **[R28]** ISO 26262, Road vehicles — Functional safety. https://en.wikipedia.org/wiki/ISO_26262
- **[R29]** ISO 15143-3, Earth-moving machinery and mobile road construction machinery — Worksite data exchange — Telematics data (AEMP standard; vendor-neutral telematics).
- **[R30]** NIST SP 800-82 Rev. 3, Guide to Operational Technology (OT) Security (network segmentation; bound an untrusted component's misbehavior). https://csrc.nist.gov/pubs/sp/800/82/r3/final
- **[R31]** ANSI/ISA-95 / IEC 62264, Enterprise-Control System Integration (the layered operations model). https://en.wikipedia.org/wiki/ANSI/ISA-95
- **[R32]** ANSI/ISA-18.2 / IEC 62682, Management of Alarm Systems for the Process Industries. https://www.isa.org/standards-and-publications/isa-standards/isa-standards-committees/isa18
- **[R33]** EEMUA 191, Alarm Systems — A Guide to Design, Management and Procurement (the alarm-management guide behind ISA-18.2 practice).
- **[R34]** ISO 10218-1, Robots and robotic devices — Safety requirements for industrial robots — Part 1: Robots. https://en.wikipedia.org/wiki/ISO_10218
- **[R35]** ISO 10218-2, Robots and robotic devices — Safety requirements — Part 2: Robot systems and integration. https://en.wikipedia.org/wiki/ISO_10218
- **[R36]** ISO/TS 15066, Robots and robotic devices — Collaborative robots (shared-space safety).
- **[R37]** ANSI/RIA R15.06, Industrial Robots and Robot Systems — Safety Requirements (US adoption of ISO 10218).
- **[R38]** IEC 61131-3, Programmable controllers — Part 3: Programming languages. https://en.wikipedia.org/wiki/IEC_61131-3
- **[R39]** IEC 61499, Function blocks for industrial-process measurement and control systems. https://en.wikipedia.org/wiki/IEC_61499
- **[R40]** IEC 61850, Communication networks and systems for power utility automation. https://en.wikipedia.org/wiki/IEC_61850
- **[R41]** IEEE Std 1815 (DNP3), Distributed Network Protocol for SCADA. https://en.wikipedia.org/wiki/DNP3
- **[R42]** IEC/IEEE 60255-118-1, synchrophasor measurements for power systems (phasor measurement units). https://en.wikipedia.org/wiki/Phasor_measurement_unit
- **[R43]** NERC Critical Infrastructure Protection (CIP) standards, North American Electric Reliability Corporation (bulk-electric-system security).
- **[R44]** U.S.–Canada Power System Outage Task Force, Final Report on the August 14, 2003 Blackout (alarm-system failure and cascade). https://www.energy.gov/oe/articles/blackout-2003-final-report-august-14-2003-blackout-united-states-and-canada-causes-and
- **[R45]** IEC 62061, Safety of machinery — Functional safety of safety-related control systems. https://en.wikipedia.org/wiki/IEC_62061
- **[R46]** OSHA 29 CFR 1910.147, The Control of Hazardous Energy (Lockout/Tagout). https://www.osha.gov/laws-regs/regulations/standardnumber/1910/1910.147
- **[R47]** ISO 22400, Automation systems and integration — Key performance indicators (KPIs) for manufacturing operations management.
- **[R48]** ISA/IEC 62443 (ISA-99), Security for Industrial Automation and Control Systems (OT segmentation / Purdue layering). https://en.wikipedia.org/wiki/IEC_62443
- **[R49]** ASHRAE Standard 135 / ISO 16484-5 (BACnet) — object model, Reliability and Out_Of_Service properties, Units enumeration, priority array. https://www.ashrae.org/technical-resources/bookstore/bacnet
- **[R50]** ASHRAE Guideline 36, High-Performance Sequences of Operation for HVAC Systems.
- **[R51]** Project Haystack — semantic tagging for building-equipment data. https://project-haystack.org/
- **[R52]** Balaji et al., "Brick: Towards a Unified Metadata Schema For Buildings," ACM BuildSys 2016, DOI 10.1145/2993422.2993577. https://brickschema.org/
- **[R53]** KNX / ISO/IEC 14543-3, building-control networking standard. https://www.knx.org/
- **[R54]** U.S. Energy Information Administration, Commercial Buildings Energy Consumption Survey (CBECS). https://www.eia.gov/consumption/commercial/
- **[R55]** NAMUR NE 107, "Self-Monitoring and Diagnosis of Field Devices." https://en.wikipedia.org/wiki/NAMUR
- **[R56]** Katipamula, S. & Brambley, M., "Methods for Fault Detection, Diagnostics, and Prognostics for Building Systems — A Review," HVAC&R Research 11(1), 2005, DOI 10.1080/10789669.2005.10391123.
- **[R57]** llama.cpp — inference engine. https://github.com/ggml-org/llama.cpp
- **[R58]** vLLM — high-throughput inference and serving engine. https://github.com/vllm-project/vllm
- **[R59]** GGUF model file format specification. https://github.com/ggml-org/ggml/blob/master/docs/gguf.md
- **[R60]** Raspberry Pi 5 (gateway-class hardware; memory-bandwidth class). https://en.wikipedia.org/wiki/Raspberry_Pi
- **[R61]** NVIDIA Jetson embedded modules (edge-accelerator / gateway-class option). https://developer.nvidia.com/embedded/jetson-modules
- **[R62]** Williams, Waterman, Patterson, "Roofline: an insightful visual performance model for multicore architectures," Communications of the ACM 52(4), 2009, DOI 10.1145/1498765.1498785.
- **[R63]** Leviathan, Kalman, Matias, "Fast Inference from Transformers via Speculative Decoding," ICML 2023. https://arxiv.org/abs/2211.17192
- **[R64]** Chen et al., "Accelerating Large Language Model Decoding with Speculative Sampling," 2023. https://arxiv.org/abs/2302.01318
- **[R65]** ISO 13849-1, Safety of machinery — Safety-related parts of control systems. https://en.wikipedia.org/wiki/ISO_13849
- **[R66]** IEC 61511 / ANSI-ISA-84, Functional safety — Safety instrumented systems for the process industry sector. https://www.isa.org/standards-and-publications/isa-standards/isa-standards-committees/isa84
- **[R67]** Chow, C.K., "On Optimum Recognition Error and Reject Tradeoff," IEEE Transactions on Information Theory 16(1), 1970, DOI 10.1109/TIT.1970.1054406.
- **[R68]** Geifman & El-Yaniv, "Selective Classification for Deep Neural Networks," NeurIPS 2017. https://arxiv.org/abs/1705.08500
- **[R69]** Guo, Pleiss, Sun, Weinberger, "On Calibration of Modern Neural Networks," ICML 2017. https://arxiv.org/abs/1706.04599
- **[R70]** Hendrycks & Gimpel, "A Baseline for Detecting Misclassified and Out-of-Distribution Examples in Neural Networks," ICLR 2017. https://arxiv.org/abs/1610.02136
- **[R71]** Parasuraman & Riley, "Humans and Automation: Use, Misuse, Disuse, Abuse," Human Factors 39(2), 1997, DOI 10.1518/001872097778543886.
- **[R72]** Bainbridge, L., "Ironies of Automation," Automatica 19(6), 1983, DOI 10.1016/0005-1098(83)90046-8.
- **[R73]** OSHA 29 CFR 1910.119, Process Safety Management of Highly Hazardous Chemicals (Management of Change). https://www.osha.gov/laws-regs/regulations/standardnumber/1910/1910.119
- **[R74]** Thinking Machines Lab, "Defeating Nondeterminism in LLM Inference," 2025 (batch-dependence of temperature-zero output). https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/
- **[R75]** Efron, B., "Bootstrap Methods: Another Look at the Jackknife," Annals of Statistics 7(1), 1979, DOI 10.1214/aos/1176344552. https://projecteuclid.org/journals/annals-of-statistics/volume-7/issue-1/Bootstrap-Methods-Another-Look-at-the-Jackknife/10.1214/aos/1176344552.full
- **[R76]** Sainz et al., "NLP Evaluation in Trouble: On the Need to Measure LLM Data Contamination for each Benchmark," EMNLP Findings 2023. https://arxiv.org/abs/2310.18018
- **[R77]** Dwork, Feldman, Hardt, Pitassi, Reingold, Roth, "The reusable holdout: Preserving validity in adaptive data analysis," Science 349, 2015, DOI 10.1126/science.aaa9375.
- **[R78]** EleutherAI, lm-evaluation-harness (a vetted, widely used evaluation harness). https://github.com/EleutherAI/lm-evaluation-harness
- **[R79]** Hendrycks et al., "Measuring Massive Multitask Language Understanding" (MMLU), ICLR 2021. https://arxiv.org/abs/2009.03300
- **[R80]** Golchin & Surdeanu, "Time Travel in LLMs: Tracing Data Contamination in Large Language Models," 2023. https://arxiv.org/abs/2308.08493
- **[R81]** RFC 5905, Network Time Protocol Version 4 (NTP). https://datatracker.ietf.org/doc/html/rfc5905
- **[R82]** IEEE Std 1588, Precision Time Protocol (PTP) for networked measurement and control. https://standards.ieee.org/ieee/1588/6825/
- **[R83]** Perrow, C., *Normal Accidents: Living with High-Risk Technologies*, 1984 (interactive complexity and tight coupling). https://en.wikipedia.org/wiki/Normal_Accidents
- **[R84]** Loss of the Mars Climate Orbiter to a units mismatch (a value meaning one thing to the producer and another to the reader). https://en.wikipedia.org/wiki/Mars_Climate_Orbiter
