# HESF — Public Technical Reference

*Version 0.1 · public architecture subset*

*This is the public face of a larger internal reference. HESF (Hierarchical Emergence
Simulation Framework) is a studio engine; DIS is the first game built on it. The
engine knows nothing about the game, and the game knows nothing about the display —
the boundaries between them are serialized bytes.*

---

## 1. What this is

This document describes the **machine**: the protocol stack, the on-disk format, the
diagnostic discipline, and the design laws that hold them in place. It is a faithful
subset of a much larger working reference that lives in a private repository. The
private document covers the same architecture in full, plus the game's content
systems and mechanics — none of which appear here.

Where this reference is silent, assume the silence is deliberate. A gap is not an
oversight; it is the line between the engine (public) and the game (private). See
§9 for an honest account of what has been left out.

The rule for everything below: **concepts, not private schemas.** You will find the
*shape* of the frame header and the *shape* of the on-disk envelope, because those
are protocol contracts any second implementation would need. You will not find the
field-level layout of any content registry, because that is the game.

---

## 2. The thesis

**HESF is a protocol stack that executes game content as its application payload.**
It is not a game engine in the usual sense — there is no scene graph, no entity
component system, no gameplay code at the center. There is a byte protocol, a record
store, an interpreter, and a set of renderers that are, formally, terminals.

The load-bearing idea: **the data files are the program, and the engine is an
interpreter over a fixed-schema record store.** Content is authored into binary
`.dat` files. The engine reads a record, looks at a type tag, and dispatches on it
without knowing whether it is running a conversation, an encounter, or a navigation
node. Add new content and you have added new data, not new code. The interpreter is
content-blind by construction; the same engine drives many kinds of content.

The record store is deliberately humble: typed accessors over a fixed schema, no
query language, no object-relational layer, no reflection. Reading a field is a
primitive call that returns a value. That humility is the point — it is what makes
the store auditable, byte-deterministic, and safe to version for the life of the
project.

**Z-machine lineage.** This is the Infocom Z-machine idea, moved forward forty
years. Infocom shipped a small virtual machine and separate *story files*; the
machine interpreted the story, and the same machine ran every game. The story file
was the program; the interpreter was game-blind and portable. HESF takes the same
split — a content-blind interpreter over a versioned, self-contained data format —
and adds a modern wire protocol, an always-on diagnostic journal, and hard schema-
versioning so that content and engine can evolve independently and old data keeps
loading. The lineage is intentional: a data-driven interpreter is a proven way to
build something that outlives any single title.

---

## 3. The layer model

The engine is four assemblies with a strict, one-way dependency order. Each is a
protocol layer; each depends only inward.

```
Primitives  ←  Transport  ←  Core  ←  Gates
   (each arrow = "is referenced by"; dependencies point one way only)
```

**Primitives** — the inert atoms. One job each: resolution math, identity, state
accumulation, boolean evaluation, the manual byte reader/writer, the neutral wire
shapes. It references no other engine assembly and no third-party packages. This is
the layer that touches bytes and nothing above it does.

**Transport** — the movement layer: the frame format, the tag system, the shape-
blind bus, the wire serializers/readers, protocol validation, and the always-on
boundary journal. It references Primitives. This is the "always-logged" tier — every
crossing here is recorded.

**Core** — compositions: anything built of two or more primitives plus logic (event
orchestration, the resolver driver, the presentation handler). It references
Primitives and Transport. Transport never references Core — composition depends on
movement, not the reverse.

**Gates** — the proving ground: stop-gate rigs, fixtures, stubs, and the gate-runner.
It references everything and is referenced by nothing, and it never ships as engine.
A brick is not "done" until its gate is green.

**Topology: a hub-and-spoke web, not a tower.** Primitives is the hub — the invariant
center, not a foundation floor. The engine runtime and the content editor are peer
anchors on the rim. The game is the arc between them, crossing the on-disk data
membrane. The two anchors both depend inward on the hub and talk to each other only
across that membrane — never through a direct anchor-to-anchor dependency. A direct
edge between the anchors would flatten the web back into a tower and is treated as a
defect.

