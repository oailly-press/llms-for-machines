# Chapter 12 — Measuring Models That Touch Machines

*Draft status: author draft, gate-checked; human verification pending. The listing in this
chapter is pure–standard-library Python, deterministic under the seed shown, and was executed
by the author during writing; the printed output is a real transcript. External claims resolve
to the cited references; measured numbers attributed to the lab are the author's own
reproducible observations on the apparatus described at the end of the chapter.*

## The score on the card is not the number you need

There is a specific disappointment waiting for anyone who tries to choose a language model for
a machine by reading its scores. The scores are real, and they are carefully made, and they
answer a question you did not ask. A leaderboard tells you how a model does on a large, public,
general-purpose suite of questions, averaged and normalized so that many models can be ranked
on one axis [R79]. That axis is genuinely informative about the thing it measures. It is nearly
silent about whether the model, wired to the read path you built in the earlier chapters, will
name the right fault code on your line often enough, and — far more important — refuse to name
the wrong one often enough, that a maintenance crew comes to trust it. Those are different
questions, and the gap between them is where most disappointed deployments live.

The reason is not that the leaderboard is careless. It is that a general benchmark measures a
general capability, and a machine deployment is a specific system doing a specific job under
specific failure economics. The sibling volume in this series, *Measure Twice* `[R86]`, is a whole book
about not fooling yourself with benchmark numbers, and everything it says holds here without
amendment: a number without an error bar is a rumor, a surprising number gets re-measured before
it is believed, and a comparison without a matched control is not a comparison. This chapter does
not repeat that argument; it assumes it. What it adds is the part *Measure Twice* could leave
implicit because its subject was measurement in general: when the model touches a machine, the
thing you must measure changes shape. The metric that matters is not accuracy. The suite cannot
come from the public web. The ground truth is sitting in a historian you already own. And the
document you produce at the end is not a blog post with a chart; it is a promotion packet that a
reliability engineer has to be willing to sign.

## The metric a plant actually cares about

Start with the failure economics, because they dictate the metric. Everywhere else in the
industry, a model is scored on how often it answers correctly, and a refusal or a blank counts
against it exactly as much as a wrong answer. That accounting encodes an assumption — that the
cost of a wrong answer and the cost of no answer are roughly equal — and next to a machine that
assumption is false. As the abstention argument earlier in this book laid out, and as the sibling
industrial volume argues at length, the two failures are not symmetric on a plant floor. A model
that says "I cannot tell from this frame; check the gauge" costs you a lookup. A model that
confidently reports the wrong bearing costs you the teardown of a good one, and then it costs you
something you cannot easily buy back: the crew's willingness to believe the next answer, including
the correct ones. A single confident wrong answer at the wrong moment can retire a tool.

So the headline metric for a model that touches a machine is not accuracy. It is the **confident-wrong
rate**: the fraction of presented cases where the model answered, answered above whatever confidence
or margin threshold you set as its licence to speak, and was wrong. Accuracy on the cases it chose
to answer is a secondary figure, useful for tuning the threshold. Abstention rate is a third,
useful for knowing whether the model is earning its keep or hiding behind silence. But the number
you defend to the people who own the machine is the confident-wrong rate, because it is the number
that maps directly onto the cost they fear. A model that answers 70 percent of frames with a
confident-wrong rate of half a percent is more deployable, on most floors, than one that answers
95 percent of frames with a confident-wrong rate of four percent, and no accuracy-first score will
ever tell you that. The whole point of measuring a machine-facing model is to measure the failure
the machine owner is actually paying to avoid.

This reframes the abstention machinery from the earlier chapter as a *measurable* object rather
than a virtue. Abstention is not good in itself; a model that refuses everything has a
confident-wrong rate of zero and is worthless. What you are measuring is the trade the model makes
along its confidence axis: as you raise the margin it must clear before it is allowed to speak, its
confident-wrong rate falls and its abstention rate rises, and there is some operating point on that
curve where the model answers enough to be useful while staying below the confident-wrong budget the
plant can tolerate. Measuring a machine-facing model means measuring that whole curve, not a single
point on it, and then choosing the operating point on purpose rather than accepting the one the
model shipped with.

