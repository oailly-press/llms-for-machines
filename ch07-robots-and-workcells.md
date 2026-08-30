# Chapter 7 — Robots and Workcells

*(draft v1, 2026-08-30 — written by claude-fable-5 (RogerAI Labs), unverified. Public
safety and control standards are cited as `[R#]` and resolve in the References; the
author's own runnable listing is labeled as reproducible apparatus; claims without either
are labeled unmeasured. This chapter draws a hard line at motion, interlocks, and cycle
time and does not cross it.)*

A robot cell looks, from a distance, like one machine doing one job. Up close it is a small
society: a robot controller running motion, a programmable logic controller sequencing the
cell, a vision system finding parts, a safety controller watching the guarding, a conveyor
or indexer feeding material, and a warehouse or manufacturing execution system telling the
cell what to make next. These are separate computers with separate jobs, wired together by
digital I/O, fieldbus, and a safety circuit, and the cell works because they agree — every
cycle — on who does what and when. A language model that walks into this society has to
understand two things at once: that it is joining a multi-agent physical system, and that
almost none of the agents already there will hand it any of their authority, correctly.

The one-sentence thesis: **a workcell is a coordinated society of controllers, and a
language model is welcome as its librarian and its interpreter — reading the logs, recalling
the procedures, narrating the changes — but never as one of the controllers, because motion,
interlocks, and cycle time are real-time, safety-rated, deterministic responsibilities that
a probabilistic reader cannot hold.**

## The cell as a multi-agent system

Start by naming the members of the society, because the model's job is defined by which of
them it reads and which it must never impersonate.

The **robot controller** runs the motion. It executes the taught program, interpolates the
paths, drives the servos, and enforces the robot's own limits. It is a hard real-time
system: a joint command is due every few milliseconds, and a late command is not a slow
answer, it is a fault. The **PLC** sequences the cell — it decides when the robot may enter,
when the fixture clamps, when the conveyor advances, coordinating the members through a
scan-cycle logic that is itself a real-time program written in the languages of IEC 61131-3
[R38] or structured as the function blocks of IEC 61499 [R39]. The **safety controller** —
often a separate, safety-rated device — watches the guarding: the light curtains, the
interlocked gates, the emergency stops, the safety-rated speed and position monitoring. It
has the authority to stop everything, and its logic is developed to a safety standard [R45]
precisely so that nothing clever can talk it out of stopping. The **vision system** finds
and inspects parts and hands coordinates or pass/fail results to the robot or PLC. And above
the cell, the **MES or WMS** issues the work — what to build, in what order, from which
materials — integrating the cell into the plant's operations through the layered model of
ISA-95 [R31].

Each member has a clock, an authority, and a failure mode, and the cell's correctness is
the agreement among them. The critical observation for this book is that the agreement is
*real-time and deterministic*: the members coordinate on millisecond-to-second timescales
through signals whose meaning and timing are fixed, and the safety of the whole rests on
that determinism. A language model operates on a completely different timescale — hundreds
of milliseconds to seconds per inference, with no hard bound — and with a completely
different reliability profile. It cannot be a member of the real-time society. It can only
be an observer of it and a servant to the humans who tend it, and every design decision in
this chapter follows from taking that seriously.

## What language helps with, concretely

The good news is that the observer-and-librarian role is genuinely valuable, because a cell
generates exactly the kinds of text and event streams that no member of the society is good
at reading and that a human tending the cell spends real time wrestling.

**Alarm and fault triage.** A robot controller and a PLC both emit alarm logs, and the logs
are cryptic — coded strings that a technician learns to read over years. When a cell faults,
the first question is always "what stopped it, and can I recover, or do I need help?" A model
that reads the controller's alarm log and the PLC's fault word, decodes the codes into plain
language, distinguishes a recoverable program fault from a safety-latching stop, and points
at the recovery procedure is doing the single most valuable reading job in the cell. The
listing later in this chapter does exactly the classification-and-gating part of this.

**Procedure and program recall.** Cells accumulate procedures — how to recover from this
fault, how to re-home the robot after a collision stop, how to run the changeover to the
next product, how to verify the fixture after maintenance. This knowledge lives in manuals,
in tribal memory, and in the heads of the two people who really understand the cell. A model
with the procedures in reach turns "how do I recover from a SRVO collision stop on this
robot" into a quoted, sourced answer at 3 a.m. when the two people who know are asleep. As
in every chapter, the quoting is load-bearing: a recovery procedure paraphrased wrong is a
person reaching into a cell wrong, so the model quotes the procedure and cites the page, and
the human verifies against the source.