---

## 4. The wire

The boundary between the simulation and any display is **a byte contract, not a C#
interface.** The simulation emits framed bytes; a renderer consumes them. That is
the entire seam.

**A self-describing, append-only frame header.** Every packet is wrapped in a small
header modeled on IP's "header-length" trick:

```
[ header-length ]   total header bytes, this field included
[ payload-length ]  little-endian
[ path-tag ]        little-endian — the exact-match dispatch discriminator
[ payload …    ]    opaque bytes
```

The header is **self-describing**: a reader locates the payload at *offset +
header-length*, never at a hard-coded constant. So the header can grow — a future
field is appended, and an older reader simply skips the bytes it does not recognize
and still finds the payload. New senders and old readers coexist. Little-endian is
written explicitly, so framing is byte-deterministic on any host. Only *structural*
faults are loud (truncation, overrun, a reserved-zero tag); an unrecognized path-tag
is not an error — the frame length carries a blind walker safely past it.

**The path-tag is the type system.** Every packet type is one unsigned integer
constant, dispatched by **exact match** — never by range test. Tag assignments are
**append-only and never recycled**: a number, once meaning something, means that
forever. Blocks of the number space are partitioned by owner for documentation and
assignment-time sanity checks, but the block boundary is never a routing predicate.

**Renderers are terminals; the engine is the mainframe.** A renderer is a peripheral
that consumes the byte stream and holds no game flow. Swapping the surface is a
single registration: put a different consumer on the display bus. The same byte
stream today drives named driver targets — a console text driver, a desktop GUI
driver, and a binary session-log driver — simultaneously, none of them referencing
the simulation or each other. A future game-engine driver is the same job done
again: read the frames, draw the screen, send input back. Nothing upstream changes.

**Input is protocol, in both directions.** The return channel is itself framed
packets: the display sends back *which node* and *which choice* — an index, never
flow. That untrusted input is validated at the boundary before the simulation
derives anything from it: correlation (does this answer the node actually on screen?)
and range (is the choice within bounds?), each a **loud** rejection that is never
silently applied. A version handshake gates the whole conversation and does not
negotiate down — a mismatch fails loud.

*(Field-level packet layouts beyond the frame-header concept are not public.)*

---

## 5. The store

At rest, content is a **self-contained binary module**: a fixed header, an embedded
schema section, and a hand-written FlatBuffers payload, all wrapped in an integrity
envelope. There is **no JSON anywhere** — not on the wire, not in content files, not
as a debug fallback. Everything is compact, byte-deterministic binary, hand-written
against a low-level builder and a manual table walk (no code generation).

**The module shape (conceptual).**

```
[ header      ]  magic + module id + version + sentinel
[ schema section ]  the module's own field description, embedded
[ payload     ]  the FlatBuffers record data
```

wrapped, for durable files, in an integrity envelope:

```
[ version byte ][ payload ][ CRC32 ]
```

On the wire, the transport frame (§4) is already the envelope, so wire messages carry
the raw payload without the durable header — the same payload builder, a different
outer wrapper per context.

**Append-only field fences.** A record is read through a table that maps *field
index → byte offset*. The index is the field's identity. Reorder two fields and every
previously written file decodes one field's bytes as another's — silent corruption,
not a load error. Delete a field and every later field shifts. So the discipline is
absolute: **append at the end; never reorder, never delete, never recycle.** A field
index is a "fence" — a permanent marker. Reserved-but-unused indices are claimed in
advance by comment so nothing else can take them. Append-only *is* the versioning
strategy: an old reader that does not know a new top-index field simply finds it
absent and substitutes the schema default; a new reader reads old files because the
lower indices never moved.

**Endpoint-resident schemas; skip, never parse.** The schema travels inside the
module so the format is self-describing, but the reading endpoint carries its own
authoritative schema and uses it to walk the payload. It **skips** the embedded schema
section rather than parsing and trusting it — the endpoint's schema is the source of
truth. The format stays self-describing for tooling without making a reader dependent
on bytes it did not author.

