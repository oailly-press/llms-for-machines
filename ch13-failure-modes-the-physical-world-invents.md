# Chapter 13 — Failure Modes the Physical World Invents

*Draft status: author draft, gate-checked; human verification pending. External claims resolve to the
cited references. Lab-specific incidents are cited to the project record by date; anecdotes the author has
not re-verified for this book are labeled unmeasured rather than presented as findings.*

## The model was right; the world lied to it

A software engineer's mental catalog of failures is a catalog of things that go wrong inside the program:
the null that was not checked, the race between two threads, the off-by-one at the edge of the buffer, the
dependency that shipped a breaking change. Those failures are real next to a machine too, and nothing in
this chapter excuses you from them. But they are not the failures this chapter is about, because they are
not the failures that ambush people who come to machines from software. The failures that ambush them are
the ones where the code is correct, the model is behaving exactly as designed, the tests all pass — and the
system still produces a confident, dangerous, wrong answer, because the world handed it an input that was
false in a way the input format had no room to admit.

This is the deep difference between a language model in a chat and a language model on a machine. In a
chat, the input is what the user typed, and while the user may lie or be confused, the text is at least an
authentic record of what was said. On a machine, the input is a rendering of physical reality through a
long chain of sensors, wires, protocols, scaling factors, and clocks, and every link in that chain can
fail in a way that leaves the rendering looking perfectly plausible. The number in the register is a real
number; it is just not the temperature. The frame is well-formed; it is just stitched together from two
different instants. The bit is set; it was set by a technician three shifts ago and forgotten. The model
reads these inputs and reasons over them flawlessly, and its flawless reasoning produces a wrong answer,
because the premise it reasoned from was false and nothing in the premise announced its own falseness. The
NIST guidance on operational technology makes the general point that OT is where the digital meets the
physical and inherits the physical world's failure modes on top of the digital ones [R30]; this chapter is
a catalog of the specific ones that catch language models, and a single lesson runs through all of them:
the defense is almost never a smarter model. It is a frame that carries its own trustworthiness, and a
model trained to refuse a frame that cannot vouch for itself.

## Clock skew: reasoning over a moment that never existed

Begin with time, because time is the failure that software instincts most reliably get wrong. In a single
program on a single machine, "now" is a well-defined thing and two readings taken close together are, for
all practical purposes, simultaneous. On a plant floor, a single reading is assembled from sensors on
different devices, each with its own clock, and those clocks drift apart unless something actively holds
them together. The protocols exist to hold them together — the Network Time Protocol disciplines clocks to
within milliseconds over ordinary networks, and the Precision Time Protocol, IEEE 1588, tightens that to
sub-microsecond on hardware built for it [R81][R82] — but they only help if they are deployed, configured,
and healthy, and in a brownfield plant it is common to find devices whose clocks were set by hand at
commissioning and have wandered ever since.

The consequence for a model is subtle and severe. Suppose the read path assembles a frame that says the
inlet temperature is 84 degrees and the outlet is 79, and the model reasons — correctly, given those
numbers — that the heat exchanger is behaving normally. If the two readings actually came from instants
forty seconds apart because the two devices disagree about what time it is, then the frame describes a
state the machine was never in: a snapshot spliced from two different moments, showing a temperature
difference that never simultaneously existed. The model has no way to detect this from the numbers, because
the numbers are individually valid. It reasons over a moment that never happened and reports a health it
cannot actually see. Clock skew is insidious precisely because it produces plausible frames; a wildly wrong
clock announces itself, but a clock off by seconds produces frames that pass every sanity check and quietly
correlate readings that were never contemporaneous.

The defense is not to make the model cleverer about time; a model cannot infer a clock error it was not
told about. The defense is to make the frame carry its own timing. Every reading that goes into a frame
should carry the timestamp of when it was actually sampled, not when the frame was assembled, and the frame
should carry the spread of those timestamps so the model — and the abstention logic around it — can see when
the readings it is being asked to correlate are too far apart in time to be correlated. A frame whose
readings span forty seconds when the process moves in seconds is a frame the model should refuse to reason
about as a snapshot, and it can only refuse if the timing is in the frame. A read path that strips the
per-reading timestamps and stamps the whole frame with one assembly time has thrown away exactly the
information needed to catch this failure, and it is a very common thing for a read path to do.

