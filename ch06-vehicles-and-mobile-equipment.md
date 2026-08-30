# Chapter 6 — Vehicles and Mobile Equipment

*(draft v1, 2026-08-30 — written by claude-fable-5 (RogerAI Labs), unverified. Public
specifications and standards are cited as `[R#]` and resolve in the References; the
author's own runnable listing is labeled as reproducible apparatus; claims without either
are labeled unmeasured. This chapter states an unconditional boundary between diagnostic
assistance and vehicle control and does not cross it.)*

A truck is a plant that moves. A wheel loader is a workcell with a diesel engine and a
seat. An over-the-road tractor pulling forty tons is, from the point of view of a language
model, a rack of controllers on a serial bus with an operator who is simultaneously the
user of the model and the pilot of a safety-critical machine at highway speed. Everything
this book has said about reading machines applies to vehicles, and vehicles add two things
that change the engineering: a standardized diagnostic vocabulary that is unusually
friendly to a language model, and a set of physical constraints — the cab, the operator's
attention, the intermittent link, the safety of the moving machine — that are unusually
unforgiving of getting the deployment shape wrong.

The one-sentence thesis: **on a vehicle, the diagnostic bus hands a language model a
better-structured vocabulary than any plant historian, and the moving machine hands it a
harder set of physical constraints than any rack — so the model in the cab is a different
animal from the model in the datacenter, and confusing diagnosis with control is the one
mistake that is not allowed to happen even once.**

## The bus already speaks a controlled vocabulary

The good news first, because vehicles genuinely are easier than brownfield plants in one
important respect. Heavy-duty on-highway and off-highway equipment overwhelmingly speaks
SAE J1939, a family of standards layered on top of the Controller Area Network (CAN) [R15].
Where a brownfield plant's tags are a historical accident, J1939 is a *controlled
vocabulary by design*. Every measured quantity has a Suspect Parameter Number — an SPN —
that identifies it: SPN 110 is engine coolant temperature, SPN 100 is engine oil pressure,
SPN 190 is engine speed, SPN 84 is wheel-based vehicle speed. These numbers mean the same
thing on a truck from one maker and a truck from another, because they are defined in
J1939-71, the application layer that assigns the vocabulary [R24]. Messages are carried in
Parameter Group Numbers — PGNs — defined in J1939-21, which also specifies how messages
too large for a single CAN frame are broken up and reassembled [R25]. And diagnostics have
their own standard, J1939-73, which defines how a controller reports active and previously
active faults [R23].

This is a gift to a language model, and it is worth being precise about why. In a
brownfield plant, half the textualization effort goes into *establishing what a tag means*
— the decode tables, the lost mappings, the undocumented registers of Chapter 5. On a
J1939 vehicle, the meaning is standardized and published; the decode table is largely the
standard itself. The model can be handed a fault that is already named — "engine coolant
temperature, above normal, most severe" — rather than a raw register it has to guess about.
The inferential leash is short before you do any work, because SAE already did the naming.

The fault-reporting structure is the part most worth understanding, because it is where a
model reads a vehicle's health. A controller reports its currently active faults in a
message called DM1 (Diagnostic Message 1, PGN 65226), and its previously active,
now-inactive faults in DM2 [R23]. Each fault is a Diagnostic Trouble Code — a DTC — and a
DTC is not a free-form string. It is a precise packing of four fields into four bytes: the
SPN identifying *what* is wrong, the Failure Mode Identifier — the FMI — identifying *how*
it is wrong, an occurrence count, and a conversion-method bit. The FMI is a small
controlled vocabulary of its own, defined in J1939-73: FMI 0 is "above normal, most
severe," FMI 1 is "below normal, most severe," FMI 3 is "voltage shorted high," FMI 5 is
"open circuit," and so on across thirty-two codes [R23]. The DM1 also carries a lamp-status
byte — is the amber warning lamp on, the red stop lamp, the malfunction indicator — packed
two bits per lamp.

The upshot is that a vehicle's fault state is a compact, standardized, machine-readable
structure, and turning it into text a model can reason over is a decoding job with a
published spec rather than an archaeology job. That decoding is worth doing in code,
carefully, once, and the next section does it.

