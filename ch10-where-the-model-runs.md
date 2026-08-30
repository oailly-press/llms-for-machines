# Chapter 10 — Where the Model Runs

*(draft v1, 2026-08-30 — written by claude-fable-5, RogerAI Labs. Published sources carry `[R#]` markers resolved in the References. Numbers attributed to this lab are the author's declared reproducible observations, stated with the apparatus that produced them; the physics that explains them is developed at book length in the sibling volume, Industrial Nº 2, *Inference on the Edge*.)*

Every domain chapter in this book ended at the same unanswered question. The building model, the vehicle model, the plant model — each has to physically run somewhere, on some box, at some cost, with some latency, inside or outside some wall. This chapter is the geography of that decision. It is deliberately a companion to the sibling volume rather than a replacement: Industrial Nº 2 spends three hundred pages on the physics of local inference — the arithmetic of quantization, the mechanics of speculation, the memory-bandwidth wall — and I will cite it rather than re-derive it. What this chapter adds is the *placement* decision as an owner of physical machines faces it: a ladder of hardware rungs, what each one can and cannot run, and how cost, latency, data residency, and version discipline attach to the rungs rather than to the marketing.

The organizing fact, the one that makes the whole geography legible, is this: **decode from a language model is bound by memory bandwidth, not by arithmetic.** Generating each token requires reading (approximately) the model's active weights out of memory once. That single sentence explains almost every placement decision in this chapter — why a Raspberry Pi runs a 7B model at seconds per token, why a GPU runs it at over a hundred tokens per second, why spilling ten percent of a model to system RAM costs far more than ten percent of its speed, and why "just add more compute" is the wrong lever. We will keep returning to it. The sibling volume proves it properly `[R62]`; here it is a tool.

## The rungs, honestly labeled

There are more places to run a model than the two the industry advertises ("the cloud" and "the edge," both under-specified). Here is the ladder as it actually exists next to physical machines, from the smallest rung up, with the honest boundary of each.

**The MCU class — and why it is not on this ladder as an LLM host.** The smallest computer next to a machine is a microcontroller: a sensor node, a smart actuator, a protocol gateway with a few hundred kilobytes of RAM and no operating system worth the name. This class is where a great deal of real machine intelligence lives, and it is important to say clearly that **it does not run a language model.** It runs deterministic code, lookup tables, and at most a tiny quantized classifier — the thirty-line rule baseline from Chapter 9 lives here and thrives. This distinction is a standing discipline in this lab, worth stating because the market blurs it constantly: the smallest *language model* this lab builds, in the "Pico" class, is a few hundred million parameters, and it is emphatically an LLM that needs a real processor and hundreds of megabytes to run — it is not an MCU-class artifact and must never be marketed as one. When a task genuinely belongs on the MCU (a fast local safety check, a deterministic decode, a debounce), the right answer is code, not a language model shrunk until it fits a place it does not belong. The MCU rung is on this ladder to be excluded from it, on purpose.

**The gateway / single-board class.** A Raspberry Pi 5, an industrial ARM gateway, a small fanless box, or a low-power embedded accelerator such as an NVIDIA Jetson module `[R61]`: a real operating system, a few gigabytes of RAM, memory bandwidth in the tens of gigabytes per second `[R60]`. This class *can* run a language model — a sub-1B or small quantized model — and the bandwidth wall tells you exactly how well. At roughly 17 GB/s of memory bandwidth, a 7B model with about 4.5 GB of active 4-bit weights reads its weights about 17/4.5 ≈ 3.8 times per second in the ideal case, so a few tokens per second at best, before the real-world de-rating. That is genuinely usable for a short typed-fault assertion (Chapter 4's output shape) issued occasionally; it is unusable for a conversational assistant or anything that must stream a paragraph in front of a person. The gateway class is the honest home of the *small always-on reader* — the model that watches one machine, emits an occasional structured verdict, and escalates everything harder upstream.

**The industrial PC / CPU-only class.** A ruggedized x86 box with dual-channel DDR5, no GPU: memory bandwidth in the low tens to ~80–90 GB/s depending on the platform. This class runs small models comfortably and mid-size models slowly, and it hits the bandwidth wall as a hard ceiling — you cannot buy your way past it with more cores, because decode is not core-bound. It is the workhorse of the sidecar deployment (Chapter 8's shape one) where a GPU is unavailable, unwanted, or unjustified, and it is the rung where "which model fits" starts to require the quantization arithmetic in earnest.

**The on-site GPU box.** One accelerator or several, VRAM in the tens to low hundreds of gigabytes, memory bandwidth approaching or exceeding a terabyte per second. This is where mid-size and — with multiple accelerators — genuinely large models run at interactive speed, and it is the rung this lab lives on, so it is the rung this chapter can speak about from measurement rather than spec sheet. Everything below about spill, quantization, and speculation was measured here.

**The plant datacenter.** A rack, redundant power and cooling, multiple GPU boxes serving a whole facility or campus. Architecturally this is Chapter 8's hierarchy shape made physical — the department server scaled up — and it introduces genuine operations (scheduling, high availability, capacity planning) but no new *placement* physics. It is the on-site GPU rung with an operations team.

**The cloud path, and when it is actually justified.** The rung the industry defaults to and this book defaults against — not out of dogma but because the premise of the whole series is that the local ceiling is high enough to make the cloud path a deliberate exception rather than a reflex. The exceptions are real and worth naming precisely: a genuinely enormous model that no on-site budget can host and whose capability the task genuinely needs; a burst workload too spiky to size hardware for; a fleet of thin edge sites that cannot each host a box and whose data is *permitted* to leave. Each of those is a specific, defensible reason. "It's easier" is not one, and in the machine domains the data-residency and availability constraints usually make the cloud path narrower than it looks. When it is justified, it is the *top* of a hierarchy whose base is local, reached only through the explicit boundary Chapter 8 drew — which renderings may leave the wall, stripped of what must not, logged as having left.

## The bandwidth wall, made usable

Because the placement decision rests entirely on the memory-bandwidth fact, it is worth having a tool that turns it into numbers you can plan against before you buy anything. The following is a back-of-envelope estimator — stdlib Python, runnable in a restricted sandbox — that computes the decode-speed upper bound for each rung, the spill penalty when a model does not fit VRAM, and a single request's latency budget. It is an *upper bound* you calibrate against one real measurement, not a spec sheet; its job is to tell you which rung is even in the right order of magnitude.

```python
#!/usr/bin/env python3
"""placement.py -- a back-of-envelope decode-throughput and latency estimator.

Decode from a transformer is memory-bandwidth bound: each generated token
reads (roughly) the active weight bytes once. So a first-order estimate of
warm single-stream decode speed is

    tokens_per_second  ~=  memory_bandwidth  /  active_bytes_per_token

This ignores compute, attention/KV traffic, and kernel overhead -- it is an
UPPER bound you calibrate against a real measurement, not a spec sheet. Its
job is to tell you which hardware rung is even in the right order of
magnitude before you buy it. Stdlib only.
"""

def decode_tok_s(active_gb, bandwidth_gb_s, efficiency=0.7):
    """Roofline upper bound, de-rated by a measured efficiency factor.

    efficiency folds in everything the clean model ignores (attention/KV
    reads, kernel launch overhead, imperfect bandwidth). 0.6-0.8 is the
    range this lab observes on warm single-stream decode; 0.7 is a safe
    planning default. Calibrate it against one real run and reuse it.
    """
    ideal = bandwidth_gb_s / active_gb          # tokens/s if perfectly bw-bound
    return ideal * efficiency


def spilled_bandwidth(vram_gb_s, ddr_gb_s, fraction_on_ddr):
    """Effective bandwidth when part of the active weights live in DDR.

    Per generated token the engine must read the VRAM-resident share at VRAM
    speed and the spilled share at DDR speed; the two happen in series, so
    the TIME adds and the effective bandwidth is the harmonic blend. This is
    why a little spill costs a lot: the slow leg dominates the sum.
    """
    f = fraction_on_ddr
    # time per unit of weight = f/ddr + (1-f)/vram ; effective bw = 1/time
    time_per_unit = f / ddr_gb_s + (1.0 - f) / vram_gb_s
    return 1.0 / time_per_unit


def latency_budget(tok_s, prompt_tokens, output_tokens, prefill_tok_s):
    """Wall-clock to first token and to completion for one request."""
    ttft = prompt_tokens / prefill_tok_s        # time to first token (prefill)
    gen  = output_tokens / tok_s                 # decode time
    return ttft, ttft + gen
```

Driving it across the rungs (a 7B-class model, ~4.5 GB of active 4-bit weights, efficiency 0.7) produces the transcript below. The numbers are illustrative of the *shape* of the ladder, not a benchmark of any specific product:

```
=== Decode-speed upper bound by hardware rung ===
(model: a 7B-class model, ~4.5 GB active at 4-bit; eff=0.7)

  MCU class (not an LLM)          --      deterministic code / tiny classifier -- see text
  Raspberry Pi 5 (LPDDR4X)           2.6 tok/s   ~sub-1B models only, seconds/token at 7B
  Industrial PC (DDR5 dual)         12.9 tok/s   CPU-only; small models usable, large ones spill hard
  On-site GPU (VRAM)               149.3 tok/s   one prosumer accelerator, model must fit VRAM

=== The spill penalty (active weights split VRAM/DDR) ===
(VRAM 960 GB/s, DDR5 83 GB/s, per generated token)

      0% on DDR5 ->  eff bw  960.0 GB/s ->   149.3 tok/s
      5% on DDR5 ->  eff bw  628.1 GB/s ->    97.7 tok/s
     10% on DDR5 ->  eff bw  466.8 GB/s ->    72.6 tok/s
     25% on DDR5 ->  eff bw  263.6 GB/s ->    41.0 tok/s
     50% on DDR5 ->  eff bw  152.8 GB/s ->    23.8 tok/s
    100% on DDR5 ->  eff bw   83.0 GB/s ->    12.9 tok/s

=== One request's latency budget ===
(on-site GPU, no spill; 2048-token prompt, 256-token answer)

  time to first token :   1365 ms
  total (256 tokens)  :    3.1 s
  decode rate         :    149 tok/s
```

Two lessons jump out of the transcript, and both are load-bearing for the rest of the chapter. The first is the two-order-of-magnitude gap between the Pi and the GPU — 2.6 versus 149 tokens per second on the *same model* — which is not a compute gap (both could do the arithmetic) but a bandwidth gap, and which tells you that the rung is chosen by how fast the answer must appear, not by how "smart" the model needs to be. The second is the spill table, and it is the one that surprises people: moving just *5%* of the active weights from VRAM to DDR5 nearly halves throughput, and 25% cuts it by almost two-thirds. Spill is not a graceful degradation; it is a cliff, because the slow leg dominates the harmonic blend. That cliff is the single most important thing to understand about running a model that does not quite fit its accelerator, and it is worth spending the rest of the sizing discussion on.

## Making the big model fit: quantization, honestly

The reason the spill cliff matters so much is that owners are perpetually tempted to run a model one notch too large for their VRAM, and the usual first response is quantization — storing the weights at fewer bits so the model shrinks. Industrial Nº 2 develops the quantization arithmetic in full `[R62]`; the placement-relevant summary, grounded in this lab's own measurements, is short and has sharp edges.

Quantization trades bits for quality, and the trade is **not uniform across the model.** This lab's central measured finding, on a large mixture-of-experts model, is that **the routed experts are the quality lever and the rest of the model is cheap to protect.** Attention, the indexer, and the normalization layers can sit at 8-bit for almost nothing; the routed experts are where precision buys or loses capability. The measured ladder, on this lab's tool-and-knowledge benches: 2-bit experts and 3-bit (`Q3_K`) experts land at roughly the same tool-use score (~46 on a 100-point tool suite), 4-bit (`Q4_K`) jumps to ~60, and the model's original native 4-bit format (MXFP4) lands around 55 with a better worst-case floor `[LAB: RogerAI Labs bench, RESULTS-MATRIX §C, 2026-07-13; MMLU 100-question sample and a 15-scenario tool suite on DeepSeek-V4-Flash quants]`.

The subtlety that makes this actionable: **knowledge and tool-use recover at different bit-widths.** General knowledge (MMLU) is largely back by 3-bit experts (84.0 versus 85.0 for the 4-bit build), but tool-calling — the structured, do-the-right-thing-with-the-schema capability that machine deployments live on — does *not* recover until 4-bit or the original bits. So the sizing rule for a machine deployment is sharper than "quantize until it fits": **protect the experts to at least 4-bit if the model must call tools or emit structured assertions reliably, and spend the savings elsewhere.** A 3-bit build that looks fine on a knowledge benchmark can be quietly worse at the exact capability your machine application depends on, and a knowledge score will not warn you.

There is one more quantization rule that is pure trap-avoidance, learned by this lab the expensive way: **never requantize sideways.** Weights already stored in a low-bit format only lose from being re-encoded onto a different same-width grid, and can even *grow*. This lab aborted a requant run eight minutes in when the load log showed the conversion turning the native 4-bit experts into a larger 4-bit format *and* adding conversion loss — worse on both axes at once. Quantize *downward*, on purpose, to shrink; or ship the original bits; but never move sideways between formats of the same width `[LAB: RogerAI Labs, RESULTS-MATRIX §F, 2026-07-13; MXFP4→Q4_K conversion grows the expert tensors]`. The general lesson for a placement decision: the "which quant" question has a right answer that depends on your capability floor and your VRAM, and it is cheaper to measure it once than to discover a subtle capability loss in production.

## Speculation, and the counter-intuitive draft length

The other lever for making a large model interactive is speculative decoding — drafting several tokens cheaply and verifying them in a single expensive pass, so that when the draft is right you get multiple tokens for one weight-read `[R63][R64]`. It is a genuine win, and this lab ships it in production. But it interacts with the bandwidth wall in a way that inverts the naive intuition, and the inversion is worth stating because it will save someone a bad configuration.

The naive intuition is "draft more tokens, go faster." The measured reality on a *spill-bound* mixture-of-experts model is the opposite: **draft length one is the sweet spot when the model is spilled, and drafting more is slower.** The reason is exactly the bandwidth wall. Verifying N drafted tokens means the experts those tokens route to must be read from memory, and on a spilled model those reads come from slow DDR5 — so verifying N tokens costs roughly N times the expensive expert reads, and the batch-verify eats its own speculative savings. This lab measured the speedup from single-token drafting growing as spill shrinks: about 1.18× at 14 layers spilled, 1.30× at 10 layers, and 2.2× at zero spill on a model that fits entirely in VRAM `[LAB: RogerAI Labs bench, RESULTS-MATRIX §E, 2026-07; DeepSeek-V4-Flash MTP head, draft acceptance measured per arm]`. The first drafted token, notably, is accepted essentially 100% of the time — it is what the draft head was trained to predict — so single-token speculation captures the reliable part of the gain without paying the batch-verify tax.

The placement lesson is not the specific numbers; it is the *shape*. Speculation's payoff is a function of how memory-bound you are, and the more spilled you are the less aggressive your drafting should be. If you buy the accelerator that lets the model fit entirely in VRAM, speculation pays back more than double; if you run spilled, keep the draft short and expect a modest gain. Either way, "turn speculation up to eleven" is a misconfiguration on a spilled model, and this is the kind of thing you learn only by measuring your own box, which is the theme of the next section.

## The engine is a first-class variable

Here is a placement fact that spec sheets cannot tell you and that this lab learned as its founding mystery: **the same weights on the same hardware can run at wildly different speeds depending on the inference engine and its build.** This is not a small effect. On one 102 GB model, on this box, this lab measured the same weights running at roughly 2 tokens per second on a pre-fix build, versus around 26 tokens per second stable on a build with a corrected GPU code path — a more-than-tenfold difference from *software alone*, same model, same GPUs `[LAB: RogerAI Labs, RESULTS-MATRIX §A, 2026-07; engine lineage on a single UD-IQ3_XXS 102 GB model]`. The 2-tok/s pathology was not a hardware limit and no flag fixed it; it fell to a single line in the load log revealing a CPU-resident indexer that should have been on the GPU. That is the origin of one of this lab's standing rules — *when no flag moves a pathological number, stop tuning and read the load log* — and it is a placement rule as much as a debugging one: **the engine and its exact build are part of the specification of "where the model runs," not an implementation detail.**

The engines that matter for on-site deployment are mature, open, and actively maintained. `llama.cpp` `[R57]` is the workhorse for CPU, mixed CPU/GPU, and the spill regime this lab operates in — it is what this box has run in production for its entire life, and its GGUF weight format `[R59]` is the portable artifact the deployment record checksums. `vLLM` `[R58]` is the throughput-oriented server for the case where the model fits VRAM and concurrency is the goal. The choice between them is a real placement decision with measured consequences, and it belongs in the deployment record next to the weights checksum.

Two engine behaviors have direct placement consequences worth naming because they bite. **Concurrency scales throughput but not single-stream latency.** Batch serving means many requests share the expensive weight-reads, so aggregate tokens per second climbs with concurrency — this lab measured a single-stream ~26 tok/s becoming ~46 tok/s aggregate at four concurrent requests on one configuration — but each individual request does not get faster, and past the memory ceiling the honest failure is a fast "server busy" rather than a graceful slowdown `[LAB: RogerAI Labs bench, RESULTS-MATRIX §B, 2026-07]`. Size for your worst concurrent burst, not your average. And **placement flags interact with weight-layout optimizations in ways that produce silently broken configurations** — high CPU-offload settings that segfault without a repacking flag disabled, forced full-loads that take the host down when the file exceeds RAM, a KV-cache precision that corrupts a specific model's output. This lab keeps a small museum of these; the operational rule is that any placement change is re-measured against the gate's throughput probe and the load log is read in full before the change is called done `[LAB: RogerAI Labs, RESULTS-MATRIX §F and serving traps, 2026-07]`.

## The cache is a line item, not a rounding error

There is a second consumer of fast memory that sizing exercises routinely forget, and it is the one that turns a comfortable fit into a spilled one at the worst possible moment: the attention cache. As the model processes a conversation or a long document, it stores intermediate state — the key/value cache — for every token in the context, and that cache lives in the same fast memory the weights compete for. Its size scales with the context length and the number of concurrent requests, and unlike the weights (a fixed, known quantity) the cache grows with *use*.

The placement consequence is that VRAM must be sized for weights *plus* the worst-case concurrent cache, not weights alone. A model that fits VRAM with gigabytes to spare when serving one short request can push itself onto the spill cliff — or fail outright — when a shift change brings a burst of concurrent long-context requests, each dragging a full cache behind it. This lab has the scar: a configuration sized on average cache use hit its ceiling under burst and the honest failure was a request that could not allocate. Two disciplines follow. First, size the cache headroom for your worst realistic concurrent burst — the shift-change number, not the daily average — because that is the moment the tool is most needed and most loaded at once. Second, treat the cache's *precision* as a gated change, not a free flag: quantizing the cache to fewer bits saves real memory and is usually safe, but this lab measured a specific model whose output *corrupted* when its cache was quantized to 8-bit, and the cache went back to 16-bit and stayed there by standing rule `[LAB: RogerAI Labs, RESULTS-MATRIX serving traps, 2026-07; q8_0 KV cache corrupts this model's output]`. The general lesson: cache precision is a per-model property you verify, not a memory-saving default you assume. The cache is where "it fit in the demo" and "it fell over in production" most often diverge, and the divergence is entirely a sizing-and-precision decision you can make correctly in advance.

## The two-model split, placed

The single most important placement pattern in this book is not a single box; it is a *split* across two rungs, and it is worth stating explicitly here because it resolves the tension between the rungs rather than choosing among them. A small, fast reader lives at the low rung — the gateway or industrial-PC class, next to the machine — always on, emitting Chapter 4's typed assertions and, crucially, *abstaining and escalating* whatever it cannot answer confidently. A larger, more capable model lives at the high rung — the on-site GPU box or plant datacenter — reached only for the cases the edge reader escalates.

This split is a placement decision, and it maps cleanly onto everything above. The edge reader is chosen for latency and always-on reliability at low cost; it runs a Pico- or Nano-class model where a few tokens per second of structured output is plenty, on a rung that survives a network outage because it does not need the network. The central model is chosen for capability; it runs the large, expert-protected model that the sizing worksheet justified, and it earns its cost by serving many edge readers rather than one. The traffic between them is Chapter 11's escalation packet flowing *up*, and the trained improvements flowing *down* (Chapter 7's loop). The design rule from Chapter 8 governs the placement: capability flows down toward the edge, data flows up toward the center, and neither crosses a boundary silently.

The reason this matters for placement specifically is that it lets you buy the *right* hardware at each rung instead of one compromised box for everything. A single box sized to run the large model at every edge location is unaffordable and unnecessary; a single box sized for the edge cannot run the capable model the hard cases need. The split lets the edge be cheap and abundant and the center be capable and shared, and it is the architecture that makes the whole "local ceiling is high enough" premise of this series affordable at fleet scale. When you hear "where does the model run," the most common right answer is *two places, deliberately*, and the escalation packet is the seam that makes the two behave as one system.

## Latency, residency, and version: the three constraints that pick the rung

Capability picks the model; three physical constraints pick the rung it runs on. Take them in the order they usually bind.

**Latency.** The question is not "how fast is the model" but "how fast must the answer appear, and measured how." Separate two numbers the transcript above already introduced: time-to-first-token (dominated by prefill, i.e., reading the prompt) and the decode rate (tokens per second thereafter). A typed-fault assertion of a dozen tokens is dominated by time-to-first-token; a streamed paragraph for a human is dominated by decode rate. A background triage job that files a work order within a minute has an enormous latency budget and can run on a slow rung or a shared queue; an operator asking a question and watching a cursor needs decode fast enough not to feel broken (roughly, faster than reading speed, which the GPU rung clears easily and the Pi rung does not). Match the rung to the *tightest* latency any application on it must meet, because the box is shared. And measure it warm, under realistic concurrency, not from a cold single-stream demo — which is where the next constraint's discipline starts.

**Data residency.** The constraint the cloud path most often fails and the machine domains most often impose. If the data may not leave the building — regulatory, contractual, or because the plant simply does not route the process network to the internet — then the rung is *inside the wall*, and the cloud path is off the table regardless of its convenience. This is not a preference to be traded against latency; it is frequently a hard boundary, and the whole local-first architecture of this series exists to make honoring it the default rather than a sacrifice. Chapter 8's air-gap shape is residency taken to its logical end: the model, the engine, and the gate all live inside the wall, and the update path is an explicit, checksummed ceremony rather than an assumed network. The placement rule is blunt: **residency is a veto, applied first.** Establish where the data may be before comparing rungs, because it eliminates whole rungs at a stroke.

**Version pinning and reproducibility.** The constraint that spec sheets never mention and audits always find. In a machine deployment, "which model answered this, on which engine build, with which contract and schema" must be *recorded, not remembered* — because a work order drafted six months ago may be questioned, an incident investigation may need to reproduce a verdict, and a regulator may ask. This is a placement constraint because reproducibility is easy on some rungs and nearly impossible on others: a pinned weights file (checksummed, versioned like firmware) plus a pinned engine build plus a pinned contract, all on a box you control, is reproducible by construction; a cloud endpoint that silently updates its model under you is not. Every response carries its weights checksum, engine version string, contract version, and schema version — the same six-artifact discipline Chapter 8's deployment record demands — and the rung that makes that pinning trivial is the local one. A model whose exact behavior on a given input you cannot reproduce on demand is not deployable next to a machine that can hurt someone, and that requirement quietly favors the rungs where you own the stack.

## The total-cost worksheet, with the assumptions labeled

Procurement wants a number, and the honest way to give one is a worksheet with its assumptions written on it, so that the number can be audited and updated rather than believed. This is Chapter 8's sizing worksheet turned toward *placement cost*, and every line names its assumption because a total-cost figure with hidden assumptions is a sales tool, not an engineering document.

1. **The accelerator (or its absence).** The dominant capital line on the GPU rungs. State the assumption that picks it: the model size at your chosen quant plus engine overhead plus cache headroom must fit VRAM to stay off the spill cliff — and the spill table above is the cost of getting this wrong. If the model does not fit one accelerator, the choice is a bigger one, several smaller ones (with their own placement physics), or accepting measured spill. Name which, and name the measured throughput at that choice, not the spec.
2. **Memory: system RAM and VRAM, sized for the worst burst.** Cache headroom for your worst concurrent shift-change burst, not your average — the line most sizing exercises omit and the one that turns into a fast "server busy" at the wrong moment. State the concurrency assumption explicitly.
3. **Power and cooling, at sustained load in the real enclosure.** Not the lab bench in October; the mezzanine or the electrical room in August. A box that thermally throttles silently fails its latency budget, so the assumption to write down is the sustained-load thermal rating of the enclosure it will actually live in.
4. **The spare, at cold-standby cost.** Reproducibility's physical form: rollback and recovery are a copy of the weights on a second box. State whether the spare is cold, warm, or absent, because that assumption is the difference between a ninety-minute recovery and a next-day one.
5. **Operations, amortized.** The rung's ongoing human cost — the sidecar box someone restarts, the datacenter rack someone schedules, the cloud bill someone watches. State the assumption about who owns it at 2 AM, because that is a real recurring cost the capital line hides.
6. **The cloud comparison, computed honestly.** If a cloud path is a candidate, compute it at *your* sustained token volume over the *amortization horizon* of the local box, and state both assumptions — because the cloud path's per-token pricing is attractive at demo volume and frequently loses at sustained production volume, and the crossover is exactly what this line exists to find. The residency veto may make this comparison moot; if it does not, do the arithmetic with the assumptions visible.

Sum it, round up one hardware notch — the marginal cost of headroom at purchase is a fraction of the cost of discovering its absence in production — and staple the worksheet, assumptions and all, to the deployment record. When a line came from a measurement on your own box rather than a spec sheet, say so; procurement respects an instrumented number and an audit remembers one.

## Measure your own box: the ±10-point warning

One discipline binds every number in this chapter and deserves its own closing note, because it is the difference between a placement decision you can defend and a lucky draw you will regret. **Benchmark numbers on small suites swing, and you must publish ranges, not single runs.** This lab measured a 15-scenario tool-evaluation suite swinging by about ±10 points across identical runs at temperature zero — pure nondeterminism from batch-packing order, not any change in the model `[LAB: RogerAI Labs, RESULTS-MATRIX §C notes, 2026-07]`. A placement decision made on a single lucky benchmark run — "this quant scores 60, ship it" — is a decision made on noise. The rule this lab operates by, and recommends: one surprising number gets run again; two runs that disagree get a third and a control that isolates the variable; and every number that reaches a deployment record carries its error bar. The gate that authorizes a configuration for production runs the throughput probe warm, under realistic concurrency, more than once — this lab's own production promotion carried a burn-in soak (dozens of requests, zero errors, measured acceptance and VRAM drift) precisely so the "it works" claim was a measurement and not a hope `[LAB: RogerAI Labs, RESULTS-MATRIX §G, 2026-07]`. The placement decision is a measured claim about your specific box, your specific engine build, your specific model, under your specific load — and it is only as trustworthy as the error bars you were willing to put on it.

## The chapter in one sentence

Decode is bound by memory bandwidth, so the rung the model runs on is chosen by how fast the answer must appear, how much of the model fits fast memory before the spill cliff, and three physical constraints applied in order — residency as a veto, latency as a budget, and version-pinning as a reproducibility requirement — while capability is bought with expert precision protected to the level your structured outputs actually need; and every number in that decision is a measured claim about your own box, published with its error bar, not a spec sheet believed.

The next chapter takes the last step this book has been building toward since Chapter 2: given that the model now runs somewhere real, next to something that can move or heat or energize, *who is allowed to let it act* — and why the most valuable thing it can do at that boundary is often to stay silent and hand a human enough to decide.

---