## Partial and torn reads: the value caught mid-change

Related to timing, and equally foreign to software instincts, is the torn read. Many industrial values do
not fit in a single register. A 32-bit measurement, a 64-bit counter, a floating-point value — these span
two or more of the 16-bit registers that protocols like Modbus expose, and Modbus itself has no notion of
reading them as one atomic unit; a multi-register read is a sequence, and the underlying value can change
between the moment the first register is read and the moment the last one is [R14]. When that happens, the
reconstructed value is neither the old value nor the new one; it is a chimera assembled from the high half
of one and the low half of another, and it can be wildly, nonsensically wrong — a counter that appears to
jump backward by a billion, a temperature that reads as a number no thermometer could produce.

Sometimes the torn read is obvious, a value so absurd that a range check catches it, and those are the
lucky cases. The dangerous ones are the torn reads that land inside the plausible range, where the high
half changed but the resulting number is still a number the machine could really produce. The model reads a
plausible-but-false value and reasons over it in perfect good faith. As with clock skew, no amount of model
intelligence recovers from this, because the input carries no evidence of its own corruption. The defense
lives in the read path: use the protocol features that exist for atomic multi-register access where the
device supports them, read the value twice and confirm it is stable before trusting it, or read a
transaction counter the device increments so a change mid-read can be detected after the fact. And where
none of that is possible, the frame should carry a confidence flag on any multi-register value, so the
model knows it is standing on a value that could have torn and can weigh it accordingly. The general
pattern is now visible and will repeat for the rest of the chapter: the failure is in the world, the model
cannot see it from the value alone, and the fix is to make the frame honest about what it does not know.

## Sticky values and silent substitution: the sensor that stopped telling the truth

A sensor that fails by going silent is a merciful sensor, because silence is detectable. The cruel failure
is the sensor that keeps reporting — that freezes at its last good value, or gets quietly swapped for a
substitute, or returns a default the device manufacturer chose, all while the data path downstream sees an
unbroken stream of plausible numbers. The earlier chapters called this the flatlined sensor, and it is
worth returning to here as a failure mode in its own right because it is the archetype of the physical
world's favorite trick: presenting a dead input as a live one.

A stuck sensor reads a constant. If the true value happens to be near that constant for a while, nothing
looks wrong, and by the time the true value has moved far from the stuck reading, the model has been
confidently reporting a stale number for hours. The model cannot distinguish a genuinely stable process
from a genuinely dead sensor from the value alone, because both produce a flat line. Silent substitution is
worse still: a maintenance action swaps a failed sensor's feed for a nearby one, or a redundant channel
fails over without announcing it, and now the tag the model trusts to mean one thing is carrying the value
of another. The number is real, the tag is familiar, and the mapping between them has silently changed
underneath the model. This is not a hypothetical; the loss of a spacecraft to a units mismatch, discussed
below, was in part a story about a value that meant one thing to the system reading it and another to the
system that produced it [R84], and the plant-floor version — a tag that quietly changes what it measures — is
the same failure wearing overalls.

The defenses here are older than any language model and belong to the instrumentation discipline: liveness
checks that flag a value that has not moved beyond its noise band for longer than physically plausible,
range and rate-of-change limits that catch a value doing something a real process cannot, and cross-checks
against a redundant or a physically related measurement. What the language-model deployment adds is the
requirement that the *results* of those checks reach the model as flags in the frame, not just as alarms on
a screen a human might see. A frame that says "inlet temperature 84 °C, this reading has not changed since
08-20, treat as suspect" gives the model somewhere honest to stand — it can report the reading with the flag
attached, or abstain, or escalate. A frame that says only "inlet temperature 84 °C" has laundered a dead
sensor into a live fact, and the model will faithfully repeat the fact.

## Sentinels, saturation, and rollover: real numbers that are not measurements

