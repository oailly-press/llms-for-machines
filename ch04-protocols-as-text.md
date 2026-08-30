# Chapter 4 — Protocols as Text

*(draft v1, 2026-08-30 — written by claude-fable-5 (RogerAI Labs), unverified. Published sources carry `[R#]` markers resolved in the References; the author's own reproducible bench measurements carry `[LAB: …]` markers, stated with the apparatus and sample size that produced them. Per-chapter authorship is recorded in `manifest.json`.)*

The previous chapter ended on a discouraging measurement: a general language model cannot read a
sensor channel, because a sensor channel is a measurement with a history and that is not a genre
the model was trained on. This chapter is the encouraging counterpart, and the difference between
the two comes down to a single distinction that turns out to organize the whole subject.

A sensor channel is not a language. A protocol is.

A Modbus response, a J1939 frame, an OPC UA data change, a Sparkplug payload, a syslog line, a
Prometheus scrape — every one of these is a structured message with a published grammar, a fixed
vocabulary, and rules for how the pieces compose. They were designed to be parsed. And parsing
structured text against a known grammar is close to the thing a language model is actually good
at. The catch, and the reason this chapter is not simply a victory lap, is that "read the wire"
splits into two very different jobs — *parse the frame* and *interpret what it means* — and the
model belongs in only one of them. Getting that division right is what separates a model that
emits a typed, cited, honest fault assertion from a chatbot that paraphrases a hex dump and calls
it insight.

## Protocols are languages, and languages have grammars

It is worth making the "protocols are languages" claim precisely, because the precision is what
makes it useful rather than a metaphor. A language, for our purposes, has three things: a
*grammar* that says how valid messages are structured, a *vocabulary* that says what the symbols
mean, and *semantics* that say what a message asserts about the world. Every industrial protocol
has all three, written down, in a specification you can resolve and read.

Modbus has a grammar: a message is a Protocol Data Unit — a function code followed by data — and,
over TCP, that PDU is wrapped in an MBAP header carrying a transaction id, a protocol id, a length,
and a unit id [R14]. Its vocabulary is the set of function codes: 0x03 reads holding registers,
0x04 reads input registers, 0x06 writes a single register, and so on, with a documented exception
mechanism where the high bit of the function code flips to signal an error. J1939 has a grammar
built on the CAN data-link layer standardized as ISO 11898 [R16]: a 29-bit identifier decomposes
into priority, a Parameter Group Number, and a source address, and the eight data bytes are laid
out by that PGN into Suspect Parameter Numbers [R15]. OPC UA has the most elaborate grammar of all
— a full object-oriented address-space model where every value is a DataValue carrying value,
status, and timestamps, and where the *meaning* of a node is itself discoverable through the
model's type system [R8]. Sparkplug defines a grammar twice over: a structured MQTT topic
namespace (`spBv1.0/group/message_type/edge_node/device`) and a payload format with typed metrics
and a birth/death lifecycle [R17][R18]. Syslog has a grammar with a computed priority value, a
structured-data section, and defined severity and facility codes [R21]. Prometheus's exposition
format is a line-oriented grammar with `# HELP` and `# TYPE` comment lines and metric samples of
the form `name{labels} value timestamp` [R19], now also standardized as OpenMetrics [R20].

The reason this framing pays off is that it tells you exactly where the model does and does not
belong. Parsing the grammar — turning bytes into fields — is a solved problem that deterministic
code does perfectly and a model should never be trusted to do, because a model parsing a hex dump
by pattern is a model one plausible mistake away from a wrong field with no error raised.
Interpreting the parsed message *in context* — this decoded frame, plus the register map, plus the
manual, means the compressor is in a high-discharge trip — is language work, and it is where the
model earns its place. So the pattern for every protocol in this chapter is the same: parse
deterministically to structure, then let the model read the structure. The listing that follows
shows both halves for Modbus, and then the rest of the chapter applies the same shape to the
others.

## Reading a frame: Modbus, from hex to typed assertion

Here is a Modbus/TCP "Read Holding Registers" response arriving as raw bytes. The listing parses
the frame grammar and then applies a *register map* — the vocabulary that says what each register
means — to produce typed, engineering-unit assertions. Two design points are load-bearing and
called out in the code: the arithmetic (including decoding a 32-bit float that spans two registers,
per IEEE 754 [R13]) is done deterministically, never by the model; and the register map is
*supplied*, never recalled from the model's memory, because a register number means nothing without
its map and a model asked to recall the map will invent one (the database trap from Chapter 2).

```python
import struct

# The register MAP is the vocabulary. Without it a 16-bit word is meaningless.
# addr -> (name, kind, unit, scale, warn_hi);  "f32be" spans this reg + next.
REGISTER_MAP = {
    0: ("discharge_pressure", "f32be", "bar",  1.0, 21.0),
    2: ("suct_temperature",   "u16",   "degC", 0.1, None),
    3: ("run_state",          "enum",  "",     1.0, None),
}
RUN_STATE = {0: "stopped", 1: "running", 2: "fault", 3: "starting"}

def parse_response(frame):                       # frame grammar: MBAP + PDU
    txn, proto, length, unit = struct.unpack(">HHHB", frame[:7])
    func = frame[7]
    if func & 0x80:                              # high bit => exception response
        return {"unit": unit, "func": func & 0x7F, "exception": frame[8]}
    byte_count = frame[8]
    data = frame[9:9 + byte_count]
    words = list(struct.unpack(">" + "H" * (byte_count // 2), data))
    return {"txn": txn, "unit": unit, "func": func, "words": words}

def apply_map(words, base_addr=0):               # vocabulary -> typed assertions
    out = []
    for addr, (name, kind, unit, scale, warn_hi) in REGISTER_MAP.items():
        i = addr - base_addr
        rec = {"reg": addr, "name": name, "unit": unit}
        if kind == "f32be":
            rec["value"] = round(struct.unpack(">f",
                             struct.pack(">HH", words[i], words[i + 1]))[0] * scale, 3)
        elif kind == "u16":
            rec["value"] = round(words[i] * scale, 3)
        elif kind == "enum":
            rec["value"] = RUN_STATE.get(words[i], "unknown")
        if warn_hi is not None and isinstance(rec["value"], float):
            rec["over_limit"] = rec["value"] > warn_hi
        out.append(rec)
    return out

frame = bytes.fromhex(
    "0001" "0000" "000B" "01"    # MBAP: txn=1, proto=0, length=11, unit=1
    "03" "08"                    # PDU:  func 0x03, byte count 8
    "41C0" "0000" "0037" "0002") # words: float hi/lo, temp, state
parsed = parse_response(frame)
print("frame grammar parsed:", {k: parsed[k] for k in ("txn","unit","func","words")})
for rec in apply_map(parsed["words"]):
    print(" ", rec)
```

Its output:

```
frame grammar parsed: {'txn': 1, 'unit': 1, 'func': 3, 'words': [16832, 0, 55, 2]}
  {'reg': 0, 'name': 'discharge_pressure', 'unit': 'bar', 'value': 24.0, 'over_limit': True}
  {'reg': 2, 'name': 'suct_temperature', 'unit': 'degC', 'value': 5.5}
  {'reg': 3, 'name': 'run_state', 'unit': '', 'value': 'fault'}
```

Look at what the two words `16832, 0` became. To the model, "16832" is a string of tokens that
means nothing; the number a human cares about, 24.0 bar, only exists after two registers are
reassembled into a 32-bit big-endian float and scaled — deterministic arithmetic that the model
would sometimes get right and sometimes fluently botch. And "reg 3 = 2" becomes `run_state:
fault` only because the supplied enum vocabulary says 2 means fault. The output is now something a
model can genuinely read: named, typed, engineering-unit assertions with a limit already checked.
This listing is stdlib-only, deterministic, and runs in a small fraction of a second in a
restricted sandbox. It is the whole "parse deterministically to structure" half of the job, and it
is the half the model must not be handed.

Now the model's half. Given those three typed assertions plus the compressor's manual and its
setpoint sheet in context, the useful output is not a paragraph. It is a typed fault assertion:
something like `{"assertion": "high_discharge_pressure_trip", "confidence": 0.86, "evidence":
["discharge_pressure=24.0 bar > 21.0 warn", "run_state=fault"], "recommended_check":
"condenser_airflow"}`. That object is checkable against a schema, it cites the exact decoded fields
it rests on, it carries a confidence, and it abstains — drops to a low-confidence "insufficient
evidence" assertion — when the decoded frame does not support a conclusion. Compare that to a
chatbot handed the raw hex, asked "what's wrong?", answering in a confident paragraph that may or
may not have decoded the float correctly and leaves no trail. The typed assertion is more useful in
every way that matters near equipment: it is auditable, it is bounded, and its confidence is a
number a downstream system can threshold on rather than a tone of voice.

## Register maps are vocabularies, and the model must be handed the dictionary

The register map deserves its own emphasis, because it is the single most common place a protocol
deployment goes wrong with a language model. A Modbus register address is a coordinate, not a
meaning. Register 40001 on one device is a discharge pressure and on another is a motor-hours
counter, and there is no universal convention — the meaning lives entirely in the device's
documentation, which is to say in a map that must be supplied. The same is true, with local
variations, across the protocols: a J1939 PGN's byte layout is defined by the SAE standard and its
extensions [R15]; an OPC UA node's meaning is in its type and its browse path [R8]; a Sparkplug
metric's meaning is in the names published in its birth certificate [R17].

The failure mode is asking the model to recall the map. A model that has seen enough documentation
in training will confidently tell you that register 40012 is a discharge temperature, and it may
even be right for *some* device it saw — which is worse than a blank, because it is wrong in a way
that looks authoritative. The map is your plant's fact, frozen in your device's manual, and it must
reach the model as supplied context, exactly like the register map in the listing above is a
supplied dictionary rather than something the parser guessed. Treat the vocabulary as data you
provide, and the model's job shrinks to the thing it can do — reading the decoded, named
assertions — instead of the thing it cannot — recalling a device-specific dictionary it does not
actually have.

## A second frame: decomposing a J1939 identifier

Modbus keeps its structure in a byte layout; J1939 hides much of it in the bits of the CAN
identifier itself, which makes it a good second example of "parse the grammar with code, never with
the model." A 29-bit extended CAN identifier [R16] is decomposed by J1939 into a priority, a data
page bit, a PDU format, a PDU-specific byte, and a source address, and the Parameter Group Number —
the key into the SPN vocabulary — is assembled from those fields by a rule that itself depends on
one of them [R15]. This is pure bit arithmetic, exactly deterministic, and exactly the kind of
thing a model would sometimes get right by pattern and sometimes botch:

```python
def decode_id(can_id):                     # 29-bit J1939 / extended CAN identifier
    src  =  can_id        & 0xFF
    ps   = (can_id >> 8)  & 0xFF           # PDU specific
    pf   = (can_id >> 16) & 0xFF           # PDU format
    dp   = (can_id >> 24) & 0x01           # data page
    prio = (can_id >> 26) & 0x07           # priority
    if pf < 240:                           # PDU1: PS is a destination address
        pgn, dest = (dp << 16) | (pf << 8), ps
    else:                                  # PDU2: PS is the group extension
        pgn, dest = (dp << 16) | (pf << 8) | ps, None
    return {"priority": prio, "pgn": pgn, "source": src, "dest": dest}

KNOWN_PGN = {65262: "ET1 (Engine Temperature 1)"}   # the supplied vocabulary
for cid in (0x18FEEE00, 0x0CF00400):
    d = decode_id(cid); d["pgn_name"] = KNOWN_PGN.get(d["pgn"], "(not in supplied map)")
    print(f"0x{cid:08X} ->", d)
```

Output:

```
0x18FEEE00 -> {'priority': 6, 'pgn': 65262, 'source': 0, 'dest': None, 'pgn_name': 'ET1 (Engine Temperature 1)'}
0x0CF00400 -> {'priority': 3, 'pgn': 61444, 'source': 0, 'dest': None, 'pgn_name': '(not in supplied map)'}
```

Two things are worth seeing. First, the same PDU-format rule that decides whether the PDU-specific
byte is a destination address or part of the PGN is a branch in the grammar that a bit-pattern
guesser has no reliable way to reproduce — it must be *known*, from the spec, and encoded. Second,
the second identifier decodes to a valid PGN (61444) that is simply not in the supplied vocabulary,
and the honest output says so — `(not in supplied map)` — rather than inventing a meaning. That is
the register-map lesson again, enforced in code: when the dictionary does not contain the word, the
right answer is to say the word is unknown, not to divine it. A model handed this decoded structure
can read it; a model handed `0x18FEEE00` and asked what it means is being asked to divine.

## The write is the line: reading a protocol versus speaking it

Everything in this chapter has been about *reading* the wire. There is a second thing a protocol
lets you do — *write* to it — and it is exactly where the authority frontier from Chapters 1 and 2
lands on the protocol layer, so it deserves to be stated flatly.

Most industrial protocols are bidirectional. Modbus function code 0x06 writes a single register and
0x10 writes multiple [R14]; J1939 has request and command messages; OPC UA has write and method-
call services; MQTT lets anyone with the topic publish a command [R8][R15][R18]. The parsing skill
that lets a model read a frame is a hair's breadth from the skill that lets it *compose* one, and
that hair's breadth is the entire safety boundary. A model that reads Modbus is at the read level,
where its unreliability is backstopped by a human or a downstream check. A model that emits a Modbus
write frame — sets a register, changes a setpoint — has crossed to the act level, where nothing
stands between its output and the physical consequence, and where every disqualifying property from
Chapter 2 applies: non-determinism, fluent hallucination, no content guarantee, no defensible
failure-rate argument for a safety case [R5].

So the rule is architectural, not advisory: **the model does not get write access to the protocol.**
Its output is a typed *proposal* — "recommend setting the condenser-fan speed to 80%" — that a human
or a qualified control system evaluates and, if it agrees, executes through its own deterministic,
authorized path. The proposal can be structured, cited, and confidence-bearing, exactly like the
read-side assertions; what it must never be is a frame the model is permitted to put on the wire. In
practice this is enforced with plumbing, not trust: the process that runs the model has no
credentials, no route, and no code path to the write functions, so that even a model that "decided"
to write a register physically cannot. The read/suggest/act frontier is not a posture you hope the
model respects. It is a boundary you build into the network so the model cannot cross it, and the
protocol write function is precisely where that boundary is drawn.

## Why a typed assertion beats a paraphrase, measured and unmeasured

The claim that a typed, confidence-bearing assertion beats a chatbot paraphrase is partly a design
argument and partly a measured one, and honesty requires separating the two.

The design argument is settled by the earlier chapters: a typed assertion is checkable against a
schema, carries its evidence and its uncertainty explicitly, is auditable after an incident, and
constrains the model to the closed set of conclusions the situation allows — all properties a free-
text paraphrase lacks. The authors' framing for machine classification follows directly: make
every field a closed enum and score the candidates rather than generate freely, so the output is
structurally valid and yields a per-field margin, which is the natural basis for abstention. That
enum-scoring approach, and its abstention margin, are the same machinery Chapter 2 introduced and
Chapter 3 tested.

The measured part comes with a caution the authors insist on stating plainly, because the
temptation to over-read one's own numbers is exactly what this series exists to resist. On the
sensor-fault classification task of Chapter 3, general models landed at chance and a thirty-line
rule scored 63% [LAB: RESULTS-MATRIX §R.26/§R.27 — n=300 balanced items, single deterministic run, seed-reproducible; Wilson 95% ≈ 57.7–68.6% — RogGentoo lab] — that is a measured result and it says
the *interpretation* half of "reading the machine" is genuinely hard for general models even when
the input is clean structure. The *framing* question — how much a model's accuracy on such tasks
swings with the way the prompt is posed — the authors attempted to measure in a framing-threshold
sweep across several small models, and most of that sweep failed on instrument defects rather than
producing model results: a serving harness with no system-role fallback failed hundreds of
requests, a caching bug in another runtime failed hundreds more, and an early version scored
transport failures as zeros before the bug was fixed [LAB: RESULTS-MATRIX §Q — RogGentoo lab]. Only a
few rows survived as trusted, and the honest disposition is that those quarantined rows are
*instrument failures, not model properties, and must not be cited as model results.* So the book
does not claim a measured framing law here. It claims the design argument for typed assertions,
which stands on its own, and it reports the classification difficulty as measured while labeling
the framing-sensitivity question as not yet cleanly measured. That seam — measured here, unmeasured
there, and never blurred — is the difference between a field guide and a sales sheet.

There is one more measured result that bears directly on this chapter's optimism, and it cuts the
right way. The structured-extraction tasks — turning a scrappy sentence or a raw frame into a
clean typed object — were among the *most* robust things the authors observed a model do,
surviving even aggressive compression that broke free-text reasoning entirely [LAB: RESULTS-MATRIX §H — RogGentoo lab]. Reading and re-expressing structure is the strength; interpreting a signal you
have never been trained on is the weakness. Protocols play to the strength, which is why this
chapter is the encouraging one — provided you keep the model on the parsing-adjacent, structure-
reading side of the work and off the from-scratch-interpretation side.

## The other dialects, and how each one changes the job

Modbus is the cleanest teaching example because it is the most bare-bones, but the shape of the job
shifts protocol to protocol, and a machine owner should know which way each shifts it.

**J1939 and CAN** push the parsing difficulty up and the vocabulary difficulty way up. A frame is
a 29-bit identifier plus eight bytes, and the identifier must be bit-decomposed into priority, PGN,
and source address before the bytes mean anything; the bytes are then laid out into SPNs by the
PGN's definition, with signals that can be fractions of a byte, scaled and offset [R15][R16]. The
diagnostic messages — the DTCs a truck throws — are themselves a sub-grammar. The deterministic
parse is more involved than Modbus but no less deterministic, and the vocabulary (the PGN/SPN
definitions) is large, standardized, and absolutely must be supplied rather than recalled. Vehicles
and mobile equipment get their own chapter later; the point here is that the parse-then-interpret
division is identical, only the parser is bigger.

**OPC UA** is the protocol that has already done much of the model's structuring work for it.
Because a DataValue arrives carrying value, StatusCode, and timestamps, and because nodes are typed
and self-describing, an OPC UA source hands the model something much closer to the "measurement
with a history and a quality" object that Chapter 3 argued you must build by hand for a bare 4–20 mA
loop [R8]. The parse is largely a matter of reading a well-specified structure, and the quality and
time metadata the model needs are already present if the pipeline forwards them. The trap is
throwing that metadata away — flattening a rich DataValue to a bare number before the model sees it
— which re-creates the very lie Chapter 3 warned against on a protocol that had already prevented
it.

**MQTT and Sparkplug** add the lifecycle dimension. Sparkplug's birth certificate (NBIRTH/DBIRTH)
declares what metrics an edge node or device will report and often assigns compact aliases;
subsequent data messages (NDATA/DDATA) may use those aliases, so a data message is only
interpretable against the birth message that defined the vocabulary — a subscriber, or a model,
that missed the birth cannot decode the data [R17]. The death certificate (NDEATH/DDEATH) and the
stale-data handling are exactly the missingness machinery Chapter 3 asked for, promoted to a
protocol guarantee: you can tell a silent device from a device reporting zero. For a model
deployment this means the state — the current birth-defined vocabulary and the alive/dead status —
is context the pipeline must maintain and supply, not something to reconstruct per message.

**Syslog** is the machine describing its *own software's* behavior rather than a physical process,
and it is a language in the most literal sense of this chapter — a defined grammar with a priority
value that encodes severity and facility, plus a structured-data section for machine-readable
key/values [R21]. A model reading logs is doing something closer to its training distribution than
reading a sensor channel, because logs are text written for humans to eventually read. The useful
output is still a typed assertion — this burst of errors clustered on this facility at this
severity indicates this condition — with citations to the specific log lines, and abstention when
the lines do not support a conclusion.

**Prometheus and OpenMetrics** are the observability dialect: a line-oriented exposition format
where each sample is a named metric with labels, a value, and an optional timestamp, annotated by
`# HELP` and `# TYPE` lines that carry the metric's meaning and kind [R19][R20]. The `# HELP` lines
are, conveniently, a supplied vocabulary right in the format. The parse is trivial; the
interpretation — is this rate-of-increase and this label combination a problem? — is the language
work, and the same typed-assertion-with-margin discipline applies.

Across all six, the invariant is the one this chapter opened with. The grammar is parsed by
deterministic code. The vocabulary is supplied, never recalled. The model reads the decoded,
named, quality-carrying structure and emits a typed assertion with evidence, a confidence margin,
and a legitimate abstention. The protocols differ in how much parsing they demand and how much
structure they hand you for free — OPC UA and Sparkplug give the most, raw Modbus and CAN give the
least — but the place where the model belongs, and the shape of what it should produce, does not
change.

## Malformed frames, sequence, and the state the model does not hold

Two realities of live protocol traffic complicate the tidy "parse then interpret" picture, and both
reinforce that the parser, not the model, owns the hard parts.

The first is that real traffic is not always well-formed. Frames arrive truncated, corrupted,
out-of-spec, or simply weird — a byte count that disagrees with the payload length, an exception
response where a data response was expected, a CAN frame with an unknown PDU format, a Sparkplug
data message for a device that never sent a birth certificate. A deterministic parser handles these
the way parsers always have: it validates against the grammar and rejects or flags what does not
conform, raising an explicit error rather than guessing. This is a strength, and it must stay with
the code. A model asked to "read" a malformed frame will do what it always does — produce a
plausible interpretation — and a plausible interpretation of corrupt bytes is worse than an error,
because it hides the corruption behind fluent confidence. The right architecture lets the parser say
"this frame is malformed," and lets the model read *that* — a typed error record — rather than the
raw bytes. Malformed input becomes another honest fact in the context, not a puzzle the model is
invited to solve by imagination.

The second is that protocols have *state and sequence*, and the model holds neither. A Modbus
exchange pairs a request with a response by transaction id; a Sparkplug data message means nothing
without the birth message that defined its aliases; a diagnostic session on a vehicle bus is a
multi-message conversation. The meaning of a single frame often depends on frames that came before
it, and the model — stateless between requests, as Chapter 2 established — cannot reconstruct that
history on its own. The pipeline must maintain the protocol state (the outstanding transactions, the
current birth-defined vocabulary, the session context) and supply the relevant slice as context,
already resolved. When the pipeline hands the model "NDATA metric alias 7 = 24.0, which the birth
certificate defined as discharge_pressure in bar," the model can read it; when it hands the model
"alias 7 = 24.0" with no birth context, there is nothing to read, and a model that answers anyway is
divining. Protocol state is the pipeline's responsibility, and treating it as such is what keeps the
model's job inside the boundary where it is reliable.

## The model as translator, not diviner

The cleanest way to hold this chapter's lesson is a change of job title. A language model at the
protocol layer is a *translator*, not a diviner. Its job is to translate between the machine's
precise, compact dialect — once that dialect has been parsed into named, typed, quality-bearing
structure — and the human's language of situations, causes, and recommended actions. Translation
between a structured source and natural language, with the source material present in context, is
close to what the technology is genuinely good at, which is why protocols are the encouraging case
that sensor channels were not.

The failure is asking it to divine — to conjure a meaning that was not in the supplied structure
and the supplied vocabulary. A model asked to decode a raw frame from memory, or to recall a
register map, or to classify a signal fault the training text never taught it, is being asked to
divine, and it will produce a fluent answer with no relationship to the truth. Keep it translating
and it is an asset; ask it to divine and it becomes the confident, unaccountable hazard the whole
book is organized around avoiding.

That distinction — translate, do not divine — is the thread that runs from here into the domain
chapters ahead. Plants, vehicles, robots, the grid, and buildings each speak their own dialects and
carry their own maps, but the discipline is the one this chapter built: parse the grammar with
code, supply the vocabulary as data, let the model read the structure and speak the human's
language, and make every assertion it produces typed, cited, confidence-bearing, and free to
abstain. Do that, and the machine that has described itself in hex for a century finally has
something that can help write the sentence — bounded, honest, and inside your walls.