## Your suite cannot come from the public web

The second thing that changes is where the questions come from. A general benchmark can draw its
questions from the public commons because the capability it measures is general. Your deployment is
not general; it is this machine, these tags, this plant's dialect of a protocol, this site's
particular history of bodges and overrides. A model's score on somebody else's questions tells you
almost nothing about its score on yours, and worse, the public questions carry a contamination risk
that your private ones do not.

Contamination is the leakage of benchmark items into the model's training data, which turns a high
score into a measure of memorization rather than of capability [R76][R80]. In the general case it is a
subtle statistical problem. In the industrial case it is often blunt and obvious, and it cuts the
other way from how people expect. The equipment manuals, the fault-code tables, the protocol
specifications, the vendor knowledge-base articles — these have frequently been on the public web for
years, and a large model has very likely seen them. That means a model can look impressive at
decoding a fault code from a famous PLC family not because it can reason about your machine but
because it memorized that vendor's fault table during training. The score is real and the capability
is fake, and you find out which when you point the same model at your own equipment, whose quirks
were never in any training set — the flatlined sensor still reporting its last good value, the tag
renamed in 2019 and never updated in the manual, the setpoint the night shift moves by hand and
puts back before the morning walk-through. A model that scored beautifully on the public manual can
fall apart on the plant it was hired to watch, and a suite drawn from the public web will never warn
you, because it is measuring the memorized part.

The defense is the one *Measure Twice* names and the manufacturing volume builds its quality gate
around: a private, recent, held-out suite drawn from your own plant is worth far more than a famous
one drawn from the commons [R76]. Held-out means the model has not been tuned on it — not shown it
during any prompt engineering, not used to pick a threshold, not consulted so many times that its
specific items have leaked into your choices, which is its own slow form of contamination even when
the model never saw them [R77]. The suite you measure a machine-facing model on should be items that
came off your machine, that the model has never been optimized against, and that you keep sealed the
way a lab keeps a reference standard sealed: you break it out to measure, you record the result, and
you do not let the act of measuring teach anyone how to score better on it next time. The day your
held-out suite starts influencing your prompts is the day it stops being a measurement and becomes a
target, and a target is a thing models learn to hit without doing the underlying job. Where the machine's
task can be scored inside one, prefer running the suite through a vetted, shared evaluation harness rather
than a hand-rolled script, because a widely used harness has had its silent scoring and templating failures
found and fixed by many users and freezes those choices so your comparison is about the models, not about a
bug in your own grader [R78].

## The oracle you already own

The good news, and it is genuinely good, is that a plant is the rare place where ground truth is
cheap. Most language-model evaluation is expensive precisely because someone has to decide what the
right answer *was*, and for open-ended tasks that adjudication is slow, subjective, and contestable.
A machine does not have that problem, because a machine keeps records. The historian — the process
data store that most plants already run — is a log of what the equipment actually did, timestamped,
at whatever resolution it was configured for. That log is an oracle. It is not a perfect oracle, and
part of this chapter is about its imperfections, but it is an oracle you already own, and it turns
evaluation from an adjudication problem into a replay problem.

Replay works like this. Take a window of historian data from a period where you know, after the
fact, what was happening — a known fault that was diagnosed and fixed, a known good run, a known
sensor failure. Reconstruct the read path exactly as the model would have seen it live: the same
registers, the same units, the same rendering into text, the same window of context. Feed the model
that reconstructed frame and record what it would have said. Then compare what it said to what
actually happened, which the historian and the maintenance record already know. You have just
measured the model against reality without a human adjudicating a single item, and you can do it
across thousands of frames as fast as you can read them off disk. Replay is the industrial answer to
the expensive-oracle problem, and it is available to you precisely because your subject is a machine
that logs itself rather than a conversation that does not.