Closely related to the stuck sensor is a family of failures where the register holds a real, in-range number
that is not a measurement at all but a code — and the model, seeing a number, treats it as one. The most
common is the sentinel value: many devices report a special number to mean "no data," "sensor fault," or
"not configured," and because the value has to fit in the same register as a real reading, it is often
something like 65535, or -9999, or 0, or the maximum the field can hold. To the instrument engineer who
chose it, that value screams "invalid." To a model reading the bare number, 65535 is just a large reading and
-9999 is just a very cold one, and the model will confidently reason about a temperature of minus nine
thousand degrees or a pressure at the top of the scale as if the machine had really reported it. The sentinel
was designed to be unmistakable to a human who knows the convention and is invisible to a consumer who does
not, and a language model is exactly such a consumer unless the frame tells it otherwise.

Saturation is the same failure without the special code. An analog input that has railed — driven past the top
or bottom of its measurable range — reports the range limit, not the true value, and it keeps reporting that
limit no matter how much further the real quantity moves. A pressure that reads exactly at the transmitter's
maximum for an hour is far more likely to be a railed sensor than a process genuinely pinned at the top of
the scale, but the number is a perfectly plausible pressure and the model has no way to tell a real maximum
from a saturated one. Rollover is the mirror image at the other end of the arithmetic: a counter that reaches
the top of its register wraps to zero, so a totalizer or a runtime counter appears to jump backward by its
full range in a single step, and a model computing a rate from two readings across the wrap will compute an
enormous negative rate that never happened. None of these — sentinel, saturation, rollover — is a sensor
lying by accident; they are the expected, documented behavior of the value domain, and they catch the model
precisely because they are numbers rather than errors.

The defenses are the instrument engineer's, promoted into the frame. The read path must know each tag's
sentinel conventions and its range, and it must translate a sentinel into an explicit "no data" flag rather
than passing the raw code through as a value; it must flag a reading sitting at a range limit as possibly
saturated rather than trusting it; and it must handle counter rollover in the arithmetic that computes rates,
so that a wrap becomes a known discontinuity rather than a spurious negative. What the model needs is not the
raw register but the interpreted value with its status: "flow: NO DATA (sensor fault code)," "pressure: 10.0
bar, AT RANGE LIMIT — possibly saturated," "throughput rate: unavailable across counter rollover." A frame
that delivers those is a frame the model can reason about honestly. A frame that delivers 65535 has handed the
model a number and dared it to be wrong, and it will oblige.

## Byte order and the endianness trap

One more value-domain trap deserves a paragraph, because it is both common and specific to the industrial
world: byte and word order. A multi-register value has to be assembled from its pieces, and there is no
universal agreement on the order — big-endian, little-endian, and the notorious word-swapped orders where the
bytes within each register follow one convention and the registers follow another. Get the order wrong and the
value is not garbage in an obvious way; it is a different, plausible number. A float assembled with swapped
words can land squarely in the operating range while being completely unrelated to the true value, so the
model sees a believable reading that corresponds to nothing physical. This is a configuration failure in the
read path rather than a physical failure in the world, but it belongs in this chapter because its signature is
identical to the others: a plausible input the model cannot recognize as false. The defense is to pin and test
the byte order for every multi-register value against a known reference reading at commissioning, and to treat
any float or long that reads implausibly — but not so implausibly that a range check catches it — as an
endianness suspect before it is a process anomaly.

## Wrong engineering units: the confident answer in the wrong dimension

Of all the failures the physical world invents, the wrong-units failure is the one that most reliably
produces a confident, precise, catastrophically wrong answer, because units are exactly the kind of
context that gets lost in the seams between systems. A register holds a raw integer; somewhere a scaling
factor turns it into engineering units; and if the scaling factor the read path applies disagrees with the
scaling factor the device intended — a factor of ten, a Celsius-versus-Fahrenheit, a gauge-versus-absolute
pressure, a per-second-versus-per-minute rate — then every number the model sees is off by that factor, and
the model reasons flawlessly in the wrong dimension. The most expensive single lesson in the public record
about units is the loss of the Mars Climate Orbiter, where one system produced a quantity in
pound-force-seconds while another consumed it as newton-seconds, and the mismatch, undetected because each
number was individually plausible, sent the spacecraft into the atmosphere instead of orbit [R84]. The
plant floor produces smaller versions of that failure constantly, and a language model is an efficient
amplifier of them, because it will take the wrongly-scaled number and produce a fluent, authoritative
diagnosis built entirely on the wrong magnitude.