**Integrity and validation discipline.** Every durable file is CRC32-wrapped; the
checksum covers the version byte and the payload, so a corrupted version byte is
caught *before* any reader is chosen. A read verifies the checksum first and fails
loud on mismatch — a corrupt or hand-edited file never reaches the interpreter as
"successfully parsed wrong data." Schema versioning is a three-way decision on load:
a *current* version reads normally; a *known-older* version is read, migrated
forward, and rewritten (old readers are retained as migration infrastructure, never
deleted); an *unknown or impossible* version **fails loud and throws** — a version
that never existed is a tamper or corruption signal, and that path is permanent. It
is the second line of integrity defense behind the checksum.

*(The field-level layout of any specific content registry is not public.)*

---

## 6. The design laws

A short constitution governs the whole tree. Some laws are general engineering
standards; some are architectural; one project-specific law is included for its
origin story, which is genuinely instructive. (One further law is the game's
founding creative statement and is withheld as game content — see §9.)

**Law 1 — A class owns its own state.** Only a class's own methods change its own
data. If something must change elsewhere, send a message and let the owner decide.
This keeps the responsibility for any piece of state singular and findable, and kills
whole categories of action-at-a-distance bugs.

**Law 2 — Boundaries communicate with messages; interiors communicate with methods.**
Cross-system and cross-boundary traffic goes through defined contracts — events,
messages, framed bytes — not direct field access. Inside a subsystem, ordinary method
calls are fine. Spend the heavier discipline where it pays (at the seams); keep the
interior readable. At assembly scale this reads as *engine-permissive, membrane-
strict*: the interior knows its residents concretely; the exported surface is minimal
and every inclusion carries a burden of proof.

**Law 3 — Command the things you own; influence the things you don't.** Directly-owned
actors take commands; simulated agents take *suggestions* and decide for themselves.
The origin is a parable the team calls **Joe and the Sticks**: Joe is hungry, food is
two steps away, but he is told "get sticks" — so Joe goes and gets sticks, and
starves. Command-and-control over a simulated agent produces literal obedience at the
cost of the agent's welfare. The inversion is the law: agents surface their needs
with urgency, and an order is *evaluated against* the agent's state rather than
executed blindly over it. It is a general principle for any system where the thing
being directed has its own internal model.

**Law 4 — State is saveable at any tick boundary.** At every tick boundary, complete
state is serializable and resumable. Pause-save-resume is a first-class operation,
not a bolt-on; every tick boundary is a valid save point. The frame-commit signal on
the wire is exactly such a boundary.

**Law 5 — New content is data, not code.** New content is added by authoring data,
not by editing source. Code reads the data and behaves accordingly. This is what
enables expansion-without-refactoring across the life of the project, and what makes
the engine reusable across future titles: change the content, keep the machine.

**Law 6 — Build for the scenes you are not building yet.** Every class, interface,
and abstraction should still work unchanged when modes that do not exist yet are
added later. A decision that only serves today's single case is the wrong decision.
A naming corollary sharpens this: **a name names the class it serves, at the
generality the design builds for — never the instance it currently happens to be the
only one of.** A name scoped to today's single case is the instance-shape made
permanent in the source; it is the cheapest place to catch an over-specific design —
at naming time, before instance-shaped code exists.

**Law 8 — Simulation knows nothing about display; display knows nothing about
simulation.** The core — logic, state, rules, decision-making — never references any
rendering technology, UI framework, or output format. A text console, a desktop GUI,
a binary log, and a headless test runner are all valid outputs from one simulation,
and the boundary between them is bytes, not a C# interface. A consequence worth
stating: the text driver is a **permanent debugging tool**, not throwaway scaffolding
— at any point you can switch to it and read the simulation's behavior in plain text,
with no visual noise.

