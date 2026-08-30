# Chapter 3 — Sensors and the Signal Path

*(draft v1, 2026-08-30 — written by claude-fable-5 (RogerAI Labs), unverified. Published sources carry `[R#]` markers resolved in the References; the author's own reproducible bench measurements carry `[LAB: …]` markers, stated with the apparatus and sample size that produced them. Per-chapter authorship is recorded in `manifest.json`.)*

A model on the chill line reads this object from the historian feed:

```json
{"tag": "TT-114.PV", "value": -24.7, "unit": "degC", "ts": "2026-08-30T02:40:02Z"}
```

It is clean, well-typed, plausible, and a lie. There is no −24.7 °C anywhere in that
refrigeration circuit. The temperature transmitter's wire has broken, the loop current has
fallen to near zero, and somewhere upstream a careless decode scaled that near-zero current
straight through to an engineering value as if it were a real reading. The number is not a
measurement. It is an artifact of a fault, wearing the costume of a measurement, and the model
— which reads it as a sentence, a claim about the world stated in a confident voice — has no way
to know the difference unless someone gave it one.

This chapter is about the distance between a physical phenomenon and the bytes a model reads,
and about everything that can happen to the truth along the way. The previous chapter's first
rule was *decode structure before the model sees it.* This chapter is that rule taken seriously
for sensors, because the decode is where most of the lies are born, and because the single most
important fact a machine owner can internalize about language models and sensors is the one the
authors measured and this chapter builds toward: **a general language model does not read a tag
value as a measurement with a history. It reads it as a sentence, and it cannot, on its own,
tell a dirty sensor from a clean one.**

## The signal path, stage by stage

A sensor reading is not a fact that exists and gets transmitted. It is manufactured, in stages,
and every stage is a place where the number can drift away from the physical truth it claims to
represent. Walk the path once, end to end, and the failure modes stop being mysterious.

**The phenomenon.** Something physical is happening — a temperature, a pressure, a flow, a
vibration, a current. This is the only thing that is actually real. Everything downstream is a
representation, and a representation can be wrong while looking fine.

**The transducer.** A sensing element converts the phenomenon into an electrical quantity — a
thermocouple's tiny voltage, an RTD's resistance change, a piezoelectric charge, a strain
gauge's imbalance. The conversion has a transfer function that is never perfectly linear, drifts
with temperature and age, and has a limited range outside which it saturates or lies. A
thermocouple reads the temperature *difference* between its junction and a reference, so it is
only as good as its cold-junction compensation; get that wrong and every reading is off by the
reference error. None of this is visible in the final number.

**Signal conditioning.** The raw electrical signal is amplified, filtered, linearized, and often
converted to a standard transmission form — most commonly the 4–20 milliamp current loop
introduced in Chapter 1. Filtering is a double-edged stage: a filter that removes noise also
removes real fast transients, and a filter time constant that is too long will smooth over
exactly the excursion you needed to catch. The choice of what to filter out is a choice about
what the downstream model is even capable of noticing.

**Sampling and quantization.** An analog-to-digital converter samples the conditioned signal at
some rate and rounds it to a finite number of steps. Two classic distortions live here. The
first is *aliasing*: sample a signal too slowly and a high-frequency component folds down and
masquerades as a low-frequency one — a vibration at 55 Hz sampled at 60 Hz can appear as a slow
5 Hz wobble that is not physically there. The second is *quantization*: a 12-bit converter has
4096 steps across its range, so it cannot resolve anything finer, and a reading that looks
precise to two decimal places may be resting on a step much coarser than those decimals suggest.

**Scaling to engineering units.** The digital count is mapped to a physical value — counts to
milliamps to degrees. This is deterministic arithmetic, and it is exactly where the lie in the
opening JSON was born. Get the range, the zero, the sign, or the units wrong and every reading
downstream is confidently, precisely incorrect. It is also, crucially, the calculator trap from
Chapter 2 made physical: this arithmetic must be done in deterministic code, never delegated to
the model, because a model asked to scale a raw count will sometimes get it right and sometimes
produce a fluent wrong number with no tell.

**The tag and its metadata.** The value is stored against a tag name — `TT-114.PV` — that
encodes, by convention, what it is: a temperature transmitter, loop 114, process value. The tag
carries metadata: engineering units, range, description, and, in a well-built system, a quality
status and a timestamp. The instrumentation identification conventions that give tags their
structure are standardized — ISA-5.1 defines the symbols and identification scheme that make
`TT`, `PT`, `FT` mean what they mean [R12] — and the tag name is often the only context a model
gets about what a number *is*. A tag named badly, or a value handed to the model without its
tag, strips away the little context there was.

**The historian.** Years of tag values are stored, usually compressed with a deadband or
swinging-door algorithm that discards points deemed uninteresting to save space. This means the
history a model reads is not raw; it has already been thinned by an algorithm that decided which
changes mattered, and a reading you retrieve may be interpolated between stored points rather
than a value that was ever actually sampled. The historian is a lossy recording of the past, and
the model reading it is reading the recording, not the past.

**The bytes the model reads.** Finally the value arrives at the model, as text in a context
window, stripped of everything except what the pipeline chose to include. If that text is the
bare number, the model has lost the entire chain above and will treat −24.7 as a temperature.
If that text is the number *plus its status, its units, its timestamp, and its recent behavior*,
the model has a chance to read it as what it is: a measurement with a history, some of which may
be untrustworthy.

## Quality is part of the value, and the protocols already know it

The opening lie was preventable, and the fix is old news to the instrumentation world: a value
without a quality status is only half a reading. This is not an idea this book invented. It is
built into the serious industrial data models. OPC UA, the modern factory information model,
does not represent a variable's value as a bare number. Its `DataValue` structure carries the
value *together with* a `StatusCode` — an explicit quality indicator that can say Good,
Uncertain, or Bad with a specific sub-reason — and a source timestamp and a server timestamp
[R8]. The protocol designers understood that "the value is 15.0" and "the value is 15.0 but the
sensor is reporting a fault" are different messages, and they made it impossible to transmit the
number without room for the caveat. A pipeline that reads OPC UA and then discards the StatusCode
before handing the value to a model has thrown away the exact bit that would have caught the
broken wire.

The transducer world encodes the same wisdom further upstream. The IEEE 1451 family of smart-
transducer standards defines a Transducer Electronic Data Sheet — a TEDS — that travels with the
sensor and carries its identity, its units, its range, its calibration parameters, and its
uncertainty, so that the digitized reading arrives already self-describing [R9]. The whole thrust
of that standardization effort is that a number is not interpretable without the metadata that
says what it is and how much to trust it. A model deployment that respects this is one that
carries units, range, status, and — where available — calibration and uncertainty all the way to
the context window, and treats a value that arrives without them as suspect by default.

The listing below shows the decode step doing this job. It takes raw ADC counts from a 4–20 mA
temperature loop and produces two outputs for each: the naive value a careless pipeline would
hand the model, and a quality-aware record that carries a status. The loop is scaled so the ADC
can see past 20 mA and below 4 mA, which is how real loop monitors detect faults — the NAMUR NE43
convention reserves currents below about 3.6 mA and above about 21 mA to signal that the
transmitter itself has failed, extending the old 4 mA-floor idea into an explicit fault-signaling
band.

```python
ADC_BITS = 12
ADC_FULL = (1 << ADC_BITS) - 1          # 4095 counts = full scale
I_MIN_MA, I_MAX_MA = 4.0, 20.0          # nominal signal band
SPAN_ADC_TOP_MA = 24.0                  # full-scale counts == 24 mA, so the ADC
                                        # can see the NAMUR NE43 fault bands past
                                        # 20 mA and below 4 mA, not just 4..20
EU_MIN, EU_MAX = 0.0, 100.0             # engineering units at 4 and 20 mA
EU = "degC"

def counts_to_ma(counts):
    return SPAN_ADC_TOP_MA * counts / ADC_FULL

def naive_value(counts):                # scales counts straight to EU, no checks
    ma = counts_to_ma(counts)
    frac = (ma - I_MIN_MA) / (I_MAX_MA - I_MIN_MA)
    return round(EU_MIN + frac * (EU_MAX - EU_MIN), 2)

def decode(counts, ts):                 # value AND a status the reader must respect
    ma = counts_to_ma(counts)
    rec = {"ts": ts, "raw": counts, "mA": round(ma, 3),
           "value": None, "unit": EU, "status": "GOOD"}
    if ma < 3.6:                        # NE43 low: broken loop
        rec["status"] = "BAD_WIRE_BREAK"; return rec
    if ma > 21.0:                       # NE43 high fault: >21.0 mA (20.0-21.0
        rec["status"] = "BAD_OVER_RANGE"; return rec  # is valid over-range, not a fault)
    frac = (ma - I_MIN_MA) / (I_MAX_MA - I_MIN_MA)
    rec["value"] = round(EU_MIN + frac * (EU_MAX - EU_MIN), 2)
    return rec
```

Run against a healthy mid-scale reading, a healthy high reading, a wire break, and a pinned-high
fault, it produces:

```
naive decode (what a careless pipeline hands the model):
  raw=1706  ->    37.49 degC
  raw=3242  ->    93.75 degC
  raw=   7  ->   -24.74 degC
  raw=4095  ->    125.0 degC

quality-aware decode (value carries a status):
  {"ts": "...:02Z", "raw": 1706, "mA": 9.999,  "value": 37.49, "unit": "degC", "status": "GOOD"}
  {"ts": "...:01Z", "raw": 3242, "mA": 19.001, "value": 93.75, "unit": "degC", "status": "GOOD"}
  {"ts": "...:02Z", "raw":    7, "mA": 0.041,  "value": null,  "unit": "degC", "status": "BAD_WIRE_BREAK"}
  {"ts": "...:03Z", "raw": 4095, "mA": 24.0,   "value": null,  "unit": "degC", "status": "BAD_OVER_RANGE"}
```

The naive column is the disease. The broken wire becomes a plausible −24.7 °C and the pinned
sensor becomes a plausible 125 °C — two numbers a model will happily reason about as if they were
temperatures, drawing confident wrong conclusions about a circuit that is actually reporting
*nothing*. The quality-aware column is the cure: the value is `null`, the status names the fault,
and any downstream reader — model or human or rule — is told, in the data, that there is no
measurement here to interpret. This listing is deterministic stdlib code and runs in well under a
second in a restricted sandbox; the arithmetic is exactly the kind that must never be handed to
the model.

## Calibration, drift, and the value that is precise and wrong

Quality status catches the gross faults — the broken wire, the pinned sensor. It does not catch
the quiet one, which is worse because it looks perfect: the sensor that is working, reporting a
Good status, and slowly, steadily wrong.

Every sensor drifts. A calibration valid on the day it was performed decays as the sensing
element ages, fouls, or shifts. A pressure transmitter reading 2% high reports a clean, Good-
status, plausible number that is simply not the pressure. There is no fault code for "correct-
looking and mis-calibrated," and this is the failure mode most likely to fool both a model and
the human reading the model's output, because everything about the number invites trust.

The metrology world has precise language for the two properties that matter here, and a machine
owner should borrow it. *Traceability* is the property that a measurement can be related, through
an unbroken chain of calibrations, to a recognized reference standard — it is what makes one
plant's "15.0 °C" comparable to another's, and it is a core concept of the international
vocabulary of metrology [R11]. *Measurement uncertainty* is the quantified doubt attached to a
reading — the ± that says how wide the true value's plausible range is — and the international
guide to expressing it exists precisely because a value without its uncertainty is an incomplete
statement of a measurement [R10]. A reading of "15.0 °C" and a reading of "15.0 ± 0.5 °C" carry
different information, and the second is the honest one.

The consequence for a language-model deployment is direct and often ignored: a model handed bare
numbers cannot reason about calibration or uncertainty it was never given, and it will not
invent the caution on its own — it will treat a mis-calibrated 2%-high reading with exactly the
confidence it treats a perfect one. If your deployment cares about the difference between a
genuine excursion and a drifting sensor, the calibration state and the uncertainty have to reach
the model as data — last calibration date, drift history, the sensor's stated accuracy — and the
model has to be prompted to weigh them. Even then, this is a place to be modest about what the
model adds: detecting drift is often better done by deterministic comparison against a redundant
sensor or a mass balance than by any amount of language-model reasoning. The model's useful role
is to *notice and surface* that a reading's calibration is stale, not to be the drift detector.

## Redundancy is how you catch the quiet lie

The calibration lie — the working sensor that is slowly, plausibly wrong — cannot be caught from a
single channel, because there is nothing in one channel's number to contradict it. It can be caught
by *comparison*, and this is worth stating because it is both the classic instrumentation answer and
a clean example of a job that belongs to deterministic logic rather than to a language model.

The oldest form is physical redundancy: two or three sensors measuring the same quantity, compared
against one another. When three agree and one disagrees, the odd one out is the suspect, and a voting
scheme — two-out-of-three, in the language of safety instrumentation — can carry on trustworthily
while flagging the deviating sensor for maintenance. This is a deterministic computation over a few
numbers, exact and auditable, and it detects drift that no amount of language-model reasoning over a
single channel could find, because the information simply is not in a single channel. The softer form
is analytical redundancy: check a sensor against a relationship it must obey — a mass balance, an
energy balance, a known correlation between two process variables — so that a reading which violates
physics is flagged even without a duplicate sensor.

The reason this belongs in a chapter about language models is the division of labor it illustrates
one more time. Detecting that two of three transmitters agree and one has drifted is arithmetic; do
it in deterministic code, where it is exact and provable. The language model's contribution is
downstream and linguistic: reading the *result* of that comparison — "transmitter TT-114C reads 2.1
°C above the other two and its last calibration was fourteen months ago" — and turning it into a
work order, a maintenance recommendation, and a plain-language explanation for the operator, with the
comparison cited as its evidence. The model is not the drift detector. It is the thing that explains
the drift detector's finding and drafts the response, which is exactly the role this book keeps
returning it to.

## Missingness: a gap is data, and the pipeline usually erases it

Sensors go silent. A network drops, a device reboots, a scan skips, a historian's compression
decides a flat stretch was not worth storing. What happens to that gap on the way to the model
determines whether the model reads the truth or a fabrication.

The dangerous default is that the gap gets filled invisibly. A pipeline that reports the last-
known value forever will hand the model a value that looks live and is hours stale — the same
memory trap from Chapter 2, now baked into the data layer. A pipeline that linearly interpolates
across a gap will hand the model points that were never sampled, smoothing over exactly the
dropout that might have been the fault. In both cases the model receives a clean, gap-free series
and has no way to know that part of it is invented. The absence of data has been converted into
the presence of fake data, which is strictly worse than an honest hole.

The protocols that take this seriously make missingness explicit rather than papering over it.
Sparkplug, the industrial MQTT profile treated in the next chapter, requires birth and death
certificates so that a subscriber can distinguish a sensor reporting zero from a sensor that has
gone offline, and requires a stale-data indication when a value can no longer be trusted as
current [R17]. OPC UA's Uncertain and Bad status codes, again, exist precisely to mark a value as
present-but-suspect [R8]. The lesson for a model pipeline is to preserve the gap as a gap: mark
missing periods as missing, mark stale values as stale, mark interpolated points as interpolated,
and let the model see the holes. A gap is information — it may *be* the fault — and a pipeline
that hides it has removed evidence before the model ever got to weigh it.

## Time is a signal too

A sensor reading is not just a value; it is a value *at a moment*, and the moment is as much a
part of the measurement as the number. A model asked to correlate events — did the pressure spike
before or after the valve command? did three alarms fire together or in sequence? — is reasoning
entirely on timestamps, and timestamps have their own long list of ways to lie.

The first problem is *which* time a timestamp names. OPC UA deliberately carries two — a source
timestamp, when the value was produced at the device, and a server timestamp, when the value
arrived at the server [R8] — because they can differ by a lot, and a model told only the second
will mis-order events that were produced in a different order than they were received. A value
buffered on a device during a network outage and dumped all at once when the link returns will
have source times spread across the outage and server times bunched at the reconnect; read the
server times as the truth and you conclude a storm of events happened in one instant that
actually unfolded over an hour.

The second problem is *clock skew*. A plant is full of independent clocks — in PLCs, in gateways,
in the historian, in the edge box — and unless they are disciplined to a common time source they
drift apart. Two events that a model reads as three seconds apart may have been simultaneous, or
reversed, if the two devices' clocks disagreed by more than the gap. Correlating across sources is
only as trustworthy as the time synchronization underneath, and a model has no way to know how
well the clocks agree unless the pipeline tells it. The third is *jitter and resolution*: a
timestamp recorded to the millisecond may have been sampled by a scan that only runs every 100
milliseconds, so the precision of the timestamp overstates the precision of the timing, exactly as
decimal places can overstate the precision of a value.

The consequence for a model deployment is that temporal reasoning is a place to be especially
careful about what the model is handed and what it is allowed to conclude. If the pipeline knows
the clocks are well-synchronized and the timestamps are source times, the model can be trusted to
order events; if it does not, the model should be told the timing is uncertain, and asked to flag
correlations as tentative rather than assert causation. The number of confident wrong conclusions
that trace back to an unstated clock skew is large, and none of them are the model's fault — they
are the pipeline handing the model times it had no business trusting.

## The historian's compression is an editorial decision

It is worth dwelling on one stage that quietly shapes everything a model reads from history,
because most people treat the historian as a faithful recording and it is not. To store years of
thousands of tags, historians compress, most commonly with a deadband or swinging-door algorithm:
a new point is stored only if it deviates from the predicted trend by more than a configured
threshold. Flat and slowly-changing stretches are discarded and reconstructed by interpolation on
retrieval. This is sensible engineering and it has a consequence the model reader must understand:
**the history is not the signal, it is an editorial summary of the signal, and the editor was a
compression threshold.**

The threshold is a choice about what counts as interesting, and it can erase exactly the feature a
fault would have shown. A brief excursion smaller than the deadband simply does not exist in the
stored history; a model asked whether the pressure was ever abnormal will read a series in which
it never was, not because it never was but because the excursion fell below the storage threshold.
Retrieved values that look like samples may be interpolations the algorithm generated on the way
out. And different tags may have different deadbands, so two series a model compares were thinned
by different editors at different rates.

None of this means the historian is untrustworthy — it is one of the most valuable assets a plant
has — but it means a model reading historian data is reading a lossy, edited recording, and the
losses are systematic, not random. Where the compression settings matter to a conclusion, they
belong in the context, and where a value is interpolated rather than stored, it should be marked
as such. A model that knows it is reading an edited history can be appropriately cautious; a model
that thinks it is reading raw truth will over-read the gaps the editor left behind.

## The lies a clean JSON object can tell

Collect the failure modes and you have a checklist of the ways a well-formed, well-typed, entirely
plausible JSON object can misrepresent a dirty sensor. Every one of these produces output that
passes schema validation and looks correct:

- **The fault lie.** A broken or saturated sensor scaled straight to an engineering value, like
  the opening −24.7 °C. Caught only by carrying quality status.
- **The stale lie.** A last-known value reported as if it were live. Caught only by carrying a
  fresh timestamp and comparing it to now.
- **The interpolation lie.** Points that were never sampled, invented to fill a gap. Caught only
  by marking interpolated and missing regions honestly.
- **The unit lie.** A correct number in the wrong units — psi read as bar, °F as °C. Caught only
  by carrying and checking units, ideally from self-describing metadata like a TEDS [R9].
- **The calibration lie.** A precise, Good-status number from a drifted sensor. Caught only by
  carrying calibration state and, better, by deterministic cross-checks.
- **The aliasing lie.** A high-frequency phenomenon folded into a plausible low-frequency reading
  by too-slow sampling. Caught only upstream, by sampling correctly in the first place.
- **The precision lie.** Decimal places that imply a resolution the ADC and calibration cannot
  support. Caught only by knowing the real uncertainty [R10].

The through-line is that none of these is detectable *from the number alone.* Every one requires
context the pipeline can choose to preserve or discard. A model handed the bare number is
defenseless against all seven, and — because it reads the number as a confident sentence — it will
amplify the lie by reasoning fluently on top of it. The engineering work is upstream of the model:
preserve the metadata, mark the faults, keep the gaps, do the arithmetic deterministically, and
hand the model a measurement with a history rather than a naked value pretending to be a fact.

## Why a model reads a tag as a sentence, and the finding that proves it

Everything so far assumes the model, given a clean and honest measurement, could then interpret
it. This section is the hard one, because the authors measured that assumption and it largely
failed, and the failure is the most important calibration in this book.

The task was deliberately minimal and fair. Take a single sensor channel, reduce it to a
deterministic *feature summary* — simple statistics that capture its shape: the overall mean, the
standard deviation of the first half versus the second half, the fraction of samples that are
zero, the fraction pinned at a rail, and so on. These features are exactly the legible fingerprint
of the common faults: a *stuck* signal moved and then stopped, so its early standard deviation is
high and its late one is zero; a *dropout* has a high zero fraction; a *railed* signal sits pinned
at a limit. Hand a model this summary, spell the fault definitions out in the prompt, and ask it
to classify the channel over a closed set — none, stuck, railed, drifting, noisy, dropout — using
the enum-scoring decoder from Chapter 2 so the answer is guaranteed valid and comes with a margin.

Across 300 channels balanced at exactly fifty per fault class, and across models spanning a
hundredfold range in size — a 270-million-parameter model, a sub-one-billion model, a 27-billion,
and, on an identical grammar-constrained harness, a 31-billion and a 72-billion — every model
landed at or below chance on the fault axis [LAB: RESULTS-MATRIX §R.26 and §R.27; chance on six balanced classes is 16.7%; n=300 balanced items, a single deterministic enum-scored run, seed 20260811, byte-reproducible. On n=300 a proportion carries a Wilson 95% interval of roughly ±4–6 points, so the scores below are point observations with that width, not exact values — RogGentoo lab]. The smallest predicted a single class for nearly every channel.
The 27-billion scored 14.3% (Wilson 95% ≈ 10.8–18.7%), at or below chance. And in the cleanest size comparison — the 31B and 72B
run on identical prompt, grammar, and decode — the 31B scored 43% (≈ 37.5–48.7%) and the 72B scored 32% (≈ 27.3–37.8%) [LAB: RESULTS-MATRIX §R.27 — RogGentoo lab]. It is tempting to read that as *more than twice the parameters reading machines worse*, but the honest version is narrower: the two models are a generation apart — the 31B is a 2026-vintage model and the 72B a 2024 one — so the ordering is confounded by model generation and this lab downgraded the bare size claim to provisional. What survives the confound, and is all the argument needs, is that *both* large models sit far below the thirty-line rule baseline, and neither acquires the fault classes that require combining features. Capability on this task is
not simply a function of scale.

The result would mean nothing without its control, and the control is what makes it a finding
rather than a complaint about the authors' features. A hand-written rule — roughly thirty lines of
if-statements over the *identical* feature summary, the fault definitions transcribed directly —
scored 63.3% (Wilson 95% ≈ 57.7–68.6% on the same n=300), with strong per-class recall on the faults the models collapsed [LAB: RESULTS-MATRIX §R.26 — RogGentoo lab]. So the signal is present and legible in the exact input the models received. A model at
chance on an input a thirty-line rule reads at 63% is not suffering from a bad feature layer. It
is telling you something about the model. Three further checks ruled out the harness: the scorer
was verified on a known answer (the France-capital probe from Chapter 2), the prompt was printed
and confirmed well-formed with the features present, and correcting for the model's content-free
prior made accuracy *worse*, not better — so the apparent signal was not signal [LAB: RESULTS-MATRIX §R.25 — RogGentoo lab].

The interpretation, stated at the strength the measurement supports: **general language models
cannot read machines — not the small ones, and not the large ones — because the task is absent
from what they were trained on.** Nothing in web-scale text teaches a model to separate
instrument health from process state given a statistical summary of a signal. The models that did
show partial capability learned only the one or two distinctions stated most explicitly in the
prompt and ignored the rest; not one acquired the faults that require reading a *combination* of
features. This is the deep reason a model reads a tag value as a sentence: it was trained on
sentences, and a sensor channel is not a sentence, it is a measurement with a history, and that is
a genre the training text does not contain.

The honest caveats travel with the finding, because they always do. This used instruct-tuned
models, one prompt formulation, and one decoder; a chat-template formulation and a few-shot
condition should be run before the strongest version of the claim is published, and the channels
were simulated with known ground truth rather than pulled from a live plant. But a hundredfold
parameter range landing uniformly at chance, with a thirty-line control at 63%, is not a sample-
size artifact or a prompt quirk. The effect is real, and its consequence for deployment is
concrete: do not deploy a general model as a sensor-fault classifier on the strength of its
fluency. It does not have the capability off the shelf, and scaling it up will not conjure it.

## What this leaves the model to do, and do well

If a general model cannot classify a sensor fault from features, is there a role for it in the
signal path at all? Yes — and it is exactly the role Chapters 1 and 2 pointed to, sharpened by
this chapter's findings.

The model's strength is structured reading and language, so let it do the parts that are
structured reading and language, and let deterministic code and, where warranted, a
purpose-trained small model do the parts that are measurement. Concretely: deterministic code
decodes the signal with quality, computes the features, applies the cheap rules, and cross-checks
calibration — all the things it does exactly and auditably. The language model then reads the
*results* of that work in context: the decoded measurement, its status, its history, the rule's
verdict, the relevant section of the manual, the tag's description. On that grounded material it
does what it is genuinely good at — summarizing the situation for the operator in plain language,
cross-referencing the fault against the maintenance manual, drafting the structured work order,
and, critically, *citing which tags and which manual section it relied on* so the human can check
it. When the grounded material does not support a conclusion, it abstains, using the measured-
margin discipline from Chapter 2 — and this chapter has shown why that abstention must be measured
rather than assumed, since on the raw classification task the model's confidence signal was empty.

The division of labor is the whole answer. The measurement belongs to instruments, deterministic
decode, and — where the accuracy work has been done — a small specialist model trained on the
task, because that specialist work is the subject of this book's later chapters and the reason the
authors are building such models rather than trusting general ones. The *language about the
measurement* belongs to the language model. Keep those straight and the clean JSON object stops
being a liar, because by the time it reaches the model it is no longer a bare number pretending to
be a fact — it is a measurement carrying its own quality, its own history, and its own honest
gaps. That is the object a model can read without being fooled, and building it is the signal-path
work that makes everything downstream possible.

The next chapter climbs one layer up the stack, from the single sensor's value to the protocols
that carry thousands of them, and asks the same question of Modbus, CAN, OPC UA, MQTT, and the
rest: can a model read the wire? The answer, it turns out, is more encouraging than this
chapter's — because a protocol frame, unlike a sensor channel, really is a kind of language.