## Decoding a DM1 into a typed rendering

Here is a stdlib-only Python decoder that takes a raw DM1 payload and produces a typed,
model-ready rendering per the J1939-73 field packing [R23]. The transcript below it is real
output from running it on this box (Python 3.13, RogGentoo). The listing is deliberately
small; the point is not a production J1939 stack but the shape of the translation from
wire bytes to text a model should read.

```python
# j1939_dm1.py — decode a J1939 DM1 (active DTCs) payload into typed text.
SPN_NAMES = {                                   # subset of J1939-71
    84:  ("Wheel-Based Vehicle Speed", "km/h"), 91: ("Accelerator Pedal Pos 1", "%"),
    100: ("Engine Oil Pressure", "kPa"), 102: ("Intake Manifold #1 Pressure", "kPa"),
    110: ("Engine Coolant Temperature", "degC"), 158: ("Keyswitch Battery Potential", "V"),
    190: ("Engine Speed", "rpm"),
}
FMI_TEXT = {                                     # J1939-73 failure mode identifiers
    0: "above normal range - most severe", 1: "below normal range - most severe",
    2: "erratic / intermittent / incorrect", 3: "voltage above normal / shorted high",
    4: "voltage below normal / shorted low", 5: "current below normal / open circuit",
    6: "current above normal / grounded", 7: "mechanical system not responding",
    11: "root cause not known", 14: "special instructions",
    15: "above normal - least severe", 31: "condition exists",
}
LAMP = {0b00: "off", 0b01: "ON", 0b10: "reserved", 0b11: "n/a"}

def decode_dtc(b):                               # 4 bytes -> SPN, FMI, CM, OC
    spn = b[0] | (b[1] << 8) | ((b[2] >> 5) << 16)
    return spn, b[2] & 0x1F, (b[3] >> 7) & 1, b[3] & 0x7F

def render(hexstr):
    data = bytes.fromhex(hexstr.replace(" ", ""))
    if len(data) < 2:
        raise ValueError("DM1 too short for lamp status")
    lamps = {"malfunction": LAMP[(data[0] >> 6) & 3], "red_stop": LAMP[(data[0] >> 4) & 3],
             "amber_warning": LAMP[(data[0] >> 2) & 3], "protect": LAMP[data[0] & 3]}
    dtc_region = data[2:]
    if len(dtc_region) % 4:
        raise ValueError("truncated DTC: DTC region not a multiple of 4 bytes")
    out = ["SOURCE: PGN 65226 (DM1) active DTCs"]
    on = [k for k, v in lamps.items() if v == "ON"]
    out.append("lamps ON: " + (", ".join(on) if on else "none"))
    dtcs = [decode_dtc(dtc_region[i:i+4]) for i in range(0, len(dtc_region), 4)]
    if not dtcs or (len(dtcs) == 1 and dtcs[0][0] == 0 and dtcs[0][1] == 0):
        return "\n".join(out + ["dtc: none active"])
    for spn, fmi, cm, oc in dtcs:
        name, unit = SPN_NAMES.get(spn, ("UNKNOWN SPN - not in loaded J1939-71 map", ""))
        u = f" [{unit}]" if unit else ""
        out.append(f"dtc: SPN {spn} ({name}){u} | FMI {fmi} "
                   f"({FMI_TEXT.get(fmi,'FMI not in table')}) | count {oc}")
    return "\n".join(out)
```

A small `__main__` driver (omitted above) loops four constructed DM1 frames — a
coolant-hot-plus-oil-low fault with two lamps lit, an all-clear frame, a fault on an SPN the
map does not know, and a deliberately truncated frame — printing each `raw:` line and, when
`render` raises, a `DECODE ERROR (abstain, do not guess):` line instead of a decoded fault.
It produces, verbatim:

```text
=== coolant-hot + oil-low, amber+red lamp ===
raw: 14 FF 6E 00 00 01 64 00 01 05
SOURCE: PGN 65226 (DM1) active DTCs
lamps ON: red_stop, amber_warning
dtc: SPN 110 (Engine Coolant Temperature) [degC] | FMI 0 (above normal range - most
  severe) | count 1
dtc: SPN 100 (Engine Oil Pressure) [kPa] | FMI 1 (below normal range - most severe) | count 5

=== no active faults (all clear DM1) ===
raw: 00 FF 00 00 00 00
SOURCE: PGN 65226 (DM1) active DTCs
lamps ON: none
dtc: none active

=== unmapped SPN, open circuit ===
raw: 04 FF E7 03 05 02
SOURCE: PGN 65226 (DM1) active DTCs
lamps ON: amber_warning
dtc: SPN 999 (UNKNOWN SPN - not in loaded J1939-71 map) | FMI 5 (current below normal /
  open circuit) | count 2

=== truncated DTC region (should be rejected) ===
raw: 14 FF 6E 00 00
DECODE ERROR (abstain, do not guess): truncated DTC: DTC region not a multiple of 4 bytes
```

Four behaviors in that transcript carry the chapter. The first case shows the payoff: two
raw four-byte codes become two named, unit-bearing, severity-graded faults, with the lamp
state that tells you the operator is already seeing a red stop lamp. A model handed this
text reasons about *coolant temperature high and oil pressure low, most severe, red stop
lamp lit* — a coherent picture that points at, say, a coolant-loss event dragging oil
behavior with it — instead of reasoning about `14 FF 6E 00 00 01 64 00 01 05`, which it
would have to hallucinate its way through. The second case matters more than it looks: an
all-clear DM1 is not silence, it is a positive statement that the controller reports no
active faults, and rendering it explicitly stops the model from treating "no faults" as
"no data." The third case is the vehicle version of Chapter 5's unmapped tag: an SPN the
loaded map does not know is rendered as *unknown*, not guessed, so the model treats it as a
gap and a human — or a fuller SPN table — resolves it. The fourth case is the one that
matters most for safety: a truncated frame is *rejected*, not decoded on a best-effort
basis, because a partially read CAN frame that is decoded anyway is how a model comes to
report a fault that does not exist or miss one that does. On a vehicle, "abstain and do not
guess" is not politeness; it is the difference between a trustworthy dashboard and a lying
one.

There is a subtlety worth flagging for anyone who builds the real version. The FMI and the
SPN are a controlled vocabulary, but the *interpretation* of a specific SPN/FMI pair on a
specific engine can be manufacturer-refined — J1939 leaves room for maker-specific SPNs in
a proprietary range, and a given code's real-world meaning is best confirmed against the
engine maker's service literature. The decoder should say what the standard says and mark
where it is out of standardized territory (the "UNKNOWN SPN" arm does exactly this), and
the model's contract should treat the standardized decode as evidence, not as a repair
verdict. The bus names the fault; the manual and the technician diagnose it.

## The cab is not a rack

Now the physical constraints, which is where vehicles stop being easy. A model in a plant
runs in a rack in a room with power and cooling and a network drop. A model in a vehicle
runs — if it runs onboard at all — in a cab or an equipment bay, and the cab imposes a set
of constraints the rack never dreamed of.

**Power and thermal.** A vehicle's electrical system is a harsh place: voltage sags on
crank, spikes on load dump, a 12- or 24-volt bus that was never designed for a hungry
accelerator. Onboard compute lives in temperature extremes — a cab that bakes in the sun
and freezes overnight, an engine bay that runs hot by definition. The hardware that
survives this is not the hardware in the rack; it is ruggedized, fanless, thermally
throttled, and modest in its power budget. Chapter 10 covers the deployment geography in
full; the vehicle-specific point is that the compute envelope in a cab is small and
hostile, which pushes hard toward small models and away from anything that assumes
datacenter thermals.

**The operator is flying the machine.** This is the constraint that dominates everything.
The person interacting with a model in a cab is, at the same time, operating a
safety-critical machine — steering forty tons at speed, or slewing a boom over a crew, or
loading a haul truck on a grade. Their attention is a safety resource, and a model that
demands attention is a model that degrades safety. This flips the usual interaction design
on its head: the best in-cab model behavior is usually *not to speak.* A model that
interrupts a driver with a chatty diagnosis is worse than no model; a model that stays
silent until stopped, or that hands its output to a screen the operator consults at a safe
moment, or that speaks only the one thing that changes what the operator should do right
now ("oil pressure dropping, safe to stop, recommend pulling over") is a model that
respects the physical situation. The interaction budget in a cab is measured in the
operator's spare attention, which at highway speed is close to zero, and designing as if
the operator can read a paragraph mid-maneuver is designing a hazard.