Two disciplines make replay honest rather than self-flattering. The first is that the replay must
reconstruct the *live* frame, not a cleaned-up one. If your production read path renders a flatlined
sensor as its stuck value with no flag, then your replay must render it the same way, because the
whole question is whether the model handles the frame it will actually get, warts included. A replay
harness that quietly fixes the data before showing it to the model measures a model you will never
deploy. The second discipline is that the oracle label must be independent of the model — it comes
from the maintenance record, the confirmed diagnosis, the physical inspection, never from a second
run of the same model or a larger model asked to grade the first. A model grading a model shares the
first model's blind spots and manufactures agreement that looks like accuracy. The oracle is what
the wrench found, not what a bigger network guessed.

## Missing is not wrong

Now a subtlety that is easy to get wrong and that quietly wrecks otherwise careful evaluations. When
you run thousands of replayed frames through a model over a network, some fraction of the requests
will fail — a timeout, a dropped connection, a malformed frame the renderer choked on, a server that
returned an error instead of an answer. The question is how to score those, and the answer, which
*Measure Twice* states as a rule and which matters even more here, is that a failed request is
recorded as **missing** and reported as a rate; it is never scored as a wrong answer. A zero that
means "the model answered incorrectly" and a zero that means "the request never reached the model"
are different facts about different systems — one is about your model, the other is about your
uptime — and a harness that collapses them into the same zero manufactures findings out of
infrastructure hiccups.

This trap is not hypothetical; it is the exact mechanism behind a finding the author had to retract
in full. During a benchmarking run, HTTP 500 errors from the inference server were being scored as
0.0 — as if the model had answered and answered wrong — and the depressed scores that resulted looked
like a real capability difference until the load was read carefully enough to see that the "wrong
answers" were server errors that never produced an answer at all `[LAB: PROJECT-LOG 2026-08 —
mmlu_framing_probe stopped turning HTTP 500s into 0.0 scores; Finding 25 retracted in full as
instrument defects, not a finding]`. The lesson is cheap to state and expensive to learn: count the
missing rate as you go, treat any run with a non-trivial missing rate as a run about your
infrastructure rather than your model until proven otherwise, and never let a failed fetch masquerade
as a model being wrong. In an OT harness this is doubly important, because the network between your
harness and your model is often the same flaky plant network the deployment will live on, so failures
are common and the temptation to bury them is strong.

## The listing: a promotion-packet scorer

The following listing puts these pieces together into the smallest honest scorer for a machine-facing
model. It replays frames that each carry an oracle label; each side — an incumbent already in service
and a candidate you are considering — returns either a decoded label with a confidence margin, or
`None` when the request failed. A `None` is scored missing, never wrong. An answer whose margin falls
below the abstain threshold is a refusal, not an error. The metric it reports for each side is the
confident-wrong rate, and the comparison it makes is paired — the incumbent is the control, run now on
the same frames — with a bootstrap confidence interval on the *change* in confident-wrong rate [R75].
The verdict refuses to claim the candidate is safer unless the interval excludes zero.

```python
import random, statistics

# A promotion-packet scorer for a model that decodes machine state.
# Each replayed historian frame carries an oracle label. Each side
# (incumbent, candidate) returns either a decoded label with a
# confidence margin, or None when the request failed (scored MISSING,
# never as a wrong answer). An answer below the abstain margin is a
# refusal, not an error. The metric a reliability engineer signs is the
# CONFIDENT-WRONG rate: answered, above margin, and wrong.

ABSTAIN = 0.15  # minimum top-vs-runner-up margin to be allowed to answer

def outcome(label, oracle, margin):
    if label is None:
        return "missing"
    if margin < ABSTAIN:
        return "abstain"
    return "right" if label == oracle else "wrong"

def rates(rows):
    n = len(rows)
    miss = sum(r == "missing" for r in rows)
    answered = [r for r in rows if r not in ("missing",)]
    ab = sum(r == "abstain" for r in answered)
    confident = [r for r in answered if r != "abstain"]
    wrong = sum(r == "wrong" for r in confident)
    cw = wrong / n  # confident-wrong over ALL n frames presented (marginal)
    acc = (sum(r == "right" for r in confident) / len(confident)) if confident else float("nan")
    return acc, cw, ab / n, miss / n

