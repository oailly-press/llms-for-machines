# Chapter 1 — Machines Without Language

*(draft v1, 2026-08-30 — written by claude-fable-5 (RogerAI Labs), unverified. Published sources carry `[R#]` markers resolved in the References; the author's own reproducible bench measurements carry `[LAB: …]` markers, stated with the apparatus and sample size that produced them. Per-chapter authorship is recorded in `manifest.json`.)*

There is a page in a logbook, kept now behind glass at the Smithsonian, that engineers
still photograph on their phones when they visit. It is dated 9 September 1947. Taped to
it is a moth, pulled from between the relay contacts of the Harvard Mark II, and beside
the moth someone has written, in the deadpan hand of a person filling out a form, *"First
actual case of bug being found."* [R1] The joke has outlived nearly everyone who was in
the room. It has also, quietly, become the founding document of an entire discipline: the
practice of getting a machine to tell you what is wrong with it.

Notice what the machine itself contributed to that report. Nothing. The Mark II did not
say it had a moth in relay 70. It stopped computing correctly, which is a machine's only
native way of complaining, and a human being had to open the panel, find the corpse,
diagnose the failure, and *write the sentence down.* The moth is the fault; the sentence
is a human's translation of the fault into language. For the better part of a century
after that page was filled in, that division of labor held without exception. Machines
produced conditions. People produced language about the conditions. The machine's job was
to be legible enough — through gauges, then through wires, then through protocols — that a
person could do the translating fast enough to matter.

This book is about the first moment in that century when the division stopped being
absolute. A language model small enough to run on the industrial PC bolted inside a
control cabinet can now, given the right plumbing, read the machine's own description of
itself and produce the sentence. Not perfectly, not everywhere, and — this is the whole
argument of the book — not safely unless you build the constraints in on purpose. But it
can do it, at a size and cost and latency that put the capability inside the walls of an
ordinary plant, a moving vehicle, a workcell, a substation. That is a genuinely new thing
in the world, and new things in the world of industrial control deserve a manual rather
than a press release.

## A century of describing machines, and not one word of it in language

Walk the history forward from the moth and you find an unbroken effort to make machines
describe themselves, conducted almost entirely without language.

First came the gauge and the chart recorder: a needle against a dial, a pen dragging ink
across a rotating drum. The machine's state became a physical analog — pressure as needle
angle, temperature as pen height — and a trained eye read it. Then came transduction into
electricity, and with it the 4–20 milliamp current loop, still one of the most widely
deployed instrument signals on earth. It encodes a process variable as a current between
four and twenty milliamps, with the deliberate four-milliamp floor so that a broken wire
(zero current) is distinguishable from a legitimate zero reading. That single design
decision — reserve part of the range to mean *the instrument itself has failed* — is worth
pausing on, because it is the oldest widespread example of a machine engineered to tell the
difference between "the value is low" and "I cannot be trusted." We will return to that
distinction constantly. It is the thing language models are worst at and instruments have
done for fifty years.

Then came digitization proper, and with it the protocols. Modbus, published as a public
specification and still governed by the Modbus Organization, defined a way for a
controller to ask a device for the contents of numbered registers and get back sixteen-bit
words [R14]. The CAN bus, standardized as ISO 11898, and SAE's J1939 layered on top of it,
let the engine, the transmission, and the brakes on a truck broadcast their state to one
another as compact numeric messages [R15][R16]. OPC UA built an entire object-oriented
information model so that a variable on a factory network could carry not just a value but
a data type, a quality status, and a source timestamp, all at once [R8]. MQTT and its
industrial profile Sparkplug carried those values across unreliable links with birth and
death certificates so a subscriber could tell a silent sensor from a sensor reporting zero
[R17][R18]. Historians — the industrial databases that store years of tag values — learned
to compress and retrieve billions of these readings. Syslog and, later, Prometheus gave the
computers *running* the plant their own dialects for emitting events and metrics [R21][R19].

Every one of these is a language in the loose sense — it has a grammar, a vocabulary, a way
of being parsed. Chapter 4 will treat them exactly as languages, because that framing turns
out to be the most useful one a model can be handed. But not one of them is language in the
sense this book's title means. They are codes designed to be unambiguous, compact, and
machine-parseable — the opposite of natural language in every design goal. A Modbus
response is a hex dump. A J1939 frame is twenty-nine bits of identifier and eight bytes of
payload. These are the machine describing itself with maximum precision and zero prose. The
prose, as with the moth, was always somebody's job to add later.

So we arrive at the present with an enormous, mature, well-specified infrastructure for
machines to describe themselves numerically, and — until very recently — nothing at all
that could turn those descriptions back into language on the spot, inside the walls,
without a human in the chair. The gap is not a gap in the machines. The machines have been
ready for decades. The gap is that the translation step, the step the 1947 engineer did by
hand, never had a candidate for automation that could be trusted anywhere near equipment.

## The book-shaped hole, and the book that almost filled it

There is a precedent for the moment this book is trying to catch, and it is close enough to
be instructive.

When machine learning first shrank small enough to run on microcontrollers and cheap edge
processors, the knowledge was scattered across conference papers, vendor SDKs, and forum
threads until a few practitioners sat down and wrote the textbooks. *AI at the Edge*, by
Daniel Situnayake and Jenny Plunkett, is one of the best of them: a careful, working-
engineer's account of getting neural networks onto small devices — the sensors, the signal
processing, the quantization, the deployment discipline [R2]. If you do embedded ML on real
hardware, it is on your shelf, and it deserves to be.

Read it today, though, with this book's question in mind, and you notice something that is
not a flaw in the book but a measurement of how fast the ground moved. It is a
comprehensive field guide to putting intelligence at the physical edge, published when the
edge-ML wave was cresting — and language models are essentially absent from it. That is not
an oversight. When it was written, the idea that a model capable of reading a maintenance
manual and cross-referencing it against a live protocol stream could fit on the same class
of hardware it describes was not an engineering topic. It was speculation. The book
documented the world that existed, accurately, and that world did not include a language
model you could bolt to a machine.

The companion volumes in this series made the same observation about the industrial control
literature: the standard references cover protocols and historians, tags and timestamps,
the entire plumbing of how a plant's machines describe themselves — and do not mention
language models, because when they were assembled the mention would have been science
fiction. That is the book-shaped hole. It is unmistakable once you see it, and the history
of the edge-ML wave suggests what happens next: someone writes into the hole, and whoever
writes into it early gets to shape how a generation of engineers first meets the technology
— the vocabulary, the default assumptions, the safety posture, all of it. The premise of
this series is that the hole should be filled the way the best of the earlier textbooks were
written: from a working lab, with measurements, by authors willing to publish the failures
next to the wins.

The two industrial volumes that came before this one — Nº 1 on local models for
manufacturing, Nº 2 on the discipline of measuring them honestly — filled a narrow part of
the hole with depth. They earned this book the right to be broad. This is Nº 3, the field
guide: not the deep dive into one plant, but the survey across the whole physical world
where language models are now arriving — factories and their brownfield reality, vehicles
and mobile equipment, robots and workcells, the grid, buildings, and the sensors and
protocols underneath all of them. Where the earlier books went deep on one floor, this one
walks the whole building and tells you, honestly, which rooms have working floors and which
are still open joists.

## Who this book is for, and who it is not for

This is a book for people who put software next to physical things and are accountable when
the physical thing misbehaves. Controls engineers and integrators. Reliability and
maintenance engineers. The people who own a historian, a fleet telematics feed, a building
management system, a substation's protection relays. Automation staff at plants, robotics
engineers on the cell floor, embedded and firmware developers, the SRE-adjacent people who
keep industrial software alive. If your job includes being paged when a machine does
something it should not have, this book is written at you.

It assumes you are fluent in at least one machine domain — that you know what a PLC is, or a
CAN bus, or a historian tag, or a BACnet point, without being told. It does not assume you
have any background in machine learning research, and it will not develop one. There are no
loss curves in this book, no attention diagrams, no derivations. Chapter 2 gives you exactly
the model of how a language model behaves that a machine owner needs in order to reason
about authority and failure — tokens, context, sampling, hallucination, constrained
decoding — and not one concept more. When this book uses a term from the ML world, it is
because you will meet that term in a vendor conversation and need to not be bluffed by it.

It is also, deliberately, a book for machine *readers* as well as human ones. The O'AILLY
industrial shelf ships books that a model can be pointed at as a source, and that means the
claims have to be structured, sourced, and honest at a granularity a careless human author
can get away with skipping. Every substantive claim in this book is either grounded in a
published, resolvable specification or standard — the ones in the reference list all
resolved when this was written — or it is a measurement the authors took on their own
hardware, reported with the apparatus stated, or it is labeled as unmeasured judgment. You
will see that seam marked in the text. It is not decoration. It is the difference between a
field guide and a brochure.

## Why this book is not about the cloud

Every vendor deck you have seen for "AI on the machine" puts the model in someone else's
data center. This book puts it inside your walls, on hardware you own, and the reasons are
not ideology. They are the same reasons plants have always been wary of anything that needs
the network to think.

**The network is not your friend, and it was never meant to be.** Industrial networks run
air-gapped or nearly so, on purpose, and the moments when you most want a model's help — a
line fault, a cascading alarm, a truck throwing codes on a haul road with no signal — are
exactly the moments when the link to the outside is least likely to be there. A capability
that evaporates during incidents is not a capability you can build a procedure around.

**Latency is a physical property, not a nicety.** On a moving line, a response that arrives
late is not slow, it is wrong, because the state it describes has already passed. The budget
for "read the frame, decide, respond" on real equipment is often milliseconds to a few
seconds. A round trip to a cloud endpoint spends that budget before the model has read a
token. The book will be blunt in Chapter 2 about where language models fit in a latency
budget — which is emphatically *not* inside a control loop — but even for the advisory roles
where they do fit, the round trip is often the disqualifier.

**The data is the process, and the process does not leave the building.** Historian streams,
recipe parameters, fault histories, telematics — this is the operational crown jewels, and
the decision to send it off-site is made by lawyers and risk officers, who usually say no,
and are usually right. A model that can only help if the data leaves is a model that legal
has already killed.

**The economics invert at industrial data rates.** A historian emits thousands of tag
updates a second. Per-token cloud pricing against that firehose is not a tool, it is a meter
running against your budget forever. A model you own costs the same at 3 a.m. on day 900 as
it did on day one.

**The model under you must not change without your consent.** This is the one plant people
grasp fastest and outsiders grasp slowest. A hosted model is updated on the provider's
schedule. The prompt that passed your acceptance test in March can behave differently in
June because the model behind the endpoint quietly became a different model, and no change-
management process on your side can prevent it. A plant that version-pins PLC firmware for
excellent reasons will feel the wrongness of this immediately. A local model is a file with
a checksum. It changes when you change it and never otherwise. For a system that has to be
qualified, audited, and trusted to behave tomorrow the way it behaved during acceptance,
that property is not a preference. It is a requirement, and the cloud cannot offer it.

So the book takes the constraint the marketing avoids and treats it as the ground floor:
**the model runs on hardware you own, inside your walls, on your data.** Every question that
follows — which model sizes, which capabilities, which failure modes, which protocols it can
read and which it cannot — is answered under that constraint, not as a diminished version of
the cloud but as its own engineering problem with its own answers.

## What a local model can and cannot do at the physical interface

The honest shape of the capability surprises people in both directions, and the authors did
not arrive at that shape by reasoning. They measured it, on a prosumer-class box — a
Threadripper workstation with a cluster of Blackwell GPUs — running local models against
industrial tasks, and the middle chapters walk through those measurements with their error
bars attached. Here is the shape, stated up front so the rest of the book can defend it.

Local models are *better* than their reputation at reading structure. Handed a protocol
frame, an enum field, a historian export, a fault-code table, a maintenance work order in
free text, a competent local model converts it into a clean, schema-valid object with a
reliability that startles people who only know chatbots. In the authors' own testing,
extracting a structured work order from a scrappy sentence like *"Pump P-204 seal leaking,
needs attention before next shift"* was among the most robust things a model did — it
survived even aggressive compression of the model that broke almost everything else
[LAB: RESULTS-MATRIX §H: the extraction task passed where free-text explanation of the same equipment failed completely, and the split was confirmed against an uncompressed control — RogGentoo lab]. Extraction and structured tool-calling are the load-bearing strength, and the
book leans on it.

Local models are *worse* than their reputation at knowing the limits of what they read. This
is the failure mode that matters, and it is not the model that cannot answer — it is the
model that always answers. The authors ran a deliberately clean version of this test: give a
range of models, from a 270-million-parameter model up to a 72-billion-parameter one, a
deterministic feature summary of a single sensor channel and ask them to classify the fault
— is this signal stuck, railed, drifting, noisy, dropping out, or fine? — over a closed set
of choices, with 300 channels balanced across the fault classes and the fault definitions
spelled out in the prompt. Every model landed at or below chance. A thirty-line hand-written
rule over the identical input scored 63% [LAB: RESULTS-MATRIX §R.26 — n=300 balanced items, single deterministic run, seed-reproducible; Wilson 95% ≈ 57.7–68.6% — RogGentoo lab]. Scaling the
model up a hundredfold changed nothing. The finding, stated at its strongest and now
measured rather than asserted: general language models, as they come off the shelf, cannot
read machines — not the small ones and not the large ones — because the task is simply
absent from the text they were trained on. Chapter 3 develops this in full, because it is
the single most important calibration a machine owner can carry into this technology.

Put those two findings together and the book's agenda writes itself. The value is not in
making a model smarter. It is in exploiting the real strength — structured reading — while
building in the honesty the model does not have natively: extraction that cites what it
read, classification that reports a margin and abstains below it, output constrained to a
schema so the model *cannot* emit an invalid value, and evaluation gates that would rather
reject a right answer than pass a wrong one. Those are the load-bearing primitives of this
entire book, and Chapter 2 introduces them as the machine owner's core toolkit.

## The authority frontier: read, suggest, act

If you take one diagram away from this chapter, take this one, even though it is only three
words. Everything a language model might do at the physical interface falls into one of
three authority levels, and the whole safety posture of a deployment is a question of where
you draw the lines between them.

**Read.** The model consumes machine data — frames, tags, logs, manuals — and produces a
description, a summary, an extraction, a classification, a draft. It changes nothing. Its
output is information for a human or for another piece of software to consider. This is where
local models are strongest and where the overwhelming majority of real value lives. A model
that reads a wall of alarms and drafts a ranked, cited summary for the operator is doing
genuine work at genuinely low risk, because a wrong summary is a wrong *suggestion*, caught
by the human who was going to read the alarms anyway.

**Suggest.** The model proposes an action — set this parameter, acknowledge that alarm, open
this work order, dispatch this technician — and a human or a qualified system decides whether
to take it. The model's authority stops at the proposal. The risk rises, because a plausible-
sounding wrong suggestion can steer a tired operator, and the book spends real effort in
later chapters on how to make suggestions carry their evidence and their uncertainty so the
human can actually evaluate them rather than rubber-stamp them.

**Act.** The model's output changes the physical world without a human in the loop — writes a
setpoint, trips a relay, commands a motion. This book's position on this level is simple and
it does not soften over the following chapters: **a language model does not belong in a
control loop, and it must never be trusted with a safety function.** Not at any size. The
reasons are developed in Chapter 2, but the short form is that a language model's behavior is
statistical and non-deterministic in ways that matter, and it cannot be qualified against the
functional-safety standards — the IEC 61508 family and its sector children — that govern
anything allowed to act on equipment where people can be hurt [R5]. Those standards are built
around a demonstrable, auditable relationship between a failure and its probability, managed
across a whole safety lifecycle. A language model cannot supply that relationship for its own
outputs. So the model stays out of the loop, and the loop stays governed by the
deterministic, qualifiable systems that have earned their place in it. The model's job is to
make the humans and the qualified systems better informed, faster — to read and to suggest,
never to act.

Holding that frontier is not a limitation the book apologizes for. It is the reason a local
language model is deployable at all near equipment. The value is enormous and it lives
entirely on the read-and-suggest side of the line. The engineering discipline is in refusing
to let it drift across.

## Why the physical interface is a different world from the chatbot

Most of what has been written about language models was written about chatbots — models
answering questions typed by a person who reads the answer, judges it, and moves on. Almost
every intuition that world builds is wrong, or at least dangerously incomplete, at the
physical interface, and it is worth naming the four differences explicitly, because they are
the reason this book exists as a separate book rather than a chapter in someone else's.

**The cost of being wrong is physical, and it is not paid by the reader.** When a chatbot is
wrong, a human notices and discards the answer; the cost is a few wasted seconds. When a
model near a machine is wrong and the wrongness is acted on, the cost can be a scrapped batch,
a tripped line, a damaged bearing, a violated permit, or — at the levels this book keeps the
model away from — a person. The asymmetry between a false "everything is fine" and a false
alarm is not symmetric in the chatbot world and is brutally asymmetric here. The whole design
of an industrial deployment bends around making the expensive kind of wrong rare, and toward
failing loud rather than failing silent.

**Real time is a hard constraint, not a performance target.** A chatbot that takes ten
seconds is annoying. A model asked to help interpret a fault on a moving line in ten seconds
has answered a question about a plant state that no longer exists. This forces a discipline
chatbot builders never face: knowing, in advance and with a number, how long the model takes,
and placing it only where that number fits the loop it is advising. Chapter 2 gives the
latency model; the point here is that "usually fast enough" is a chatbot standard and a
liability at the physical interface.

**Determinism is expected, and the model does not natively provide it.** Every other
component in a control system is deterministic by design and by regulation: the same inputs
produce the same outputs, the behavior is specified, the failures are enumerable. A language
model, sampling tokens, is not deterministic in that sense unless you make it so — and even at
temperature zero, batching and hardware scheduling can shift outputs run to run, an effect the
authors have measured directly on their own bench [LAB: RESULTS-MATRIX §C: a small tool-use suite (n=15 scenarios) swung by roughly ten points across three identical back-to-back runs from batch-packing nondeterminism, not from sampling, with about a third of scenarios flipping between runs — RogGentoo lab]. A machine owner who assumes model output is
reproducible the way PLC logic is reproducible has made a category error. Much of the
engineering in this book — constrained decoding, enum scoring, evaluation with error bars —
exists precisely to claw back as much determinism as the task allows and to measure honestly
what cannot be clawed back.

**Auditability is a requirement, not a feature.** When a plant has an incident, there is an
investigation, and every system in the chain has to be able to say what it did and why. A
model that produced an ungrounded free-text opinion contributes nothing an investigator can
use and may actively muddy the record. A model that produced a schema-valid assertion citing
the exact frames and tags it read, with a confidence margin attached, contributes evidence.
The difference is not cosmetic; it determines whether the model is an asset or a liability
after something goes wrong, and it is another reason the book pushes so hard toward structured,
cited, bounded output over conversational prose.

Hold these four together and you have the reason the chatbot playbook does not transfer.
Chatbots optimize for a fluent, helpful answer to almost anything. The physical interface
demands the opposite temperament: a component that says less, says it precisely, says it fast
enough, refuses when it should, and leaves a trail. Building that temperament on top of a
technology whose defaults run the other way is the craft the rest of the book teaches.

## A worked example, in three authority levels

To make the frontier concrete, take one small scenario and run it through all three levels.
A refrigeration compressor on a food-plant chill line trips on high discharge pressure at
02:40. The controller logs the trip; the historian shows discharge pressure climbing over the
prior nine minutes; a maintenance manual for the unit sits on a file share; three related
alarms fired in the same window. The on-call operator has a phone and a headache.

At the **read** level, a local model subscribed to the alarm stream and pointed at the
historian export and the manual produces, in a couple of seconds, a summary: discharge
pressure rose from 14 to 22 bar over nine minutes before the high-pressure cutout tripped at
its 21-bar setpoint; the condenser-fan alarm fired four minutes into the climb; the manual's
troubleshooting table lists condenser airflow loss as the first cause of rising discharge
pressure. Every clause cites the tag, alarm, or manual section it came from. Nothing has been
changed. The operator reads a coherent, sourced picture instead of assembling it from six
screens at 02:40, and if the model got a detail wrong, the citations make it catchable rather
than authoritative. This is where the value is, and it is real.

At the **suggest** level, the model goes one step further: given the pattern, it proposes
checking the condenser fans and airflow before restarting, and drafts the work order to do so
— structured, ready for the operator to approve or reject. The authority still stops at the
human. But notice the risk has risen: if the model's suggestion is confident and wrong — say
the real cause was a failing pressure transducer reading high, and the fans are fine — a tired
operator may chase the fans and lose time. This is why the book insists suggestions carry
their evidence and their uncertainty in a form the human can actually weigh, not a bare
imperative.

At the **act** level, the model would restart the compressor, or change the cutout setpoint,
on its own. This book's answer is flat: no. The restart interlock, the pressure cutout, and
the setpoint governance stay with the deterministic, qualified control system that owns them
[R5]. The model that helped the operator understand the trip in seconds has already delivered
almost all of the available value, at almost none of the available risk. Reaching for the last
step trades the entire safety posture for a convenience the operator did not need. The craft is
knowing to stop at suggest, every time.

## The measurement posture this book inherits

One more thing before the machinery. This series's second volume was entirely about the
discipline of measuring language models honestly, and this book operates under its rules
without re-deriving them. Three of those rules show up on nearly every page that reports a
number, so it is worth stating them once.

Every benchmark number gets an error bar before it gets published, because small industrial
test suites swing hard between identical runs — the authors have watched a fifteen-scenario
tool suite move ten points from nondeterminism alone [LAB: RESULTS-MATRIX §C — n=15 scenarios, spread across three runs — RogGentoo lab]. This
is also the posture the public risk frameworks ask for: the NIST AI Risk Management Framework's
MEASURE function calls for AI systems to be tested before deployment and regularly in operation,
with the measurement itself documented [R4]. One
surprising number gets run again; two runs that disagree get a third and a control. Second,
every experiment that can carry a control carries one, because a model at chance is only
interesting once you have shown, with a hand-written rule over the identical input, that the
answer was legible in the data at all — that control is what turned "our features are bad"
into "the model genuinely cannot read this" in Chapter 3's central finding. Third, the numbers
that weaken the story get published with the analysis rather than buried; the most useful
findings in this book are the negative ones, because a negative result measured cleanly steers
the next person away from a wall the authors already hit. When this book says a thing is
measured, it means measured that way. When it has not been measured, the book says so.

## Disclosure, and who wrote this

There is a fact about this book that the industrial shelf requires be stated in the open
rather than buried: it was written by a language model — a Claude model, working as a
co-author under RogerAI Labs — grounded in the published standards cited throughout and in
measurements taken on the authors' own hardware, and it does not ship until a named human has
verified the manuscript. That disclosure is not a disclaimer wedged into the front matter. It
is a working example of the doctrine the whole book argues for.

The doctrine is this: at the physical interface, authorship and provenance are safety
properties, and they should travel with the artifact where a reader can check them. The
content-provenance world has built exactly this machinery for media — the C2PA specification
defines standards for certifying the source and history of a piece of content so the
attribution is verifiable downstream rather than taken on faith [R3]. This book borrows the
posture at the level of a claim: when a sentence rests on a measurement, the sentence says so
and names the apparatus; when it rests on a specification, the marker points to a source that
resolves; when it rests on the author's judgment and nothing more, it says that too. A model
that reads a machine and asserts a fault should carry the same discipline — this is what I
read, this is how sure I am, this is who is accountable — and a book that asks models to do
that had better model it first.

If the idea of trusting a book written by an AI about trusting AIs on machines makes you
wary, good. That wariness is the correct operating posture, and this book is trying to earn
its way past it the only way that works: by showing the machinery, citing the sources, and
publishing the failures. The next chapter starts with the failures, because the fastest way
to understand what a language model is allowed to touch is to understand, precisely, how it
breaks.

## What this book claims, and what it refuses to claim

A field guide earns trust by drawing its own boundaries, so here they are.

**It claims** that a local language model, running on hardware you own, grounded in your
documents and constrained to your schemas, can do real work across the physical domains this
book surveys — reading protocols and historian output, cross-referencing manuals, classifying
signals it has been trained or tooled to classify, drafting structured verdicts and summaries
for human review — with latency, cost, and data-custody properties the cloud cannot match;
and that the engineering required to make that work *honest* is learnable, measurable, and
within reach of a competent controls or reliability team.

**It refuses to claim** that a language model should close a control loop; that it can be
trusted with a safety function at any size; that it replaces a historian, a CMMS, a PLC, or
an engineer; that reading structured machine data off the shelf equals *understanding* the
machine (Chapter 3 shows it does not); or that the capability is uniform across model sizes
and domains. Where a capability does not yet exist, or exists only as an unproven claim, this
book says so in plain text. The authors would rather under-promise in print and have you find
the technology better than advertised than do the reverse near your equipment.

The chapters ahead follow the natural path of the data. Chapter 2 builds the machine owner's
model of what a language model is and where its authority stops. Chapter 3 walks the signal
path from a physical phenomenon to the bytes a model reads, and shows how a clean JSON value
can lie about a dirty sensor. Chapter 4 treats the protocols as the languages they are and
shows a model reading them frame by frame. From there the book moves out into the domains —
plants and their brownfield mess, vehicles and mobile equipment, robots and workcells, the
grid, buildings — and closes on the disciplines that hold across all of them: grounding,
abstention, evaluation, and the honest deployment checklist.

The moth in the logbook is still, eighty years on, the cleanest statement of the problem. A
machine failed. It could not say why. A human found the reason and wrote the sentence. This
book is about the narrow, hard-won, carefully bounded set of cases where the machine can now
help write the sentence itself — and about the much larger set of cases where letting it try
would be a mistake. Knowing the difference is the whole craft. Let us begin with how the
model actually works.