*(Laws 1, 2, 5, and 6 are general engineering standards; Law 8 is architectural;
Laws 3 and 4 are project-specific but stated here in their general form. Law 7 is the
game's founding creative statement — withheld.)*

---

## 7. Replay and diagnostics

The engine ships with its own protocol analyzer built in. Every crossing of every
boundary is journaled — **always on, both directions, full payload, no off switch and
no capture-depth dial.** The journal is not a debug feature you enable when a bug
appears; it is default structural infrastructure, because the rare bug you need it
for is exactly the one that would have had it turned off.

**Journal-at-intake.** A frame is recorded the moment it enters the boundary,
*before* any delivery side effect runs — including frames that are then rejected. The
disposition (delivered, dropped, rejected-loud) rides in the record. Unframeable
bytes are captured too, with the fault noted, so malformed traffic does not simply
evaporate.

**The seed lives in the header.** The journal's session header carries the
determinism anchor — the random seed, recorded once — plus the checksums of the exact
content files the session ran against. That is enough to answer "what produced this
log" precisely.

**Replay as debugging.** Because the seed and every inbound frame are on record, a
logged session **replays byte-for-byte**: feed the logged inbound frames back through
the same walker with the random source reconstructed from the logged seed, and the
run reproduces exactly. This collapses the most expensive part of debugging — the
elimination phase, proving whether a fault is in game logic or in the engine — into a
lookup. Roughly nine bugs in ten are content bugs; the rare engine bug hides behind
the common case, and an always-on journal of exact values at every crossing is what
drags it into the light. Full-payload capture (not header-only) is what makes verbatim
replay possible, which is why there is no capture-depth dial.

**A field-diagnostics journal folder.** For diagnosing a build in the wild, a run can
additionally emit its full-fidelity journal into a dedicated field-diagnostics folder
beside the executable — a self-creating drop that a tester can send back for
byte-exact replay. It is opt-in and off by default.

---

## 8. Why the discipline

This engine is built by one developer working with AI coding assistants, and the
discipline in this document is a direct response to that working model.

An AI assistant is trained on a vast corpus of ordinary code, and its default pull is
toward the *average* of that corpus. Ask for a data reader and it will reach for a
generic serializer framework; ask for a boundary and it will reach for a shared
mutable object; ask it to "clean up" a deliberately unusual pattern and it will
regress it toward the common shape. None of that is wrong in general — it is wrong
*here*, because the whole value of this architecture is in the places it deliberately
departs from the average. Left unchecked, that pull quietly erodes the very
properties the design exists to guarantee.

The laws (§6) and the decision records behind them are the fence against that erosion.
They exist so that an architectural decision, once made, **stops being re-litigated
every session** — by the next human reader or the next AI one. Each significant
decision is written down *with the alternatives that were rejected and why*, precisely
so that a well-meaning "simplification" that misses the point can be recognized as
such before it lands. The rule of thumb is stated plainly in the private records:
before refactoring any pattern here that looks unusual, read why it is that way. Most
of the time the unusual shape is load-bearing.

This is what lets a solo developer hold a large, unusual architecture stable across
hundreds of sessions with a collaborator that has no memory between them: the
constraints are written into the tree, and the gates prove they still hold.

---

## 9. What's deliberately absent

This reference covers the machine, not the game. Withheld — and living only in the
private repository — are: the content systems (how questions, events, and encounters
are authored and wired); the character model and its mechanics (the hidden state the
engine tracks, how it is scored, and how it feeds outcomes); several whole design
layers (personnel, vehicles and augmentation, exploration and combat subsystems); the
game's creative founding statement and tone; and the specific field-level layout of
every content registry. The boundary mapper that sits between the game and the display
is mentioned as a concept — it exists, and by construction the display-bound shape has
no field able to carry the game's private state — but *what* it withholds is not
described here. If you are looking for how the game works, this is the wrong document
by design; this one covers only the engine it runs on.

---

*Public reference, v0.1. It reflects the architecture as a subset of a private working
reference. Where a second implementation would need a contract to interoperate — the
frame header, the on-disk envelope, the versioning discipline — it is here; where a
detail is the game rather than the machine, it is not.*