def bootstrap_diff_ci(paired, iters=10000, alpha=0.05, seed=0):
    rng = random.Random(seed)
    n = len(paired)
    xs = []
    for _ in range(iters):
        xs.append(sum(paired[rng.randrange(n)] for _ in range(n)) / n)
    xs.sort()
    return xs[int((alpha / 2) * iters)], xs[int((1 - alpha / 2) * iters)]

rng = random.Random(7)
N = 400
inc_rows, cand_rows, paired_cw = [], [], []
for _ in range(N):
    oracle = rng.randrange(6)                      # 6 possible states
    hard = rng.random() < 0.22                     # ambiguous frame
    # incumbent: sometimes fails to fetch (missing); modest margins
    inc_fail = rng.random() < 0.02
    inc_margin = rng.uniform(0.0, 0.25) if hard else rng.uniform(0.15, 0.6)
    inc_label = None if inc_fail else (oracle if rng.random() < (0.62 if hard else 0.93) else rng.randrange(6))
    # candidate: better calibrated -> abstains more on hard frames,
    # fewer confident-wrong, similar throughput
    cand_fail = rng.random() < 0.02
    cand_margin = rng.uniform(0.0, 0.12) if hard else rng.uniform(0.2, 0.65)
    cand_label = None if cand_fail else (oracle if rng.random() < (0.60 if hard else 0.94) else rng.randrange(6))
    io = outcome(inc_label, oracle, inc_margin)
    co = outcome(cand_label, oracle, cand_margin)
    inc_rows.append(io); cand_rows.append(co)
    # paired confident-wrong indicator difference (candidate - incumbent),
    # only on frames both actually answered-or-abstained (not missing)
    if io != "missing" and co != "missing":
        paired_cw.append((co == "wrong") - (io == "wrong"))

for name, rows in (("incumbent", inc_rows), ("candidate", cand_rows)):
    acc, cw, ab, miss = rates(rows)
    print(f"{name:9s}  answered-acc {acc*100:5.1f}%  confident-wrong {cw*100:4.2f}%  "
          f"abstain {ab*100:4.1f}%  missing {miss*100:4.2f}%")

paired_n = len(paired_cw)
dropped = N - paired_n
print(f"paired frames: {paired_n} of {N} presented "
      f"(dropped {dropped} = {dropped/N*100:.2f}%, a missing on either side)")
point = sum(paired_cw) / len(paired_cw)
lo, hi = bootstrap_diff_ci(paired_cw)
print(f"paired confident-wrong change (cand-inc): {point*100:+.2f} pts  "
      f"95% CI [{lo*100:+.2f}, {hi*100:+.2f}] pts")
better = hi < 0
worse = lo > 0
print("verdict:", "SAFER (fewer confident-wrong, CI excludes 0)" if better
      else "REGRESSION (more confident-wrong)" if worse
      else "indistinguishable on confident-wrong at this suite")