Units are dangerous specifically because they are invisible in the value. The number 90 is a perfectly good
temperature in Celsius and a perfectly good temperature in Fahrenheit and they are forty degrees apart in
reality; nothing in the digits tells you which. A model that has been told, somewhere in its context, that
the plant runs in Celsius will confidently interpret a Fahrenheit reading as a dangerous overtemperature or
a Celsius reading as benign, and it has no way to know which mistake it is making, because the unit was
never in the frame. The defense is to make units a first-class part of the schema — every value that enters
a frame carries its unit explicitly, end to end, from the tag definition through the scaling to the rendered
text — so that a unit mismatch becomes a detectable inconsistency rather than an invisible offset. This is
one of the strongest arguments for the closed, typed output contract the measurement chapter insisted on:
a schema that names units cannot silently drop them, and a read path that carries units cannot silently
apply the wrong scale without the mismatch showing up somewhere it can be caught.

## Maintenance overrides and forced points: the value a human set on purpose

Every one of the failures so far has been an accident of the physical world. This one is deliberate, which
is what makes it so easy to forget. In the course of maintenance, technicians routinely *force* points:
they override a sensor reading, hold a valve in a fixed position, disable an interlock, or inject a test
value, all as legitimate parts of doing work on a running or partly-running system. A forced point is a
value a human set on purpose, and while it is forced it is not measuring anything — it is whatever the
technician typed. This is normal, necessary, and correct plant practice. It is also a landmine for any
automated system that reads the value as if it were a measurement, because the forced value looks exactly
like a real one: same register, same format, same plausible number.

A model that reads a forced point as truth can go wrong in both directions. It can be reassured by a forced
"normal" value while the real condition behind it is anything at all — the sensor was forced to a safe
reading precisely so that work could proceed without tripping alarms, and the model, reading the forced
value, reports health that no one is actually observing. Or it can be alarmed by a forced test value that a
technician injected to check a downstream response, and it can escalate a crisis that exists only in the
override. Either way the model is confidently wrong about a value that was never a measurement, and it has
no way to know, because "this point is forced" is a fact about the plant's maintenance state that lives in
a completely different place from the value itself — if it is recorded anywhere at all.

The defense requires the read path to know the plant's maintenance state and carry it into the frame. Where
the control system tracks forced and overridden points — and good ones do — that status belongs in the frame
next to the value, so the model sees "valve position 45%, FORCED by maintenance" rather than a bare 45%.
Where the maintenance state is not tracked digitally, which in brownfield is common, the honest response is
to widen abstention during known maintenance windows and to treat the model's outputs as advisory-only
whenever a work order is open on the equipment it watches. A model that keeps confidently diagnosing a
machine that three technicians are actively rebuilding is a model that has not been told the machine is on
the bench, and telling it — through the frame, through the maintenance calendar, through the change-management
system — is the fix. This is the change-management discipline every plant already runs, extended to include
the automated reader as one more consumer that must be told when reality is under construction.

## The map that went stale: when a register changes meaning

Every failure so far assumes the mapping between registers and reality is fixed and only the values move. The
quieter, more corrosive failure is the one where the *mapping itself* changes and the model is never told. A
firmware update reorders a device's registers, or repurposes a spare, or changes a scaling default. A
configuration change repoints a tag. A device is replaced with a newer model that is compatible in every
respect except that register 40012 now means outlet pressure where it used to mean inlet temperature. The
values keep flowing, every one of them plausible, and the model keeps reading them against a map that is now
wrong — reporting pressures as temperatures with total confidence because the tag it trusts has silently
changed what it points at. This is the silent-substitution failure from earlier promoted to the level of the
whole schema, and it is especially dangerous because it survives every value-level sanity check: the numbers
are individually fine, the frame is well-formed, and only the meaning has rotted.

