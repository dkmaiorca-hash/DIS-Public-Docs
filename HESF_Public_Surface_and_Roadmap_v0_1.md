# HESF — Engine Surface & Roadmap

*Public reference. Names and purposes only — signatures, parameters, and content schemas live in the private repo.*
*HESF is the studio engine. DIS (Dwarves in Space) is the first game built on it. The engine knows nothing about the game; the game knows nothing about the display. The boundary between them is serialized bytes.*

---

## The Layer Model

```
DISPLAY      console · desktop GUI · session log · (game engines next)
   ▲ bytes — framed, tagged, checksummed, append-only evolvable
ENGINE       HESF — protocol, orchestration, resolution, diagnostics
   ▲ bytes — versioned content files, CRC-verified
GAME         DIS — content, flow, meaning (authored via the editor, never hand-edited)
```

Three renderers currently consume the same byte stream simultaneously. None reference the simulation. None reference each other.

---

## The Primitive Surface (current, shipped)

**Wire / framing**
- `Packet` — self-describing frame: header-length, payload-length, path-tag, payload. Headers grow without breaking old readers.
- `StreamWalk` — blind multi-frame traversal; unknown tags are skipped, never guessed.
- `Tags` — append-only tag registry; exact-match dispatch only, tags never recycled.
- `TextChoiceNode` (+ serializer/reader) — the clean text-event shape: prose, question, choices. Carries no game mechanics by design.

**Control plane**
- `Hello` / `HelloAck` — version negotiation and capability declaration between simulation and any display.
- `SceneSet` — simulation names the scene; the display owns what that looks like.
- `EndOfUpdate` — frame-commit signal; every one is a valid save point.
- `Shutdown` — clean close, with reason.

**Input plane**
- `Selection` / `Advance` — the entire return channel: which node, which choice. A display returns an index; it never holds flow.
- `SimInputBoundary` — sim-side validation of untrusted display input: correlation and range, loud failure on both.
- `Handshake` — exact protocol version agreement.

**Bus**
- `Broadcaster` / `IConsumer` — shape-blind fan-out of opaque bytes. Adding a renderer is one registration call.

**Diagnostics**
- `BoundarySniffer` — every boundary crossing, both directions, logged to a binary session log.
- `BoundaryLogReader` — replays a logged session deterministically. The seed is in the log header; same inputs, same outcomes, every time.

**Event chain**
- `EventOrchestrator` / `EventBoundary` — load an event, present it, route the outcome by payload type.
- `TextEventHandler` — presents a text event across the serialized boundary and returns the selection.
- Payload/result pairs — state changes and handler calls cross as typed data, never as callbacks.

**Resolution**
- `Resolver` / `RollSpec` / `Op` — seeded, deterministic check resolution: difficulty, modifiers, dice, margin.
- `Iteration` — bounded retry with explicit termination reasons. Nothing loops forever.

**Identity**
- `Id<TTag>` — typed ids with sentinel-zero: zero is invalid everywhere, validity asserted at every boundary.

---

## Proven (stop gates, all green)

1. Serialization round-trips byte-stable — serialize, read, re-serialize, identical.
2. A full event presents, returns a selection, and resolves across the byte boundary end-to-end.
3. A selection correlated to the wrong event is rejected loudly, never applied.
4. An out-of-range selection is rejected loudly.
5. A logged session replays to an identical run.
6. Version handshake completes; mismatches fail loud, not silent.
7. Unknown frame tags are skipped; the stream walk survives.
8. Frames with future header fields are read correctly by today's readers.

Content files are versioned and checksum-verified on every read. Corrupt or hand-edited files fail loud before anything reaches the engine.

---

## Roadmap (criterion-based — steps complete when their gates pass, not when a calendar says so)

**Done**
- Engine primitive layer, wire protocol, validation, replay infrastructure
- Content editor write spine: authored content → validated, checksummed binary
- Three simultaneous renderers on one byte stream (console, session log, desktop GUI)

**Now**
- Character generation flow: the player answers three questions at a frontier outpost; the system quietly builds a character
- Editor user interface (cross-platform desktop)
- Display-layer placement: the clean game-to-display channel as its own brick

**Next (rough order, quarter-grade guesses, no promises)**
- First game-engine renderer (Unity) — same bytes, fourth surface (~Q3)
- Vertical slice: outpost, rover, surface exploration, one delve, one mystery (~Q3–Q4)
- Public playable demo on itch.io (~Q4)
- The dwarves remain serious throughout

**Later**
- Combat as its own cross-scene subsystem
- Multi-planet content; the engine's second game wakes up
- Unreal renderer — the same fourth job, a fifth time

---

*Everything above is shipped or gated. The commit history is the proof. The dates are guesses; the gates are not.*