**The link is intermittent.** A vehicle moves through the world, and the world has dead
zones. Telematics connectivity — the cellular or satellite link that carries data off the
vehicle — is intermittent by nature: a tunnel, a canyon, a remote quarry, a border
crossing, a basement loading dock. Any architecture that assumes the model can reach a
cloud endpoint when the operator needs it is an architecture that fails exactly when a
vehicle in trouble is somewhere remote, which is precisely when help is most needed. This
is the vehicle's version of the brownfield air gap, and it points to the same conclusion
by a different road: the model that a vehicle can rely on is the model that runs onboard,
because the model that runs in the cloud is unavailable on the schedule the physical world
chooses, not the schedule the operator needs.

## Telematics bandwidth and the store-and-forward reality

Even when the link is up, it is thin and metered. Telematics bandwidth off a moving vehicle
is a scarce, expensive resource — cellular data on a fleet of thousands of vehicles is a
real line item, and satellite backhaul from remote equipment is scarcer still. This has a
direct consequence for where the intelligence lives.

You do not stream every CAN frame off a vehicle to a model in the cloud. The CAN bus
carries a firehose — hundreds of PGNs, many at high rates — and shipping it all off-vehicle
is neither affordable nor necessary. The architecture that works is the one the plant used
in Chapter 4, adapted for the vehicle: deterministic code on the vehicle decodes and
summarizes, and only the *meaningful* result travels. The exceptions — the active DTCs, the
excursions, the state changes — are what a fleet back end needs, not the raw stream. A DM1
decode is a few hundred bytes of text; the raw CAN traffic it summarizes is orders of
magnitude more. Decode at the edge, ship the summary.

For off-highway equipment there is even a standard for this. ISO 15143-3 — the AEMP
telematics standard — defines a common data model for reporting earth-moving equipment
telematics off the machine to a back office: hours, fuel, location, fault codes, in a
vendor-neutral format [R29]. It exists precisely because fleets run mixed-vendor equipment
and needed one language for "how is this machine doing" that did not require a separate
integration per manufacturer. A model reading a mixed fleet reads this standardized report,
the same way the plant model reads the historian: at the boundary, in a normalized form,
after the vehicle's own code has done the decoding.

The store-and-forward pattern falls out of the intermittent link. The vehicle logs its
decoded exceptions locally, and syncs them when the link returns. A model that reasons over
this data has to treat the timestamps and the gaps honestly — this is the brownfield's
historian-gap lesson riding along in a truck. A fleet back end that shows a seamless
timeline over a vehicle that spent four hours in a canyon with no link is lying to whoever
reads it, and a model reasoning over that timeline will reason as if the vehicle reported
continuously. Render the gap. The vehicle was out of contact from 14:12 to 18:03; the
absence is data, and a model that cannot see the absence cannot abstain when it should.

## Diagnosis is not control

Now the boundary, stated without hedging because it does not admit hedging.

A language model on a vehicle may assist with diagnosis. It may decode faults, recall
service procedures, summarize a machine's recent health, draft a work order, and help a
technician or an operator understand what the bus is reporting. It may not, under any
circumstance in this book, participate in controlling the vehicle. It does not command
torque, steering, braking, throttle, transmission, hydraulics, or any actuator. It does not
sit in a control loop. It does not make a decision that moves the machine or changes how
the machine responds to its operator.

This is not a matter of current model quality, and it will not change when models improve.
The reason is structural and it is written into the safety framework of the entire
automotive and mobile-equipment world. Driving automation has a defined taxonomy — SAE
J3016 — that separates levels of automation precisely so that responsibility for the
dynamic driving task is never ambiguous [R26], and the regulatory posture around automated
vehicles is built on that taxonomy [R27]. Vehicle functions that can affect safety are
developed under functional-safety standards — ISO 26262 for road vehicles [R28], with the
broader IEC 61508 framework behind it [R5] — that impose a development, verification, and
validation discipline designed for deterministic, analyzable systems with quantified
failure rates. A large language model is, by construction, not such a system: its behavior
is probabilistic, its failure modes are not enumerable in advance, and its outputs cannot
be assigned the kind of failure-rate bound that a safety case requires. You cannot write
an ISO 26262 safety case for "the language model decided to reduce torque," and the honest
response to that is not to try — it is to keep the model entirely out of the control path,
where the question never arises.