Map drift is a change-management failure, and it is defended against the way plants defend against any
undocumented change: by treating the register map, the tag definitions, the scaling factors, and the firmware
versions as controlled artifacts that cannot change without the change reaching everyone who depends on
them — including the automated reader. Practically, the frame should be able to assert the provenance of its
own schema: a version stamp on the tag map, a firmware version read back from the device where it is
available, so that a mismatch between the map the model was built against and the device it is actually reading
becomes a detectable, refusable condition rather than an invisible reinterpretation. A model that can read
"this device is running firmware the tag map has not been validated against" can abstain and escalate; a model
handed values against a map no one confirmed is still valid will confidently narrate a machine it is no longer
actually looking at. The commissioning discipline of validating the map against known readings, discussed
under endianness above, is not a one-time act — it is a thing that has to be re-run every time the equipment or
its configuration changes, and the deployment has to know when that has happened.

## Power events mid-inference: the half-written world

Software written for data centers assumes the computer stays on, because in a data center, with its
generators and its redundant feeds, it very nearly does. A model deployed next to a machine does not get
that assumption for free, and the failure it must survive is the hard power cut in the middle of doing
something. The author's own lab, which runs models continuously on owned hardware the way this book
proposes a plant should, lost building power twice in one week, and the two recoveries are a compact lesson
in what a power event actually breaks. The first crash cost about ninety minutes of recovery work; the
second, two days later, cost about twenty-five, and the entire difference was that the first crash had been
turned into engineering rather than treated as a bad day `[LAB: PROJECT-LOG 2026-08-22 and 2026-08-24 —
power-loss recoveries #1 and #2]`.

The damage path is worth walking because each stage is a decision you can make ahead of time. Storage is
hurt first and lies about it: a journaling filesystem recovers to a consistent state by design, but a
filesystem without journaling — including the network-share formats mounted through compatibility bridges
that are common in mixed environments — can be silently corrupted by a mid-write power loss and need offline
repair before it will even mount. The lab's first crash spent most of its ninety minutes there, and the
fix was boring and total: repair the mount configuration so every data volume comes back automatically and
consistently, after which the second crash cost zero minutes on storage `[LAB: PROJECT-LOG 2026-08-24 —
"the crash-#1 fixes held; zero filesystem work"]`. Services resurrect in the wrong order, or resurrect when
told not to: the second crash surfaced a service that had been explicitly disabled coming back anyway,
because other services declared it as a dependency and dependency pulls ignore the disabled flag, and it
occupied a GPU that other work was supposed to own. The durable fix was not a stronger off switch but a
condition — the service now checks for a hold-marker file and refuses to start while the marker exists, an
interlock rather than a request `[LAB: PROJECT-LOG 2026-08-24 — dependency pull vs. disablement;
condition-gated unit fix]`. The general rule that falls out of both crashes is one a plant engineer already
lives by: on shared hardware, "stopped" enforced by intention decays, while "stopped" enforced by a
condition survives reboots, dependency graphs, and colleagues.

For the model specifically, a power event mid-inference means an interaction that was in flight is simply
gone — the answer that was half-generated will never arrive, the request that was in the queue was never
served — and the surrounding system must treat that absence as a *missing* answer that triggers the safe
fallback, exactly as the measurement chapter insisted a failed request must never be scored as a wrong one.
A deployment that treats "the model went dark" as equivalent to "the model said everything is fine" has
built a system that reports health most reliably at the exact moment it has failed, which is the worst
possible correlation. The safe state on loss of the model is the state the machine would be in with no model
at all, reached automatically, and a power event is the test of whether you actually built that or only
believed you had.

## Alarm floods: the model that joins the noise it was meant to cut