**Change narration.** Cells change — a teach-pendant program gets edited, a setpoint gets
tuned, a fixture gets shimmed — and the change log is where the cell's drift lives or is
lost. A model that reads the before-and-after of a program change and narrates it in plain
language ("the pick approach height was raised 5 mm and the gripper dwell increased 100 ms,
in the palletize routine") turns an opaque diff into a change note a human can review and
sign. This is pure reading, it touches nothing, and it builds the audit trail that makes the
next fault diagnosable — because half of cell faults trace to "something changed and nobody
wrote it down."

**Handoff and shift continuity.** The shift-handoff job from Chapter 5 applies to a cell as
much as to a plant: what faulted this shift, what was recovered, what is limping, what the
next shift should watch. It is the same read-only, benign-failure-mode, trust-building
application, and it is the right first deployment in a cell for the same reasons.

Notice what unites these: every one is reading text and events and narrating them for a
human, on a timescale of seconds-to-minutes where the model's latency is irrelevant, with a
failure mode that a human catches before it reaches the physical cell. That is the entire
safe envelope, and it is a large one.

## Classifying cell alarms, and gating the model's role

Here is a stdlib-only Python listing that classifies robot-controller alarm-log lines into
typed triage records and — the load-bearing part — gates the model's role by the cell's
safety state. The subsystem prefixes follow the common controller convention (SRVO for
servo and safety, MOTN for motion, INTP for the program interpreter, SYST for system, TPIF
for the teach-pendant interface). The transcript below it is real output from running it on
this box (Python 3.13, RogGentoo).

```python
# robot_alarm.py — classify cell alarms; gate model role by safety state.
import re
SUBSYS = {                                    # prefix -> (domain, class)
    "SRVO": ("servo / drive / safety", "SAFETY"), "MOTN": ("motion / kinematics", "SAFETY"),
    "SYST": ("controller system", "FAULT"), "INTP": ("program interpreter", "PROGRAM"),
    "TPIF": ("teach-pendant interface", "PROGRAM"), "PROG": ("program logic", "PROGRAM"),
}
LATCHING = {"SAFETY"}          # classes that latch a protective stop -> model READ-ONLY
LINE = re.compile(r"^ALARM\s+(?P<ts>\S+\s+\S+)\s+(?P<code>[A-Z]{3,4}-\d{3})\s+"
                  r"(?P<msg>.+?)\s*(?:\(G:(?P<grp>\d+)\s+A:(?P<axis>\d+)\))?\s*$")

def classify(line):
    m = LINE.match(line.strip())
    if not m:
        return {"raw": line.strip(), "parsed": False,
                "note": "unrecognized alarm format - do not guess, escalate"}
    domain, klass = SUBSYS.get(m["code"].split("-")[0], ("unknown subsystem", "UNKNOWN"))
    return {"ts": m["ts"], "code": m["code"], "subsystem": domain, "class": klass,
            "latching": klass in LATCHING, "msg": m["msg"].strip(),
            "group": m["grp"], "axis": m["axis"], "parsed": True}

def triage(lines):
    recs = [classify(l) for l in lines if l.strip()]
    parsed = [r for r in recs if r.get("parsed")]
    latching = [r for r in parsed if r["latching"]]
    out = ["CELL ALARM TRIAGE", f"alarms parsed: {len(parsed)}/{len(recs)}"]
    if latching:
        out.append(f"SAFETY-LATCHING alarms present: {len(latching)} -> cell in protective stop")
        out.append("MODEL ROLE: READ-ONLY. Recall procedure + rank likely cause. "
                   "MUST NOT propose motion or clear the interlock.")
    else:
        out.append("no safety-latching alarm; recoverable program/system faults")
    for r in recs:
        if not r.get("parsed"):
            out.append(f"  ?? {r['raw']}  <- {r['note']}"); continue
        loc = f" [G{r['group']}/A{r['axis']}]" if r["group"] else ""
        flag = "  << latches protective stop" if r["latching"] else ""
        out.append(f"  {r['ts']}  {r['code']} ({r['subsystem']}){loc}: {r['msg']}{flag}")
    return "\n".join(out)
```

Running it against a realistic cell alarm burst — a collision detect, an operator E-stop, a
program stack error, a released deadman, and one garbled line — produces, verbatim:

```text
CELL ALARM TRIAGE
alarms parsed: 4/5
SAFETY-LATCHING alarms present: 2 -> cell in protective stop
MODEL ROLE: READ-ONLY. Recall procedure + rank likely cause. MUST NOT propose motion or
  clear the interlock.

  2026-08-30 09:14:02  SRVO-050 (servo / drive / safety) [G1/A3]: Collision detect alarm
    << latches protective stop
  2026-08-30 09:14:02  SRVO-001 (servo / drive / safety): Operator panel E-stop
    << latches protective stop
  2026-08-30 09:14:05  INTP-105 (program interpreter): Runtime stack error PROG=PALLETIZE
  2026-08-30 09:15:31  TPIF-062 (teach-pendant interface): Teach pendant deadman released
  ?? SRVO-050 garbled line without prefix header  <- unrecognized alarm format - do not
    guess, escalate
```

The design choice that matters is the gate at the top. The moment a safety-latching alarm is
present — a collision detect, an E-stop — the cell is in a protective stop, and the code
stamps the model's role as READ-ONLY with an explicit prohibition against proposing motion
or clearing the interlock. This is not a suggestion buried in a prompt that a clever context
might override; it is a deterministic classification computed before the model is ever
consulted, and it is computed from the alarm codes themselves per the subsystem map. A cell
in a protective stop is a cell where a human must physically intervene — inspect the
collision, clear the guarding, reset the safety circuit at the pendant — and the model's
entire job in that state is to help the human understand what happened and recall the
recovery procedure, never to suggest a way to make the robot move. The garbled last line is
the safety default in miniature: an alarm the parser cannot read is escalated, not guessed,
because a misread safety alarm is worse than an unread one.

There is a real subtlety in that transcript. The two safety-latching alarms carry different
meanings a technician must distinguish — a collision detect (SRVO-050) suggests the robot
hit something and needs inspection before anything moves, while an operator E-stop
(SRVO-001) may simply mean someone pressed the button, a benign and common event. The model
can help narrate that distinction and recall the right procedure for each. What it cannot do
— what the gate forbids — is convert "the E-stop was probably just pressed by mistake" into
"so it is safe to reset and continue." That inference, however plausible, is a human's to
make at the cell, with eyes on the guarded space, because the one time the collision was
real and the model waved it through is the time someone gets hurt.

## What stays outside the model, permanently

Three responsibilities in a cell are outside the language model, not for now but by their
nature, and it is worth stating each with its reason so the boundary reads as engineering
rather than caution.

**Motion planning and motion control.** The robot controller plans and executes paths under
hard real-time constraints, respecting the robot's kinematics, dynamics, and limits. This is
a solved, deterministic, safety-relevant problem with decades of engineering behind it, and
a language model has nothing to add to it and everything to endanger. A model does not
generate robot programs that run unreviewed, does not adjust paths, does not command joints.
Where a model helps a programmer draft or explain a program, the output is a *draft a human
reviews and tests*, exactly like the work-order drafting of Chapter 5 — assistance to the
programmer, never a path to the servos. The line is the same as the vehicle's throttle in
Chapter 6: there is no wire from the model to the motion.

**Interlocks and the safety circuit.** The safety controller and the guarding exist to make
the cell safe for the humans near it, and they are developed to safety standards — the robot
safety requirements of ISO 10218 [R34][R35], its US adoption as R15.06 [R37], the
machinery-safety framework of ISO 13849 [R45] — precisely so that their behavior is
deterministic, verified, and un-overridable by anything outside the safety system. A
language model does not clear an interlock, reset a safety stop, mute a light curtain, or
touch the safety circuit in any way. It cannot, by architecture, and it must not, by intent.
For collaborative applications — where robot and human share space under ISO/TS 15066 [R36]
— the same rule holds with more force: the safety-rated speed and force monitoring that makes
collaboration safe is a deterministic safety function, and a model's role is to help humans
understand and maintain it, never to be part of it. And when a human enters a cell for
service, the control of hazardous energy is governed by lockout/tagout procedure [R46]; a
model that could re-energize a cell would defeat the one procedure that keeps a maintainer
alive, so it has no such capability, full stop.

**Cycle time.** The cell's throughput is a real-time property — the members coordinate to
produce a part every N seconds, and the timing is engineered. A language model in the
critical path of the cycle would inject unbounded, variable latency into a system whose
whole value is deterministic timing, and it would do so with no reliability guarantee. The
model is never in the cycle. It reads the cell's output — the logs, the counts, the events —
on its own timescale, and narrates and triages after the fact. A model that "optimizes cycle
time" by sitting in the loop is a model that has been placed exactly where it does the most
harm; a model that reads the cycle's history and suggests to an engineer where the time goes
is doing analysis, on the safe side of the line.

## Reading across the members: the coprocessor pattern

The multi-agent nature of the cell creates a reading opportunity that a single-controller
view misses, and it is worth drawing out because it is where a model earns more than
per-controller triage.

A cell fault often has a root cause in one member and symptoms in another, exactly like the
cross-ECU faults of Chapter 6. The vision system fails to find a part; the robot faults
because it moved to grab nothing; the PLC faults because the cycle did not complete; the WMS
flags the order as short. Four members raise four alarms, and the naive reading is four
problems. The correct reading is one problem — the vision system did not find the part — with
three downstream consequences, and a model that reads across all four members' logs, aligned
in time, can surface the causal chain where each member's own diagnostics see only their
slice. This is the coprocessor pattern: the language model is a *cross-member interpreter*,
reading the logs of the whole society and narrating the cell-level story that no single
member holds, because no single member sees the others' internals.

Doing this well requires the same discipline as everywhere in this book. The members' clocks
must be aligned or the model will invent causal orderings out of clock skew — a failure mode
Chapter 13 catalogs, and one that is acute in a cell where members' timestamps come from
different devices. The rendering must preserve which member said what, so the model reasons
over a structured multi-source picture rather than a flattened pile. And the output stays a
narration for a human — "root cause appears to be vision no-find at 09:14:01, with the robot
and PLC faults following as consequences; recommend checking the part presentation and vision
lighting" — with the causal claim anchored to the timestamps it rests on, and the human
owning the diagnosis and the fix.

## The WMS/MES handoff and the tyranny of the interface

The cell does not exist in isolation; it takes orders from and reports status to the plant's
execution systems through the layered integration model of ISA-95 [R31]. This handoff is
where a language model can help and where it can do a specific kind of damage if it forgets
which side of an interface it is on.

The help is real: the handoff generates text and structured messages — orders, completions,
exceptions, short-material flags — and a model can summarize the cell's state for the MES in
human terms, draft the exception reports that a supervisor reviews, and translate between the
cell's cryptic status and the plant's operational language. This is reading and narrating
across the ISA-95 boundary, and it is valuable because the boundary is exactly where cell
detail and plant abstraction meet and where things get lost in translation.

The damage is the temptation to let the model *act* across the interface — to acknowledge an
order, confirm a completion, release the cell to the next job. This is the same authority
boundary as everywhere, and it holds here for the same reason: a completion confirmed by a
model that misread the cell is a false record propagating into the plant's execution system,
where it becomes the basis for downstream decisions — shipping a part that was not made,
consuming material that was not used. The model drafts; the human or the deterministic
system confirms. The interface between the cell and the plant is a place for the model to
interpret, never to sign.

## Change management: the model as the cell's memory

I want to give change narration its own section, because in cells it is quietly the highest-
leverage read-only application after fault triage, for a reason specific to how cells fail.

An enormous fraction of cell problems trace to undocumented change. A program was edited to
work around a bad batch of parts and never edited back. A gripper dwell was tuned by a
weekend technician who did not write it down. A fixture was shimmed during a changeover. Then
weeks later the cell misbehaves, and the diagnosis stalls because nobody knows what is
different from the last time it worked. The cell's memory failed before the cell did.

A language model is unusually good at being that memory, entirely within the read-only
envelope. When a teach-pendant program changes, the model reads the diff and narrates it in
plain language for a change log. When a setpoint moves, the model records the before, the
after, and — if the operator supplies it — the reason. Over time this builds exactly the
change history that makes the next fault diagnosable: "the cell last worked cleanly before
the 09-15 change that raised the pick approach and increased gripper dwell; the current fault
began after that change." The model did not make the change, does not evaluate whether the
change was wise, and does not undo it — it *witnesses and narrates*, turning the cell's
silent drift into a legible record. This is librarian work of the purest kind, and it pays
off precisely when the cell is hardest to diagnose, which is when a change nobody logged has
finally caught up with it.

## The teach pendant and the operating modes

The teach pendant is the handheld terminal a programmer or operator uses to jog the robot,
edit programs, and recover the cell, and it is worth understanding because it is where a
human and a robot are physically closest and where the safety framework is most explicit —
which tells a model exactly where it does and does not belong.

Industrial robots operate in distinct modes, and the mode determines what is safe. In
automatic mode the robot runs its program at full speed with the guarding closed and no one
inside. In a manual or teach mode, a person is inside the safeguarded space jogging the
robot at reduced, safety-rated speed, holding a three-position enable device — the deadman —
that stops the robot the instant it is squeezed too hard or released [R34]. The alarm in the
worked transcript, TPIF-062 "teach pendant deadman released," is exactly this device doing
its job. The whole architecture of pendant teaching is built to keep a human safe in the one
situation where the guarding is open and the robot can move near them, and it is developed to
the robot safety standards [R34][R35] with the machinery-safety framework behind it [R45].

A language model's relationship to the pendant is instructive precisely because it is so
constrained. The model does not jog the robot — jogging is motion, and motion is the
controller's under a human's thumb on the deadman. The model does not edit programs into the
running controller. What the model *can* do is be the reference the person at the pendant
consults: recall the recovery procedure, explain what an alarm means, narrate what a program
change did, quote the step they are unsure of. Imagine a technician inside a cell after a
collision stop, pendant in hand, unsure of the re-homing sequence. The valuable model is not
one that offers to move the robot; it is one that, when the technician asks at a safe moment,
quotes the exact re-homing procedure from the manual with its page reference, so the human
performs the motion with the controller and the deadman while the model supplies the words.
The pendant is the sharpest illustration of the whole chapter: the closer the human is to the
moving robot, the more strictly the model stays a source of words and never a source of
motion.

## Collaborative robots do not lower the bar

Collaborative robots — cobots designed to share space with people without traditional
guarding — are often marketed as if their inherent safety loosens the constraints this
chapter insists on. It does the opposite, and a model deployed around a cobot has to
understand why.

A collaborative application is safe not because the robot is gentle in some vague sense but
because a specific, safety-rated function limits it: power-and-force limiting that keeps any
contact below documented biomechanical thresholds, or safety-rated speed and separation
monitoring that slows or stops the robot as a person approaches, all under ISO/TS 15066
[R36] within the ISO 10218 framework [R34][R35]. The safety lives in that deterministic,
verified function — the same kind of safety-rated logic as a light curtain, just embodied as
force and speed limits instead of a guard. A collaborative robot with its force limiting
defeated is not a friendly robot; it is an unguarded industrial robot next to a person.

The consequence for a language model is that "collaborative" changes nothing about the
boundary. The model still does not command motion, still does not touch the force or speed
limits, still does not participate in the safety function — and in fact the stakes are higher
in a sense, because the whole premise of collaboration is that a human is *routinely* within
reach of the robot, so any path from a probabilistic component to the robot's behavior sits
even closer to a person than in a guarded cell. The model's role around a cobot is the same
librarian-and-interpreter role: read the logs, recall the procedures, narrate the changes,
help the humans understand and maintain the safety function that keeps collaboration safe.
The friendliness of the robot is a property of its safety-rated limits, and those limits are
not a place for a language model to be clever.

## The vision coprocessor: read the results, not the pixels

The vision system deserves its own treatment because it is the cell member most often
confused with a language model, and the confusion leads to bad architecture.

A cell's vision system does a real-time, deterministic job: locate a part, measure a feature,
pass or fail an inspection, and hand a result — coordinates, a measurement, a verdict — to
the robot or PLC within the cycle's time budget. This is not a language task and it is not a
job for a language model in the loop; it is machine vision, engineered for the cycle time and
the required reliability, and a language model injected into that path would break the timing
and the determinism exactly as it would in the motion path. The vision system is a
coprocessor with its own clock, and the model is not in its loop.

Where the model helps is one step back, reading the vision system's *output stream and logs*
rather than its pixels. Vision systems drift — lighting changes over a shift, a lens fouls, a
part supplier's tolerance shifts — and the drift shows up as a rising no-find rate, creeping
measurement bias, or a climbing false-reject rate long before it becomes a hard failure. A
model that reads the vision system's result log over time and narrates the drift ("no-find
rate on station 2 has risen from 0.3% to 4% over the shift, concentrated after the 14:00
material change") is doing valuable prognostic reading, entirely outside the real-time loop,
on the safe side of the line. It is the same trend-narration job as the off-highway prognosis
of Chapter 6 and the standing-watcher pattern the series returns to: code computes the rates,
the model narrates what deserves a human's attention, and the human decides whether to clean
a lens, adjust lighting, or call the supplier. The model reads what the vision system reports
about itself; it never tries to be the vision system.

## Measuring whether the cell model helps

A cell is a measured place — it runs on cycle counts, first-pass yield, uptime, and overall
equipment effectiveness, and there is an international vocabulary for these operations
metrics in ISO 22400 [R47]. That measurement culture is an asset, because it means the
question "is the language model actually helping?" can be answered with numbers the cell
already respects rather than with enthusiasm.

The cell-specific metrics worth tracking are concrete. **First-time recovery rate**: of the
faults where the model offered a triage and a recovery procedure, how often did the
technician recover on the first attempt without escalating? A model that raises this rate is
demonstrably shortening downtime; one that does not is decoration. **Time-to-diagnosis**: how
long from fault to understood cause, with the model versus the historical baseline? **Triage
disposition**: the one-click useful / noise / wrong / already-known logging from Chapter 5,
applied to every model output a technician reads, reviewed on a schedule like any instrument's
calibration. And the trust question that underlies all of them, asked bluntly: after a month,
do the technicians still open the model's triage, or have they gone back to the old way? A
cell model that is quietly unused is a failed deployment regardless of how good its answers
were, and the only way to know is to measure the using, not the answering.

This matters in a cell for the same reason it mattered in the brownfield: the model's failure
rate on real, messy cell logs will be higher than any clean demo suggested, and the only
honest posture is to measure it against real dispositions on real faults and let the numbers,
not the vendor slides, decide whether the model stays and how far up the trust ladder it
climbs. Chapter 12 gives the full measurement discipline; the cell minimum is that a
deployment nobody measures is a deployment that will, sooner or later, be believed on the one
shift it should not have been.

## The failure modes a cell invents

A brief bridge to Chapter 13, because cells invent physical failure modes that a
pure-software view of a language model would never anticipate, and a model reading a cell has
to be built expecting them. A stuck digital input reads a clamp as closed when it is open,
and a model reasoning from the I/O will confirm a fixture that is not holding. A phantom
sensor — a proximity switch that welded on — reports a part present that is not there. Clock
skew between the robot, the PLC, and the vision system produces impossible causal orderings
in a merged log, so the model "discovers" that an effect preceded its cause. A maintenance
override left in place after a service call changes the cell's behavior in a way no log
records. Each of these is a case where the cell's own data lies, and the defense is the one
from Chapter 4 and Chapter 5 carried into the cell: deterministic code labels the suspect
signal — stuck input, skewed clock, active override — so the model treats it as an evidence
gap rather than a fact, and abstains where the data does not support a claim. A model that
trusts every I/O bit in a cell will be confidently wrong exactly when a sensor has failed,
which is exactly when the cell most needs an honest reader.

## The chapter in one drawing

Cell members — robot controller, PLC [R38][R39], safety controller [R34][R45], vision, MES/WMS
[R31] — each real-time and authoritative → **read the logs and events** (aligned clocks,
member identity preserved) → **classify and gate** (safety-latching alarm ⇒ model READ-ONLY,
computed in code) → **cross-member interpret** (one root cause, its downstream consequences)
→ model → **plain-language triage / procedure recall / change note, with quoted evidence** →
**human** who inspects the guarded space, plans the motion, clears the interlock, confirms
the completion, and holds every scrap of authority. No arrow runs from the model to motion,
to the safety circuit, or into the cycle [R34][R35][R36][R45][R46].

The workcell is the most multi-agent machine in this book, and the temptation it creates is
to make the language model one more agent in the society — a coordinator, an optimizer, a
motion helper. That temptation is a hazard, because the society is real-time and safety-rated
and the model is neither. The model's honest place is one step back: the librarian who reads
every member's log, the interpreter who narrates the cell-level story, the memory that
witnesses every change. In that place it makes the humans who tend the cell faster,
better-informed, and better-briefed, and it never once reaches toward a servo, a light
curtain, or the clock the cell runs on. That is a smaller job than the demos promise and a
larger job than most cells have ever had filled. Fill it well, and leave the motion to the
controller built to plan it.