So the boundary is drawn in code and in architecture, not in prompt text. The model reads
from a diagnostic tap; it has no write path to any controller. The bus arbitration, the
network segmentation, the physical wiring all enforce that the diagnostic reader cannot
become a commander, the same way the plant's model lived on a read-only boundary in Chapter
5. A prompt that says "do not control the vehicle" is a comfort, not a control; the control
is that there is no wire from the model to an actuator, and there never is one.

There is a softer version of this boundary that is just as important and easier to get
wrong: the model must not *distract* the operator into a control error either. A model that
is technically read-only but pops a paragraph of diagnosis onto a screen while the driver
is merging into traffic has affected control — by consuming the attention the control task
needed — as surely as if it had grabbed the wheel. The read-only boundary is necessary and
not sufficient; the interaction has to respect the operator's attention as a safety
resource, which is why the "best behavior is often silence" point above is a safety point
and not a UX preference.

## What the vehicle model is actually for

Strip away the two temptations — streaming everything to the cloud, and letting the model
touch control — and what remains is a genuinely valuable, entirely bounded set of jobs, and
it is worth naming them concretely so the boundary does not read as mere prohibition.

**Roadside and in-shop diagnostic assistance.** A driver stopped on the shoulder with a
red stop lamp, or a technician at a bay, points the model at the active DTCs, the recent
history, and the service manual, and gets a plain-language summary: what the codes mean,
what typically causes this combination, what to check first, and — crucially — whether the
machine is safe to move or should be shut down. Every claim quotes its source, the fault
decode or the manual page, and the "safe to move?" judgment is framed as information for a
human decision, not a decision. This turns a cryptic dash lamp into an explanation, which
is exactly the reading job models do well.

**Fleet health triage.** Across a fleet, decoded exceptions flow to a back end where a
model summarizes and ranks: which machines have recurring faults, which fault patterns
predict a road failure, which units are due for attention before they strand a load in a
remote place. This is the plant's exception-first summarization at fleet scale, reading the
normalized telematics report [R29], and it is read-only and off-vehicle, so the cab's
constraints do not apply — it is a rack job over vehicle data.

**Maintenance history and procedure recall.** The same repair-history and manual-recall
jobs from Chapter 5, applied to mobile equipment, where they matter even more because a
vehicle's service history is spread across shops and its manuals span variants and model
years — the revision trap of Chapter 5 rides in every truck. A model that can tell a
technician "this transmission code has appeared on this unit three times, twice fixed by a
sensor, once by the valve body, and here is the current procedure for this serial range"
is doing pure, valuable, bounded recall.

**Operator-facing plain-language status, at safe moments.** Not a chatbot in the cab, but a
model that can, when the operator asks at a safe moment or when stopped, explain in plain
language what the machine is reporting — turning the amber warning lamp from an anxiety
into an understood, bounded condition. The design discipline is the attention budget above:
speak little, speak only what changes the operator's decision, and default to silence.

## The bus is a network of controllers, not a sensor

A subtlety that trips up newcomers: a J1939 vehicle is not one computer with sensors. It is
a *network of controllers* — the engine controller, the transmission controller, the
anti-lock braking controller, the aftertreatment controller, the body controller, and more
— each with its own address on the bus, each reporting its own parameters and its own
faults [R25]. A DM1 does not come from "the vehicle"; it comes from a specific source
address, which identifies which controller is raising the alarm. A coolant-temperature
fault from the engine controller and an ABS fault from the brake controller are two
different subsystems reporting independently, and a model that flattens them into one
undifferentiated pile of codes has thrown away the structure that makes triage possible.