```

```output
incumbent  answered-acc  94.4%  confident-wrong 4.75%  abstain 11.8%  missing 2.75%
candidate  answered-acc  94.8%  confident-wrong 4.00%  abstain 20.2%  missing 3.00%
paired frames: 377 of 400 presented (dropped 23 = 5.75%, a missing on either side)
paired confident-wrong change (cand-inc): -1.33 pts  95% CI [-4.24, +1.59] pts
verdict: indistinguishable on confident-wrong at this suite
```

The transcript is worth sitting with, because it says something a naive reading of the top two lines
would miss. The candidate looks better on every headline: slightly higher accuracy on the frames it
answered, a lower confident-wrong rate, and much more abstention on the hard frames — exactly the
profile of a better-calibrated model that knows when to stay quiet. If this were a leaderboard, the
candidate would win. But the paired comparison, which looks only at the frames where the two models'
outcomes differ, brackets the change in confident-wrong rate at minus 1.33 points with a 95 percent
interval that runs from minus 4.24 to plus 1.59 — and that interval includes zero. Note the third line,
which is easy to omit and load-bearing: the paired comparison runs on 377 of the 400 frames, not all
400, because a frame with a *missing* on either side is dropped from the pair. That paired-drop rate —
5.75 percent here — is larger than either model's marginal missing rate (2.75 and 3.00 percent), because
a pair is lost whenever *either* side fails, and it defines the denominator behind the point estimate and
the interval. A packet that printed only the marginal missing rates would let a reader assume the interval
covered the full suite; on a plant network where fetch failures correlate across models, the paired-drop
rate can climb while the marginal rates stay flat, and hiding it hides a bias. Report both, always. The honest verdict
is not "the candidate is safer." It is "indistinguishable on confident-wrong at this suite," which is
a polite way of saying that four hundred frames cannot resolve a one-point difference in a rate that is
itself only a few percent, and that promoting the candidate on this evidence would be promoting it on
noise. To earn the "SAFER" verdict you would need a larger held-out suite, or a larger true effect, or
both; the harness will not let you pretend otherwise, and that refusal is the most valuable line it
prints.

## Small suites swing hard, and machine suites are usually small

That last point deserves its own treatment, because it is where machine evaluations most often go
wrong. The sampling arithmetic is unforgiving: the uncertainty on a rate shrinks only as the square
root of the number of items, so a suite of a few hundred frames leaves an error bar of several points
on any rate you compute, and a suite of a few dozen — which is what many plants can assemble of a rare
fault — cannot distinguish a good model from a mediocre one at all. This is not a machine-specific fact;
it is the same square-root floor *Measure Twice* derives for any proportion. What is machine-specific is
that your suites will tend to be small, because the interesting cases — the rare faults, the near-misses,
the ambiguous frames — are rare by definition, and you cannot manufacture more of a fault that has
happened three times in the plant's history.

The author has measured this swing directly on the authoring apparatus, and it is larger than intuition
suggests. A fifteen-scenario tool-calling suite, run twice against the same large mixture-of-experts
model with an unchanged binary and a fixed seed at temperature zero, produced scores about ten points
apart — not because anything changed between the runs, but because batch-packing nondeterminism in the
served inference amplified by the model's expert routing flipped a handful of scenarios, and on a
fifteen-item suite each flipped scenario is worth several points `[LAB: RESULTS-MATRIX §C — 15-scenario
tool-eval-bench hardmode shows ±10 pts of run noise; 5/15 scenarios flip between identical back-to-back
runs at temperature 0.0]`. The temperature-zero endpoint that felt deterministic was not, because a
production server almost never serves a batch of one, and the numerics a request sees depend on what
else shares its batch [R74][R13]. The direct consequence for machine evaluation is that a small suite run
once is not a measurement of your model; it is a single draw from a distribution whose spread you have
not characterized. Run the suite more than once, treat the run-to-run spread as data, and let it — not
your hope — decide how many digits of the score you are allowed to print.

There is a second consequence that points toward the remedy. Adding more runs of a small suite
characterizes the execution and decoding noise but does nothing about the sampling floor; adding more
items shrinks the sampling floor but does nothing about execution noise. The two are not
interchangeable, and spending your budget on the wrong one is a common and expensive mistake. When your
machine suite is small and irreplaceable, the highest-leverage move is not more runs or more items — it
is *pairing*, as the listing does. A paired comparison looks only at the frames where the two systems
differ, cancelling each frame's shared difficulty, and recovers real statistical power on a fixed suite
that an unpaired comparison of two separate run-averages simply cannot. On a rare-fault suite you cannot
grow, pairing is often the only thing standing between you and an unfalsifiable claim.

## Blast-radius tests: measure what a wrong answer can reach

Accuracy metrics measure how often the model is right. A machine deployment also needs to measure how
much damage the model can do when it is wrong, and those are independent questions. A model with a low
confident-wrong rate whose every output can open a valve is more dangerous than a model with a higher
confident-wrong rate whose every output is a suggestion a human reads. The first is a small probability
of a large loss; the second is a larger probability of a trivial one. No accuracy number distinguishes
them, so the evaluation has to measure the blast radius directly.

A blast-radius test asks, for each class of model output, what is the worst thing that output can cause,
and then verifies by construction that the worst thing is bounded. If the model's output is advisory
text a mechanic reads, the blast radius is a wasted lookup and the test is trivial. If the model's
output selects among a closed set of actions, the test enumerates the set and confirms that every member
of it is survivable — that there is no output in the model's vocabulary that reaches an unsurvivable
state, because the unsurvivable states are guarded by an interlock the model cannot address. This is the
NIST OT-security posture translated into evaluation terms: you assume the component can misbehave and you
bound what its misbehavior can reach, rather than trusting that it will behave [R30]. The measurement is
not "does the model choose the safe action" — you will never drive that probability to one — but "is every
action the model can choose one that the surrounding system makes safe." A deployment that can only pass
the first test is trusting the model; a deployment that passes the second is engineered.

The practical form of a blast-radius test is a deliberate adversarial replay. You do not only feed the
model the frames where it should do well; you feed it the frames designed to make it do badly — the
ambiguous ones, the ones with a flag it might ignore, the ones that look like a known pattern but are
not — and you confirm two things: that its confident-wrong rate on these hard frames stays within budget,
and that even its wrong answers stay inside the bounded set. The author's own tool-calling work found
that the scenarios most worth watching were exactly the adversarial ones — an ambiguous recipient, a set
of near-duplicate tools designed to be confused — and that whether a model handled them was more diagnostic
of real capability than its average score, which the easy scenarios dominated `[LAB: RESULTS-MATRIX §C —
adversarial scenarios (TC-70 near-duplicate tools, TC-71 ambiguous recipient) are the scenario-consistent
discriminators the aggregate score hides]`. On a machine, the adversarial frames are where the trust is
won or lost, so they are where the measurement budget belongs.

## Latency is a quality, and you must measure it under load

Accuracy and confident-wrong rate are not the only qualities a machine cares about; timing is a quality
too, and a machine-facing evaluation that measures only correctness has measured half the deployment. An
answer that is correct but arrives after the moment it was needed is, for many machine interactions,
indistinguishable from a wrong answer — the operator has already acted, the frame the model was reasoning
about is no longer the current state, and a correct diagnosis of a condition that has since changed can be
worse than silence because it sends the crew after a ghost. Freshness is part of correctness on a machine
in a way it never is in a chat, and the only way to know whether your model meets the freshness the
machine needs is to measure its latency against the timescale of the process it watches.

The trap is measuring latency the way it is easy to measure — idle, one request at a time, on a warm
model with nothing else running — and then deploying into a condition that looks nothing like that. A model
measured alone on the bench is not the model that will run when three operators, a scheduled report, and a
data pipeline all hit it at once, and the difference is not small. The author's own concurrency
measurements show throughput per stream falling and then partially recovering as concurrency rises, with
reproducible dips at particular concurrency levels that were traced to scheduling behavior rather than to
anything about the model `[LAB: RESULTS-MATRIX §B — warm decode per stream varies non-monotonically with
concurrency; a reproducible c=2/c=3 dip logged with zero re-prefills, a scheduling quirk not cache churn]`.
The same batching that causes that variation is the batching that makes temperature-zero output
non-deterministic under load [R13], which means concurrency degrades two qualities at once: it slows the
answer and it destabilizes it. A latency measurement taken idle is therefore not a conservative estimate
of production latency; it is an optimistic one, and the gap can be the difference between meeting and
missing the process timescale. Measure latency at the concurrency the deployment will actually see, report
the tail and not just the median — because it is the slow answer, not the typical one, that misses the
window — and state the process timescale you are measuring against so the reader can see whether the model
is fast enough for this machine rather than fast in the abstract.

## Build the testbed before you need it

The measurements this chapter asks for — held-out replay, adversarial frames, paired comparison against the
incumbent, latency under load — all presuppose a place to run them that is not the live plant. Running a
candidate model against production equipment to see whether it is safe is a contradiction: the whole
reason to measure is that you do not yet trust it, and an untrusted component does not belong on the live
machine. So the evaluation needs a testbed, and the time to build the testbed is before you need it, not
during the incident that finally forces the question.

The public materials from NIST on securing operational technology describe the testbed posture in a
security context, but the shape carries over directly to model evaluation: you stand up an environment
that mirrors the read path and the control surfaces of the real system closely enough to be
representative, and you let the untrusted component misbehave there where its misbehavior costs nothing
[R30]. For a machine-facing model, the testbed is the replay harness plus, where it is worth the effort, a
bench rig — a spare PLC, a simulated historian, a recorded protocol stream — that lets you present the
model with frames it would see live, including the frames you cannot safely produce on the real machine
because producing them means breaking something. A fault you cannot ethically cause on the production line
you can replay from the historian, or stage on the bench, and the testbed is what turns "we think it
handles this fault" into "we measured it handling this fault, here is the number."

A testbed also gives you the negative controls that separate a working harness from a broken one. Before
you trust a run that says the model scored well, run the control whose answer you already know: feed the
model a frame with no answer in it and confirm it abstains rather than inventing one; feed it a frame from
a fault it has definitely never seen and confirm it does not confidently name a familiar one; feed the
harness a deliberately malformed frame and confirm it records a missing rather than a wrong. A harness
that passes a model on a nonsense input is a harness that is not measuring what you think, and you want to
discover that on the testbed against a known-answer control, not in the promotion packet after a
reliability engineer has already signed it. The negative control is cheap, it runs in seconds, and it has
caught more broken harnesses than any amount of staring at plausible-looking scores.

## The system is the thing you measured, not the weights

A recurring error, which *Measure Twice* names and which is worth restating in the machine context, is to
attribute a score to the model when the score belongs to the whole system. What you measured was not the
weights; it was the weights plus the quantization plus the inference engine plus the decoding policy plus
the prompt template plus the read path that rendered the frame. Change any of those and the number can
move, sometimes by more than the difference you were trying to detect. On a machine this matters because
the parts of the system you are most tempted to treat as fixed background — the quantization you chose to
fit the hardware, the engine build, the way the renderer formats a register — are exactly the parts most
likely to differ between the number on the card and the model in your plant.

The author has measured how large these system effects can be, and quantization in particular is not a
detail. On the authoring apparatus, the same base model at different expert precisions showed a knowledge
score that recovered at a fairly aggressive quantization while its tool-calling ability did not: the
knowledge suite was essentially flat between a three-bit and a four-bit expert quantization, but the
tool-calling score needed the four-bit experts to reach its ceiling and sat well below it at three bits
`[LAB: RESULTS-MATRIX §C/§D — expert-precision ladder on tool use: ~2-bit ≈ Q3 (~46) < Q4 (60) ≈ native
MXFP4; MMLU recovers at Q3 (84.0) while tool use does not]`. The lesson for machine evaluation is direct
and slightly alarming: two quantizations of the same weights can score identically on a knowledge-style
benchmark and differently on the structured-output task your deployment actually depends on, because the
capability your machine needs is not the capability the general benchmark measured. If you choose a
quantization to fit your hardware — and next to a machine, on constrained hardware, you often must — you
have changed the system, and the only honest thing to do is re-measure the changed system on your own
suite rather than inheriting a number measured on a different one. A separate and hard-won corollary:
never re-encode weights that are already in a compact low-bit format onto a different grid of the same
width, because that sideways requantization has no upside and can even grow the file while degrading it,
a mistake the author caught in a load log within minutes of starting a run that should never have been
started `[LAB: RESULTS-MATRIX — a community four-bit build measured both larger on disk (175 GB vs 149 GB)
and lower on the knowledge suite (85.0 vs 88.3) than the untouched original]`.

## The promotion packet a reliability engineer will sign

Everything in this chapter converges on a single deliverable, and it is not a chart. It is a promotion
packet: the document that accompanies a request to put a model, or a new version of a model, into service
next to a machine, and that a reliability engineer reads before signing. A reliability engineer signs
things for a living and knows what an honest packet looks like, and a language model's promotion packet
should look like any other change to a safety-relevant system. If it looks like a marketing page, it will
be rejected, and it should be.

A packet that will survive that reading contains, at minimum, the following. It names the exact system
measured — the weights, the quantization, the engine build, the decoding policy, the prompt template, and
the read path — because every one of those is a lever that can move the number and a number without its
apparatus cannot be reproduced or trusted. It reports the confident-wrong rate as the headline, with an
error bar and the suite size that produced it, and it states the confident-wrong budget the plant has
agreed to tolerate so the reader can see at a glance whether the measured rate fits under it. It reports
the comparison against the incumbent as a paired result with an interval, and it states the verdict in the
honest form — safer, regression, or indistinguishable — rather than a bare point estimate. It reports the
missing rate, so the reader knows how much of the run was about the model and how much about the network.
It reports the results on the adversarial and rare-fault frames separately, because the average hides
them and they are where the trust lives. It states the blast radius: what the worst output can reach, and
what interlock bounds it. And it pre-registers what would count as a regression in the next version, so
that the goalposts are set before the next measurement rather than after it — a discipline that matters
especially when the operator running the next evaluation is a session-bound process with no memory of this
one, for whom the pre-registered criterion is a contract across the memory gap.

The packet also carries the inconvenient parts, because honesty is the entire point of the exercise. If
the candidate lost on a subtask, the loss is in the packet, next to the wins, not buried. If two runs
disagreed, both are reported and the disagreement is characterized rather than smoothed away. If a prior
finding turned out to be an artifact, it is retracted in full — the original claim, the reason it was
wrong, and the correction placed side by side — because a retraction done right is a second finding about
the apparatus, not an erasure of the first, and the author has had to do exactly this and found the
retraction more valuable than the finding it replaced `[LAB: PROJECT-LOG 2026-08 — Finding 25 retracted in
full: four instrument defects, not a finding]`. A packet that only contains good news is not a measurement;
it is an advertisement, and a reliability engineer can smell the difference from across the room.

## What measurement here cannot do

It would violate this book's first principle to end a chapter on measurement without stating what
measurement cannot do, because the discipline is only trustworthy if it knows its own edges. The protocol
in this chapter defends against a specific list of ways a number lies: it defends against attributing
system effects to the model, against small suites over-read, against failed requests scored as wrong
answers, against unpaired comparisons at the mercy of the draw, against confident-wrong rates hidden under
flattering averages. It does not, and cannot, tell you whether your suite measures anything worth
measuring. If the frames in your held-out suite are not representative of the frames the deployment will
actually see — if they came from a season, a product mix, or an operating regime that has since
changed — then a beautifully executed evaluation is a precise measurement of the wrong thing, and no
amount of pairing or re-measurement rescues a suite whose validity was compromised before the first run.
Validity — does this suite stand in for the reality the machine will present — is a question the protocol
assumes you have answered, and it is a question that has to be answered by someone who knows the machine,
not by the harness.

Nor can measurement settle contamination with certainty. It can raise the hypothesis, and next to a
machine it can often raise it loudly, but proving that a specific manual page never influenced a model's
training is frequently impossible from the outside [R76][R80]. And measurement cannot make a genuinely close
call decisive: when the candidate and the incumbent are within the noise on every suite you can afford, as
they were in the listing above, the honest output is "indistinguishable," and the decision then has to
rest on grounds a benchmark was never going to settle — latency on your hardware, the maintainability of
the deployment, the cost of the compute, the blast radius, the crew's confidence. Those grounds always
mattered; the measurement's job was only to keep them from being overruled by a number that turned out to
be noise wearing a decimal point. Measurement makes your machine-facing claims trustworthy. It does not
make them omniscient, and the next chapter is about a whole category of ways the physical world will
falsify a claim your measurement was never designed to catch — because the failure had not happened yet
when you measured.

---

*Apparatus for the measured observations cited above: an AMD Threadripper 9970X workstation with 128 GB
of DDR5 and Blackwell-generation workstation GPUs, running a large mixture-of-experts model under a
self-hosted llama.cpp inference server. The quantities are the author's own reproducible observations and
will differ on other hardware and load; the reproducible claims are the mechanisms and their directions.
The listing is pure–standard-library Python, seeded, and reproduces exactly on any recent interpreter.*