The plant floor already has a well-studied failure of its human-facing information systems, and a language
model can walk straight into it or, deployed carelessly, make it worse. The failure is the alarm flood: a
disturbance trips one condition, which trips several more, and within seconds the operator is buried under
more alarms than any human can read, at which point the alarm system has stopped conveying information and
started conveying panic. The process industries codified the response to this in the alarm-management
standard ANSI/ISA-18.2, whose whole premise is that an alarm is only useful if it is actionable, prioritized,
and rare enough to be read — that the value of an alarm system is destroyed by its own volume past a
threshold [R32]. An operator in an alarm flood does not read faster; they start ignoring, and the one alarm
that mattered drowns with the rest.

A language model interacts with this failure in two ways, and both must be designed for. The first is that a
model producing outputs — advisories, diagnoses, escalations — is itself a source that can flood. A model that
emits a message on every frame, or that escalates every ambiguous reading, does not augment the operator; it
becomes another screen full of noise competing for the attention the standard is trying to protect, and the
crew learns to ignore it exactly as they learn to ignore a chattering alarm. A machine-facing model must
therefore be rate-aware and priority-aware in its own right: it must be built to stay silent when it has
nothing actionable to add, to consolidate rather than multiply during a disturbance, and to respect the same
prioritization the alarm system uses so that its outputs slot into the operator's attention budget rather
than blowing it. The second interaction is the opportunity: a model that can read the flood — that can take a
hundred simultaneous alarms and identify the probable root cause among them — is genuinely valuable, because
root-cause identification during a flood is exactly the cognitive task humans do worst under that load. But
that value is only realized if the model has been measured on real floods, including floods where the root
cause is not the first or loudest alarm, and if it abstains honestly when the flood is ambiguous rather than
confidently naming a root cause to reduce the operator's discomfort. A model that always names a cause during
a flood, right or wrong, is not cutting the noise; it is adding a confident voice to it, which is worse.

## Crying wolf once: the trust you cannot re-earn

Every failure in this chapter has a technical cost, but they share a second cost that is not technical and
is larger, and it is the failure that pure-software thinking most completely misses. When a model produces a
confident wrong answer to a machine operator — names the wrong bearing, raises a false alarm, misreads a
forced point as a crisis — it does not just waste that one interaction. It spends trust, and trust does not
refill on the same schedule it drains. The human-factors literature has a precise name for what happens
next: *disuse*, the rejection of an automated aid because it has cried wolf, and it sits alongside its twin,
*misuse*, the over-reliance on an aid that has not yet been caught being wrong. Parasuraman and Riley's
study of how people actually use automation makes the case that a system's real-world value is governed not
by its accuracy in the abstract but by whether operators calibrate their trust to it correctly, and that a
few salient false alarms can drive trust below the point where the aid is used at all — after which even its
correct outputs are ignored [R71]. An operator who has been burned once by a confident wrong answer does not
carefully re-weight their prior; they stop looking at the tool, and the tool's subsequent accuracy is
irrelevant because no one is reading it.

Lisanne Bainbridge's older observation, the "ironies of automation," sharpens the point and explains why
the failure compounds [R72]. Automating the easy cases leaves humans responsible for exactly the hard,
rare, high-stakes cases the automation could not handle — while simultaneously eroding the humans' practice
at handling them, because the automation took away the routine cases on which skill is maintained. A machine
model that handles the ordinary frames and cries wolf on the extraordinary ones is the pure form of this
irony: it deskills the operator on the common case and then fails them on the rare case, and the operator,
having lost both trust and practice, is worse off than if the model had never been deployed. This is why the
confident-wrong rate from the previous chapter is the metric that matters more than accuracy, and why
abstention is worth more than a marginal correct answer. The economics are not symmetric and they are not
even linear: the cost of a confident wrong answer includes the discounted future value of every correct
answer the operator will now ignore, and that is a cost no accuracy number captures. A tool that is trusted
and eighty percent accurate helps more than a tool that is ignored and ninety-five percent accurate, because
help delivered to no one is not help.

## Tight coupling: when the model becomes a path