This matters for reading in two ways. First, the source address is part of the fault's
meaning and belongs in the rendering: "engine controller reports coolant temperature high;
aftertreatment controller reports a separate DEF-quality fault" is a coherent picture,
where "two faults" is not. Second, faults on a multi-controller bus can be *causally
linked across controllers* — an engine derate commanded because the aftertreatment system
faulted shows up as symptoms on the engine controller with a root cause on another node,
and a model that reasons across the source addresses can surface that linkage where a
single-node view cannot. The rendering layer should preserve which controller said what,
and the model's job over that structure is exactly the cross-subsystem reasoning that a
technician does in their head and that no single controller's self-diagnosis performs.

The bus also has a property that shapes how faults are read: CAN arbitration is
priority-based, and under heavy bus load lower-priority messages wait [R15]. This is rarely
a problem for diagnostics — DTC messages are not high-rate — but it is a reason the
decoding code must be robust to messages arriving late, out of the order a naive reader
expects, or split across frames by the transport protocol [R25]. The truncated-frame
rejection in the listing is not a corner case; on a busy bus it is the normal defense
against reading half a message and reporting a fault that the other half would have
qualified or cleared.

## Active, previously active, and freeze-frame: reading fault history

A single snapshot of active faults — the DM1 — is a thin basis for diagnosis, because it
tells you what is wrong *now* and nothing about the pattern. J1939 provides more, and a
model that reads a vehicle well uses it. DM2 reports previously active faults — codes that
have logged but are not currently present [R23] — which is exactly the difference between "the
engine is overheating right now" and "the engine has overheated intermittently five times
this month." An intermittent fault that keeps clearing is a different repair from a hard,
present fault, and the active-versus-previous distinction is the first thing a good triage
establishes.

Richer still is the freeze-frame concept: some controllers capture the operating conditions
at the moment a fault set — engine speed, load, temperature, vehicle speed — so that a
fault can be read in its context rather than as a bare code. A coolant fault that set at
full load on a grade tells a different story than the same code setting at idle in a
parking lot. Where this data is available, it is gold for a model, because it supplies
exactly the situational context that turns a code into a diagnosis. The rendering should
surface it as structured evidence — "SPN 110 FMI 0 set at 1850 rpm, 95% load, 12 km/h,
ambient recorded high" — and the model's reasoning over that context is far more grounded
than reasoning over the code alone.

The occurrence count in each DTC — the field the listing decodes as `count` — is the
cheapest history there is, and it is easy to overlook. A fault with an occurrence count of
one is a first event; a count of five is a pattern the controller has already been
watching. In the worked transcript, the oil-pressure fault carried a count of five while
the coolant fault carried one, and that asymmetry is a real clue: the oil-pressure
condition has recurred, the coolant event is new, and a technician reading the pair would
weight them accordingly. A model that reads the counts reasons about recurrence for free;
a model that ignores them treats a chronic condition and a first event as equals.

## Off-highway is its own world

On-highway trucks and off-highway equipment share J1939, but they diverge sharply in
duty cycle, environment, and what "a problem" means, and a model built for one reads the
other badly if the differences are ignored.

Off-highway equipment — excavators, loaders, dozers, haul trucks, agricultural machines —
lives a life of extreme duty cycles and brutal environments. An excavator's hydraulic
system sees load reversals thousands of times a shift; a haul truck's brakes and drivetrain
work on grades that would be illegal on a highway; agricultural equipment runs seasonally
hard and then sits. The "normal" against which a fault is judged is duty-cycle-specific, and
a model reasoning about whether a reading is anomalous needs to know the machine's operating
context, not just its instantaneous values. A hydraulic temperature that is alarming on a
highway truck is a Tuesday on a dozer digging in summer.

The worksite context also changes the value proposition. Off-highway equipment often works
in fleets on remote sites — a mine, a quarry, a farm, a construction project far from a
dealer — where a stranded machine is a bigger problem than a stranded truck near an
interstate. This raises the stakes on the "safe to keep working?" and "will this strand me
if I push it?" questions that a diagnostic model can help answer, and it raises the value of
prognostic triage that catches a developing fault before the machine dies in a pit with no
road to it. It also intensifies the intermittent-link problem: a quarry floor or a field is
exactly where cellular coverage is worst, which is exactly why the onboard, store-and-forward
architecture is not optional for off-highway fleets. ISO 15143-3 exists in large part
because these mixed off-highway fleets needed one telematics language across vendors to make
fleet-level reading possible at all [R29].

