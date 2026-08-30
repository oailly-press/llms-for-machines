# Chapter 11 — Authority, Interlocks, and Abstention

*(draft v1, 2026-08-30 — written by claude-fable-5, RogerAI Labs. Published sources carry `[R#]` markers resolved in the References. Numbers attributed to this lab are the author's declared reproducible observations, stated with their apparatus. The method sibling for this chapter is the machine-readers volume, *The Abstention Reader — Cases for machines that must know when not to answer*; the plant-floor treatment is Industrial Nº 1, *Local LLMs for Manufacturing*, Chapter 5.)*

This is the chapter the whole book has been walking toward. Every earlier chapter drew the same line in a different domain — read, but do not act; suggest, but do not command; the human stays between the model and the consequence — and every earlier chapter promised that this one would say *why*, and *how*, in full. The why is a claim about the physical world that most of the AI industry has never had to confront: **next to a machine, the value of a language model is not measured by what it says but by what it is allowed to make happen, and the safest thing it can make happen is frequently nothing at all.**

That sounds like a limitation. It is actually the design. A model that stays silent when it should, and hands a human enough context to decide, is worth more on a plant floor or a mezzanine than a model that answers everything and is right most of the time — because the failure modes next to machines are not symmetric, and asymmetry is the entire subject of this chapter. We will build the argument in three layers, each older and better-understood than language models: **authority** (who may do what, a question industrial engineering answered decades ago), **interlocks** (deterministic preconditions that no probabilistic component may override, a discipline with its own body of international standards), and **abstention** (the model's trained judgment about the limits of its own competence, which is where the newest and most interesting engineering lives). Then we will connect them, because the value is in how they compose.

## Authority: the oldest question, asked of the newest component

Industrial systems have always distinguished who may *observe* a process from who may *change* it, and they enforce the distinction with far more rigor than software culture is used to. An operator watches; a control engineer changes a setpoint under a change procedure; a safety system acts autonomously within a narrowly certified envelope and nowhere else. These are not roles in an org chart; they are enforced by physical and logical access controls, by procedure, and by law. When you place a language model next to such a system, the first question is not "how good is it" but "what tier of authority does it occupy" — and the answer this book insists on is: **the model is an observer that may draft, and never more.**

It helps to name the tiers explicitly, because conflating them is the most common and most dangerous mistake:

**READ.** The model consumes telemetry, status flags, histories, manuals, and prior work orders. This is unrestricted and is where nearly all the value lives. Reading is not free of risk — a model can misread — but a misreading that goes nowhere but into another read is bounded.

**SUGGEST.** The model produces a draft that a human dispositions: a work order, a triage ranking, a diagnosis, a proposed procedure. Everything the model suggests is badged as machine-drafted and enters a queue owned by a person with the authority to accept, correct, or reject it. This is the ceiling of routine deployment in this book, and the badging is not cosmetic — it is what keeps the human's approval a real act of authority rather than a rubber stamp.

**ACT.** Writing a setpoint, commanding an actuator, resetting an alarm, releasing an interlock. This tier is not the model's, full stop — and the reason is not a prediction that models will always be too unreliable. It is structural: acting is the one interface where a probabilistic component's error becomes a physical event with no human between the error and the consequence, and the physical world does not offer an undo. The whole architecture of this series is that capability flows down toward the edge and authority does not cross the SUGGEST/ACT line without a human. A deployment that cannot state which tier its model occupies, in one sentence, at budget time, is a deployment that has not made this decision — it has only postponed it.

There is a subtlety worth stating so the rule stays credible rather than dogmatic. A model may *participate* in an action that goes through a human-owned process: it may draft the setpoint change that a control engineer reviews, tests, and commits; it may propose the revised sequence of operation that goes through commissioning. That is the model helping write the change, not making it — SUGGEST wearing work clothes. The line is not "the model never touches anything that leads to an action"; the line is "the model never *closes the loop* to a physical consequence without a human's authority in the loop." Everything in this chapter's machinery exists to keep that line bright even when the model is confident, even when it is usually right, even when acting would be convenient.

Why hold the line so hard? Because the human-factors literature settled the question before language models existed, and it settled it against the convenient answer. Bainbridge's "Ironies of Automation" `[R72]` observed that the more you automate, the more the residual human role becomes monitoring an automation that is usually right — which is precisely the task humans are worst at, and which erodes the very skill needed to catch the automation when it fails. Parasuraman and Riley's taxonomy of *use, misuse, disuse, and abuse* of automation `[R71]` named the failure directly: **misuse** is over-trust in automation past its competence, and **abuse** is designers automating a function because it is technically possible without regard for the human consequences. A model with ACT authority next to a machine is an engraved invitation to both. The authority tiers are how you design the system so that the human's role stays an *active* authority — dispositioning drafts, keeping the skill and the context that let them catch the wrong one — rather than the passive, deskilled monitor Bainbridge warned would fail exactly when needed.

## Interlocks: the deterministic floor no model may cross

Authority says who *may* act. Interlocks say what *must be true* before an action is even possible, and — this is the crucial property — they are enforced deterministically, below and independent of any model, so that no amount of model confidence can buy past them. This is the layer where industrial safety engineering has the most to teach, and the most humility to demand.

An interlock is a precondition wired into the control system such that a dangerous action cannot occur unless the precondition holds: a press will not cycle unless the light curtain is unbroken; a robot cell's motion is inhibited unless the gate is closed; a burner will not light unless purge airflow has been proven. The defining feature is that the interlock does not *advise* — it *prevents*, in hardware or in certified logic, without consulting anything as fallible as a decision-making component. The relevant standards are a family worth knowing by name even if you never certify to them, because they encode the discipline: **IEC 61508** is the foundational functional-safety standard for electrical/electronic/programmable safety-related systems `[R5]`; **IEC 62061** `[R45]` and **ISO 13849** `[R65]` specialize it to machinery safety; and **IEC 61511** (with its US sibling ANSI/ISA-84) covers safety instrumented systems for the process industries `[R66]`. Their shared concept is the *safety integrity level* — a quantified, verified reliability target for a safety function — achieved through simplicity, redundancy, diversity, and above all *determinism*. A safety function must do the same thing every time, provably.

Now hold a language model up against that standard, honestly. A model is stochastic, its behavior on an unseen input is not provable in the sense a safety case requires, and — as Chapter 10 discussed — even its *speed* varies with software beneath it. **A language model cannot be a safety function, and this book makes no claim toward IEC 61508 certification for any model.** That is not a weakness to apologize for; it is a category fact to design around. The relationship between the model and the interlock is not that the model implements the interlock, and not that the model is trusted because an interlock exists somewhere. It is this: **the interlock is the deterministic floor that stays valid no matter what the model does, and the model operates entirely above it, in the space of things that are safe regardless of whether the model is right or wrong.**

This has a clean design consequence, and it is the most important architectural rule in the chapter. **The interlock check comes first, it is deterministic, and it fails closed.** Before any model output is even considered as a suggestion, the deterministic preconditions are evaluated by non-model code; if they do not hold, the proposed action is blocked, and no model confidence — however high, however well-calibrated — changes that. The model does not get a vote on the interlock. It gets to operate in the space the interlock has already guaranteed is safe. A model that proposes "reset the overtemperature trip" is not evaluated on how confident it is; it is blocked because resetting a safety trip is not in the space the interlock leaves open to a suggestion. This ordering — interlock, *then* everything else — is what the code listing later in this chapter encodes as its first and unconditional check.

## Abstention: the model's judgment about its own limits

Authority is a policy. Interlocks are deterministic engineering. Abstention is the one layer that lives *inside* the model — its trained judgment about when the evidence supports an assertion and when it does not — and it is where the newest and most interesting work is. It is also the layer this lab has measured most directly, so this section can move from principle to apparatus.

Selective prediction — the idea that a classifier should be allowed to *decline* on inputs where it is likely wrong, trading coverage for reliability — is not new to language models. Chow formalized the error-versus-reject tradeoff in 1970 `[R67]`, and the modern treatment runs through El-Yaniv and Wiener's foundations of selective classification and Geifman and El-Yaniv's extension to deep networks `[R68]`. The core object is a **risk–coverage curve**: as you allow the model to abstain on more of its least-confident inputs, the accuracy on the ones it *does* answer climbs. The engineering question is where to set the threshold, and the safety question is whether the confidence signal you threshold on is *trustworthy* — which is the calibration problem Guo and colleagues showed modern networks fail by default, being systematically overconfident `[R69]`. For language models specifically, Kadavath and colleagues found that models have a meaningful, if imperfect, internal sense of what they do and do not know `[R22]`, and the out-of-distribution detection literature `[R70]` studies the harder case of an input unlike anything the model was trained on. This is a real, mature field, and the machine-readers sibling volume, *The Abstention Reader* `[R88]`, is built entirely around applying it to machine data. This chapter's contribution is to connect it to the authority and interlock layers, and to report what this lab measured when it built the machinery for real.

The practical signal this lab uses, and the one that connects abstention to the constrained-decoding discipline of Chapter 4, is the **enum-decode margin.** When every output field is a closed enumeration — a fault class from a fixed taxonomy, a NAMUR NE 107 state, a BACnet reliability value — the decode is not free generation; it is a set of multiple-choice questions, each candidate value scored by its summed token log-probability, with the argmax taken. This is exactly deterministic, it structurally cannot emit an invalid value (a stronger guarantee than a grammar, which can still loop into schema-valid nonsense), and — the point here — **it yields a margin per field: the gap in nats between the best value and the runner-up.** That margin is the natural basis for abstention. A wide margin means the model strongly prefers one value; a thin margin means it is nearly indifferent, which is precisely when it is most likely wrong.

This lab measured how well that margin actually works, on a channel-level fault-assertion task with known ground truth, and the numbers are worth stating precisely because they are both encouraging and cautionary. On the per-channel task — the kind of single-machine fault reading this book's typed-output shape targets — the margin is a strong reliability signal: channel-level accuracy is **monotone across all ten margin deciles**, climbing from about 34% in the lowest-margin decile to 100% in the highest, with an AUROC of about 0.94 and a low expected-calibration error `[LAB: RogerAI Labs bench, RESULTS-MATRIX R.158, 2026-08; enum-scoring decoder, private holdback n=3,725 (channel-level slice n=3,535); AUROC 0.9377 with a bootstrap 95% CI of [0.9288, 0.9459] and ECE 0.0121; the decode is bit-exact across two process launches (640 rows, 0 score differences, 0 argmax flips), so there is no run-to-run variance and the interval is the bootstrap over items; replicated on a 9,000-item public-dev set at AUROC 0.9354 / ECE 0.0178, with no holdback gap]`. Turned into the operator's language: at the shipped escalation floor of 2.0 nats, the model answered about **72% of channels at roughly 98.8% accuracy while catching about 93% of the errors it would otherwise have made** (measured 71.9% / 98.78% / 92.9% on the holdback, and 71.6% / 98.96% / 94.5% on public-dev) `[same observation]`. That is a deployable abstention policy — it converts a model that is right most of the time into a model that is right almost always on the fraction it chooses to answer, and honest about the rest.

## The margin's honest limit, and why it changes the design

Now the cautionary half, because it is more important than the encouraging half and it is the kind of finding this lab's honesty policy requires be published beside the good news. **A margin ranks errors *within* a task the model can do; it cannot tell you a task is *beyond* the model.** The mechanism is clean and worth internalizing: a model with no real signal on a task does not produce thin, uncertain margins — it *collapses onto a prior* and reports that prior with a wide, confident margin. This lab saw it directly: on a scene-level diagnosis task the model could not actually do, it became a near-constant predictor (returning the same value on almost every input) at a *median margin above the escalation floor* — confidently wrong, with a wide margin, on a task it had no competence for. On the same instrument, at a smaller model size, the scene-level reliability signal did not just weaken; it *inverted* `[LAB: RogerAI Labs bench, RESULTS-MATRIX R.158, 2026-08; scene-level slice n=190, AUROC 0.5482 with a bootstrap 95% CI of [0.4703, 0.6352] — an interval that straddles the 0.5 no-signal line, which is the measurement of "no signal"; at 281M the scene-level AUROC fell to 0.306, significantly inverted]`. This is exactly the R.26 finding from Chapter 9 wearing its abstention clothes: hand a general model a task it cannot do, and it will not be uncertain — it will be confidently, uniformly wrong, and the margin will *endorse* it.

The generalizable statement, and it belongs on a wall somewhere: **a confidence margin tells you which of the model's answers to trust on a task it can do; it does not tell you whether the task is one it can do.** Those are different questions, and conflating them is how a well-calibrated-looking system fails silently on the tasks it was never able to handle. The design consequence is that **margin thresholding needs a per-task degenerate-strategy check sitting beside it** — a cheap detector for "the model is collapsing onto a prior" (has it returned the same value for the last several distinct inputs? is its output distribution suspiciously flat across inputs that should differ?) — because the margin alone will happily wave through a degenerate strategy. The code listing below implements exactly this: the margin check and the degenerate-strategy check are separate gates, and a wide margin does not exempt an output from the degeneracy check. That separation is not gold-plating; it is the difference between an abstention policy that works and one that endorses the failure it was supposed to catch.

One more measured nuance, because it corrects a natural but wrong assumption. It is tempting to equate "low margin" with "should have abstained." This lab's data says they are different things: choosing the explicit `abstain` value (when the taxonomy offers one) was highly precise, but a *low margin on a non-abstain answer* almost never meant abstain was the correct answer — it meant the answer was probably *wrong* `[LAB: RESULTS-MATRIX R.158, 2026-08]`. So "low margin" routes to *escalation or verification*, not to a claim that the true answer was "insufficient data." The distinction matters for how you wire the downstream: a thin-margin output is a "get a second opinion" signal, not an "the evidence was absent" conclusion, and treating them the same throws away information the operator needs.

## The instrument is part of the claim

A short but load-bearing digression, because this lab learned it the hard way and it is the kind of thing that invalidates abstention numbers everywhere. **Your abstention measurement is only as trustworthy as the harness that produced it, and abstention harnesses have a specific, sneaky failure mode.** While building an independent control scorer, this lab found that its production enum-scorer's results depended on the *order* the candidate values were presented — the scoring code reused a single cache object across candidates, and the underlying library mutated it in place even when told not to, so a candidate's score drifted with its position in the choice list, by up to tens of nats. The measured impact on one task's accuracy was several points once fixed — one task moved 88.93% → 95.91% (+6.98 points, 45 argmax flips), replicated on public-dev at +6.73 points `[LAB: RogerAI Labs, RESULTS-MATRIX R.159/R.160, 2026-08; order-dependent enum scores, disagreement up to 48.7 nats; one-line fix (crop the cache to the prompt length), verified order-independent afterward and agreeing with an independent full-forward scorer to 0.177 nats; the fix did not change the calibration verdict — AUROC 0.9377 corrected vs 0.9259 buggy]`. The lesson generalizes past this bug: a margin-based abstention policy is a claim about a scoring instrument, and the instrument must be validated (score a known-answer probe, check order-invariance, cross-check against an independent scorer) before its margins mean anything. This is the same discipline as Chapter 10's "measure your own box, with error bars," applied to the abstention signal itself. An abstention gate built on an unvalidated scorer is a gate that abstains and asserts for reasons that have nothing to do with the evidence — the worst possible outcome, because it *looks* principled.

And the deeper cultural point behind that digression: this lab retracts its own findings when the instrument turns out to be the story rather than the world. A recent finding here was marked under retraction and then retracted in full when investigation showed it rested on four instrument defects rather than a real effect. That is not an embarrassment to hide; it is the honesty policy working, and it is the standard any abstention claim — including every number in this chapter — should be held to. A model that must know when not to answer is being built by people who must know when not to publish.

## Composing the three layers: an authority gate you can run

Authority, interlocks, and abstention are only as good as the way they compose, and the composition has a fixed order that never changes. Here is that order, implemented as a small, deterministic, stdlib-only gate — the code that sits between a model's output and any consequence. It runs in a restricted sandbox (no network, milliseconds, trivial memory). Read it as the executable form of everything above.

```python
#!/usr/bin/env python3
"""authority_gate.py -- an authority/interlock/abstention gate for a model
proposing something next to a machine. Stdlib only; deterministic.

Three checks stand between a model's output and any consequence, applied in a
fixed order that never changes:

  1. INTERLOCK  -- a deterministic precondition the model cannot override.
                   Fails closed. No model confidence buys past it.
  2. MARGIN     -- the enum-decode margin (top choice minus runner-up, in
                   nats). Below a floor, the model ABSTAINS rather than
                   assert. A margin ranks errors WITHIN a task the model can
                   do; it does NOT prove the task is within reach, so a
                   degenerate-strategy check sits beside it.
  3. AUTHORITY  -- what this actor is permitted to do with the assertion.
                   READ and SUGGEST are the ceiling; ACT is never the model's.

The output is always one of: SUGGEST (a draft for a human), ABSTAIN (with the
missing evidence named), ESCALATE (a packet handed up), or BLOCKED (interlock).
It is never a direct action on the machine.
"""

import math

MARGIN_FLOOR_NATS = 2.0          # below this the model abstains
MODEL_MAX_AUTHORITY = "SUGGEST"  # the model never exceeds this


def enum_margin(scores):
    """scores: {value: summed_token_logprob}. Returns (top_value, margin_nats)."""
    ranked = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)
    top_val, top_lp = ranked[0]
    runner_lp = ranked[1][1] if len(ranked) > 1 else float("-inf")
    return top_val, top_lp - runner_lp


def is_degenerate(history):
    """Sits BESIDE the margin: a no-signal model collapses onto a prior and
    reports it with a WIDE, confident margin. If the same value came back for
    the last k distinct inputs, treat a high margin as suspect, not trusted."""
    k = 8
    if len(history) < k:
        return False
    return len(set(history[-k:])) == 1


def gate(assertion, interlock_ok, recent_history):
    """Fixed order: interlock, then margin+degeneracy, then authority.
    Every branch fails toward silence, never toward action."""
    top_val, margin = enum_margin(assertion["scores"])

    # 1. INTERLOCK -- deterministic, fails closed, model cannot buy past it.
    if not interlock_ok:
        return {"decision": "BLOCKED",
                "reason": "interlock precondition not satisfied",
                "proposed": top_val}

    # 2. MARGIN + degenerate-strategy check.
    if is_degenerate(recent_history):
        return {"decision": "ESCALATE",
                "reason": "degenerate output pattern -- task may be beyond model",
                "proposed": top_val, "margin_nats": round(margin, 2)}
    if margin < MARGIN_FLOOR_NATS:
        return {"decision": "ABSTAIN",
                "reason": f"margin {margin:.2f} < floor {MARGIN_FLOOR_NATS}",
                "needed": assertion.get("would_resolve", "corroborating evidence"),
                "proposed": top_val, "margin_nats": round(margin, 2)}

    # 3. AUTHORITY -- clamp to the model's ceiling; ACT is never returned.
    requested = assertion.get("requested_authority", "SUGGEST")
    granted = requested if _rank(requested) <= _rank(MODEL_MAX_AUTHORITY) \
        else MODEL_MAX_AUTHORITY
    return {"decision": granted, "assertion": top_val,
            "margin_nats": round(margin, 2),
            "for_human": assertion.get("action_hint")}


def _rank(a):
    return {"READ": 0, "SUGGEST": 1, "ACT": 2}[a]
```

Driving it through five representative cases produces this transcript:

```
=== authority/interlock/abstention gate ===

  A. clear fault, interlock OK:
      decision        : SUGGEST
      assertion       : reheat_valve_stuck
      margin_nats     : 5.3
      for_human       : inspect reheat valve/actuator on VAV-4-17

  B. model requests ACT (over its ceiling):
      decision        : SUGGEST
      assertion       : reheat_valve_stuck
      margin_nats     : 5.3
      for_human       : inspect reheat valve/actuator on VAV-4-17

  C. thin margin (0.4 nats):
      decision        : ABSTAIN
      reason          : margin 0.40 < floor 2.0
      needed          : zone temp sensor reliability flag
      proposed        : reheat_valve_stuck
      margin_nats     : 0.4

  D. confident, but interlock open:
      decision        : BLOCKED
      reason          : interlock precondition not satisfied
      proposed        : reheat_valve_stuck

  E. wide margin, degenerate pattern:
      decision        : ESCALATE
      reason          : degenerate output pattern -- task may be beyond model
      proposed        : none
      margin_nats     : 16.0
```

Read the five cases as the chapter's argument in miniature. **A** is the happy path: a clear reading, the interlock satisfied, a draft handed to a human with the missing-evidence-resolved and the action hint attached. **B** is the authority tier doing its job: the model *requested* ACT authority, and the gate clamped it to SUGGEST regardless — the model does not get to escalate its own authority by asking. **C** is abstention: a thin 0.4-nat margin falls below the floor, so the gate declines to assert and names the specific evidence that would resolve it (the zone-temp sensor's reliability flag) — an abstention that is a work item, not a shrug. **D** is the interlock's absolute priority: the *same confident assertion* from case A is BLOCKED because the interlock precondition is open, and note that the gate never even reached the margin or authority checks — the interlock is first and unconditional, and no model confidence bought past it. **E** is the margin's honest limit made operational: a *wide* 16-nat margin that would normally sail through, escalated instead because the degenerate-strategy check caught the model returning the same value on eight straight inputs — the failure mode the R.158 measurement warned about, caught by the gate that sits beside the margin rather than trusting it.

Notice what the gate never returns: ACT, or anything that touches the machine. Its entire output space is {SUGGEST, ABSTAIN, ESCALATE, BLOCKED} — four flavors of "hand this to a human, or hand it up, or hand it nowhere." That is not a limitation of this particular listing; it is the thesis of the chapter compiled into a return statement.

## Two kinds of interlock, and where the model's reading is allowed near them

The word "interlock" covers two things that must not be confused, and the distinction determines exactly how close a model's output is allowed to get. A **hardware interlock** is a physical circuit — a light curtain wiring the machine's power, a gate switch in series with a motor contactor — that prevents a dangerous action electrically, independent of any software at all. A **logic interlock** is a precondition enforced in the control logic (often in a safety PLC certified to the standards above), which prevents an action in software that is itself built and verified to a safety integrity level. Both share the property that matters: they are deterministic, verified, and outside the model's reach. A model cannot participate in either, cannot advise either, and cannot be a reason to weaken either.

Where, then, is the model's reading allowed near the interlock? In exactly one place: the **advisory layer that sits above the interlock and is distinct from it.** A model may read the same signals the interlock reads and produce a *non-safety* advisory — "the light curtain on cell 3 has tripped 14 times this shift, well above its baseline; the beam alignment may be drifting" — precisely because that advisory changes nothing about the interlock's operation and reaches a human who schedules maintenance. The advisory is helpful *because* it is powerless over the interlock; the moment a model's output could relax, bypass, or reset an interlock, it has crossed from the advisory layer into the safety function, and that crossing is prohibited without exception. The gate listing above encodes this as the unconditional first check: the interlock state is an input the gate *reads and respects*, never an output the gate *produces*. A model that could produce it would be a safety function, which it cannot be `[R5]`.

This distinction is what lets a model be genuinely useful right next to safety-critical equipment without being part of the safety case. It reads everything, it advises freely in the space the interlocks have already made safe, and it is structurally incapable of touching the mechanisms that keep that space safe. That is not a compromise between usefulness and safety; it is the configuration in which both are maximized at once.

## Against autonomy creep

The most likely way a carefully-designed deployment fails is not a dramatic breach; it is *creep* — the slow, reasonable-at-each-step migration of authority from SUGGEST toward ACT, driven by a model that keeps being right. The pattern is seductive precisely because every individual step is defensible. The model's suggestions are accepted 95% of the time, so someone proposes auto-accepting the high-confidence ones to save the technician's clicks. That works, so the confidence bar for auto-acceptance drifts down. The auto-accepted actions are almost always fine, so the human review becomes a glance, then a batch approval, then a weekly audit, then nothing. No single step crossed a bright line, and yet the human is no longer between the model and the consequence — which is exactly the deskilled-monitor endpoint Bainbridge predicted and Parasuraman and Riley named as misuse `[R71][R72]`.

The defenses against creep are structural, not attitudinal, because attitudes erode and structures do not. First, the authority ceiling is enforced in *code that the model cannot influence* — the gate clamps to SUGGEST regardless of confidence, as case B in the transcript showed, so "auto-accept the confident ones" requires a human to change a policy constant under change management, not a threshold that quietly drifts. Second, the abstention and disposition streams are monitored for the *tell* of creep: a rising auto-accept rate, a falling human-correction rate that is not matched by rising measured accuracy, a review step whose average dwell time is collapsing toward zero. Those metrics make creep visible while it is still reversible. Third — and this is the cultural spine — the deployment states its authority ceiling *out loud*, in the deployment record, in the same words at budget time and at incident review, so that raising it is a deliberate, documented, reviewable decision rather than an emergent property of a hundred small conveniences. A ceiling nobody wrote down is a ceiling that rises on its own.

The point is not that the ceiling can never rise. It is that raising it must be an *act of authority by a named human under review*, subject to the same management-of-change discipline as any other change to what a machine does — never a side effect of the model being good at its job. The better the model gets, the stronger the pull toward creep, and the more the structural defenses earn their place. A model that is right 99% of the time next to a machine is not an argument for giving it ACT authority; it is an argument for watching the disposition metrics more carefully, because it is exactly the regime in which the human's vigilance decays fastest.

## The escalation packet: making "I don't know" the first step of the answer

Abstention and escalation are only valuable if what the model hands up is *useful*. An abstention that produces a dead end on a technician's screen teaches the crew that the tool "doesn't know anything," and usage collapses — the deployment dies with perfect calibration, which Industrial Nº 1's abstention chapter names as the predictable organizational failure. The fix is not in the weights; it is in the *packet* the model hands up when it declines or escalates.

A good escalation packet carries everything the human (or the larger model upstream) needs to decide *without re-doing the work*: the input as rendered (the actual telemetry and status flags the model saw, so the human is not re-fetching), the proposed value and its margin (so the human knows how close the call was), the specific evidence that would resolve the question (the "what's missing" field — the sensor to check, the manual section to pull, the corroborating trend to look at), the interlock and authority state (so the human knows whether this is blocked, low-confidence, or beyond-competence), and the reason for the escalation in the model's own terms. The gate above populates the skeleton of this packet on every non-SUGGEST branch: the `needed` field on ABSTAIN, the `reason` on ESCALATE and BLOCKED, the `proposed` and `margin_nats` throughout. A packet with those fields turns "I don't know" into "here is exactly what I saw, how close I came, and what would let anyone answer this — for you, and for everyone after you." That is the refusal becoming the first step of the answer, which is what a refusal next to a machine should always be.

The packet also closes the two-model loop from Chapter 8. An abstention from the small always-on reader at the edge is not a terminus; it is a *routing event* that forwards the packet — rendered input, proposed value, margin, missing evidence — up to the larger on-site model or the human queue. Capability flows down, data flows up, and the escalation packet is the data flowing up in a form the next tier can act on immediately. Design the packet well and the hierarchy works; design it as a bare "insufficient data" and the hierarchy is just a slower way to reach a shrug.

## Change management: where the human authority actually lives

The authority tiers are enforced by policy and code, but they *live* inside an organization's change-management culture, and a deployment that ignores that culture will have its tiers quietly eroded no matter how good its gate is. The process industries codified this as management of change — in the US, OSHA's Process Safety Management standard requires that changes to process, technology, and equipment go through a documented review before they are made `[R73]` — and the discipline generalizes to any machine domain: a change to what a machine does is reviewed, authorized by a named person, documented, and reversible.

A language model deployment has to fit *inside* that discipline, not around it, and this is where the SUGGEST-not-ACT rule stops being an abstract safety principle and becomes an organizational fit. When the model drafts a setpoint change or a revised procedure, that draft enters the *existing* management-of-change process as a proposal — reviewed and committed by the control engineer who owns that authority, under their name, logged and reversible like every other change. The model has not bypassed change management; it has fed it. This is why "the model participates in a change that goes through a human-owned process" is safe while "the model acts" is not: the former is a proposal into a process built to catch bad changes, and the latter is a change that skipped the process. A deployment that lets the model act is not just taking a safety risk; it is routing around the organization's own control of its machines, and no gate the model runs can substitute for the review the organization already requires of every human who changes those machines.

The organizational reciprocal, from Industrial Nº 1's abstention chapter and worth restating here: **the organization has to reward the refusal, or the refusal dies.** Track abstentions by cause the way you track alarms by tag — the histogram is a ranked map of your documentation and instrumentation debt, ordered by how often reality asks for each missing piece. In review meetings, treat a correctly-refused unanswerable exactly like a correctly-answered question (both are the system working) and treat a confident fabrication that reached a human as the incident it is. An organization that punishes the model's "I don't know" trains its people to route around the gate, and an organization that rewards it gets, for free, a prioritized list of what to fix so that next time the answer is available to everyone, forever.

## The failure this chapter exists to prevent

It is worth naming the specific failure all three layers are aimed at, because it is subtle and it is the one that kills deployments and, occasionally, worse. It is not the model being wrong — models are wrong, that is priced in. It is the model being **confidently wrong in a way that reaches a physical consequence with no human able to catch it.** Every layer targets a different route to that outcome. Authority ensures a human is structurally between the model and the consequence. The interlock ensures that even if everything above it fails, the deterministic floor holds and the dangerous action simply cannot occur. Abstention ensures the model itself declines the calls it is likely to get wrong — and the degenerate-strategy check ensures it declines even the calls it is *confidently* likely to get wrong on tasks beyond its competence. The escalation packet ensures the human who catches it has enough to actually decide. And change management ensures the whole thing sits inside an organization built to review changes rather than around it.

Remove any one layer and you have a plausible path to the failure. Keep all of them, composed in the fixed order the gate encodes, and the confident-wrong-with-consequences path is closed at multiple independent points, each of which would have to fail together — which is exactly the redundancy-and-diversity logic the safety standards teach `[R5]`, applied to a component the standards themselves would never certify. The model is not made safe; the *system around the model* is made safe, deterministically, in ways that hold whether the model is right or wrong. That is the only kind of safety a probabilistic component can honestly be part of.

## The chapter in one sentence

Next to a machine, put three layers between a model and any consequence — authority that keeps the model an observer that may only draft, interlocks that are deterministic and fail closed and that no model confidence can buy past, and abstention that declines the calls the model is likely to get wrong (with a degenerate-strategy check beside the margin, because a model with no signal is confidently wrong, not uncertain) — compose them in that fixed order, hand up an escalation packet that makes "I don't know" the first step of the answer, and fit the whole thing inside the organization's change-management culture rather than around it; do that, and the most valuable behavior your model has next to equipment is the silence it keeps, and the honest hand-up it makes, exactly when it should.

The next chapter turns to measurement: how you *prove*, with holdouts and error bars and promotion packets, that a model wired this carefully actually behaves this way before it is allowed anywhere near a machine that can hear it.

---