There is a systemic version of these failures worth naming before the chapter closes, because it governs how
badly any individual failure can propagate. Charles Perrow's analysis of complex, tightly-coupled systems
argues that accidents become *normal* — expected, structural, not merely bad luck — when a system combines
interactive complexity, so that failures interact in ways no one anticipated, with tight coupling, so that a
failure propagates faster than anyone can intervene [R83]. A machine is often already such a system, and
inserting a language model into it can, if done carelessly, add both properties at once: interactive
complexity, because the model's behavior depends on a vast opaque context in ways that are hard to predict,
and tight coupling, because a model wired to act closes a loop that a human in the loop would have kept open
and slow.

The design implication is the through-line of this whole book restated as a systems principle: the model's
job is to be a source of information and, at most, of suggestion, and the coupling between its output and the
machine's action must stay loose enough that a human, or an independent interlock, sits in the path where the
consequences are serious. A model that suggests is a loosely-coupled component; its worst failure is a wasted
human glance. A model that acts is a tightly-coupled component, and its worst failure is bounded only by what
its actions can reach, which is why the previous chapter's blast-radius test exists and why the next
chapter's field guide refuses control authority that cannot be independently certified. The physical world
invents enough failures on its own; a deployment's job is to avoid inventing a new coupling path through which
those failures can propagate faster than anyone can stop them.

## Labeling what is measured and what is not

A chapter that catalogs failures owes the reader honesty about which of its claims are measured and which are
received wisdom, because the whole book turns on that distinction. The power-loss recoveries and their causes
are the author's own directly logged incidents on the authoring apparatus, cited to the project record by
date, and the numbers — ninety minutes, twenty-five minutes, zero minutes on storage the second time — are
real observations that will differ on other hardware but whose mechanism any operator can re-derive. The
run-to-run scoring swing and the quantization effects referenced from the previous chapter are likewise the
author's measured observations. The rest of this chapter — the clock-skew splice, the torn read landing in
range, the forced point read as truth, the alarm-flood dynamics, the trust that does not re-earn — is
grounded in the published references cited and in standard instrumentation and human-factors practice, and
where the author has seen a version of one of these on real equipment but has not staged and measured it for
this book, it is offered as illustration of a documented mechanism, not as a fresh measured finding. That
line matters: a book about honest measurement cannot smuggle in unmeasured plant anecdotes wearing the
authority of data, and the reader is entitled to know that the mechanisms here are well-established while the
specific plant stories that make them vivid are, unless cited to the lab record, illustrations rather than
results.

## The common shape, and the way out

Step back from the catalog and the shape is unmistakable. Clock skew, torn reads, stuck sensors, silent
substitution, wrong units, forced points, power events, alarm floods — every one of them is a case where the
model's input was false and the input carried no evidence of its own falseness, so that a correct model
reasoning correctly produced a wrong answer. The failures are not in the model, and they are not, mostly, in
the code; they are in the seam between the physical world and its digital rendering, and they are invisible at
exactly the layer the model reads. This is why "get a better model" is almost always the wrong response to a
machine-deployment failure, and why the industry's instinct to reach for a larger network solves the wrong
problem: a larger network reasons more impressively over the same false premise and arrives at a more
convincing wrong answer.

The way out has been the same in every section, which is how you know it is a principle rather than a
collection of tips. Make the frame carry its own trustworthiness. Every value arrives with its timestamp, its
units, its liveness, its forced-status, and its confidence, so that a value which cannot vouch for itself
announces the fact, and the model — through the abstention machinery of the earlier chapters — can refuse a
frame it cannot trust rather than confidently reasoning over it. The model's intelligence is not the defense;
the frame's honesty is, and the model's trained willingness to stand on that honesty and say "this reading
cannot be trusted, verify at the gauge" is the single most valuable behavior it can have next to a machine.
The physical world will keep inventing failures faster than any model can be taught to reason around them.
The only durable response is to stop asking the model to reason around them and start building frames that
tell the model when the world has lied. The next chapter turns that principle, and everything before it, into
a checklist you can run.

---

*The lab incidents cited above are the author's own logged observations on an AMD Threadripper 9970X
workstation running models continuously on owned hardware; recovery times will differ elsewhere and the
reproducible content is the mechanism, not the minutes. Mechanisms not cited to the lab record are grounded
in the referenced literature and standard practice and are offered as illustration rather than as fresh
measured findings.*