There is a prognostic opportunity here that stays firmly on the reading side of the control
boundary. Diagnosis reads what is wrong now; prognosis reads trends toward what will be
wrong soon — a hydraulic temperature creeping up shift over shift, a fuel-rail pressure
that takes longer to build each cold start, a fault occurrence count ticking up. A model
that summarizes these trends across a fleet and flags the machines drifting toward failure
is doing high-value work that touches nothing: it reads the decoded telematics history and
narrates the drift for a human to schedule against. The discipline from Chapter 5 applies —
dwell before speaking, track the hit rate, do not re-announce the same drift every day —
because a prognostic model that cries wolf trains the fleet manager to ignore it, and an
ignored prognosis is worse than none.

## The OBD-II boundary, briefly

A word for readers coming from light-duty vehicles, because the standards differ and
conflating them causes real confusion. Passenger cars and light trucks use OBD-II with
its own diagnostic standards and a different DTC format (the P/B/C/U-code scheme), commonly
carried over CAN but with a different application layer than J1939. Heavy-duty and
off-highway equipment is the J1939 world this chapter describes. The concepts transfer — a
standardized fault vocabulary, a diagnostic tap, a hard boundary against control — but the
specific decoders do not: a J1939 SPN/FMI decoder is not an OBD-II P-code decoder, and a
model's decode tables and manual corpus must match the vehicle class it actually reads.
The architectural lessons of this chapter are class-independent; the vocabulary is not.

## Cry wolf in the cab

The dashboard warning lamp is the oldest machine-to-human alarm there is, and it has a
well-known failure mode: familiarity. An operator who has seen the amber warning lamp come
and go for a fault that never amounted to anything learns to ignore it, and then misses the
one time it meant something. This is the alarm-fatigue problem of Chapter 8 in miniature,
riding in every cab, and a diagnostic model can make it better or much worse.

It makes it worse if it becomes another crying source — a model that flags every transient,
narrates every minor code, and interrupts the operator with low-value information trains the
operator to dismiss it, and a dismissed diagnostic model is a liability, because someone
paid for it and now nobody reads it. It makes it better if it does the opposite of the
lamp: instead of a binary light with no explanation, it supplies *graded, explained,
context-aware* information at the right moment — distinguishing the transient that cleared
from the pattern that is building, telling the operator plainly when a lamp means "note it
at your next stop" versus "stop safely now," and staying silent the rest of the time. The
value of a diagnostic model in the cab is not that it adds information; the cab already has
too much. Its value is that it *triages* the information the lamp cannot, and turns a wall
of undifferentiated warnings into the few that change what the operator should do. Measured
against the operator's trust — do they still read it after a month? — that triage is the
whole game, and a model that has not earned that trust by being right and quiet has not
earned a place in the cab.

## The chapter in one drawing

CAN bus (J1939) [R15] → **onboard decode** (SPN/FMI/PGN via the standard [R23][R24][R25],
truncated frames rejected) → **label and summarize** (named faults, lamp state, exceptions;
gaps rendered as gaps) → **store-and-forward** over the intermittent link → **normalized
telematics** [R29] at the fleet boundary → model (in the cab for bounded operator
assistance, in the rack for fleet triage) → **constrained diagnostic output + quoted
evidence** → **human** who operates, decides, and repairs. No arrow runs from the model to
an actuator, on any vehicle, ever [R26][R27][R28].

The vehicle gives a language model the best-structured vocabulary in this book and the
harshest physical constraints. The vocabulary makes the reading easy; the constraints make
the deployment hard; and the one line that must never blur is the line between reading the
machine and moving it. A model that decodes a fault, quotes a manual, and tells an operator
their machine is safe to limp to the next town has done something genuinely useful with the
gift the bus handed it. A model that touches the throttle has crossed a line that this
book, and the safety framework of the entire industry, draws in permanent ink. Stay on the
reading side of that line and there is a great deal of honest work to do.
