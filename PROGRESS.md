# Progress Ledger

Sanitized brick ledger: one line per landed milestone; the private repo is the source of truth.

2026-07-13 — Internal bus re-point — dual-spine internal message bus (direct/journaled, runtime-selectable); legacy broadcast path retired; all gates green
2026-07-14 — Display data tier — layouts, themes, and pane compositions now authored data in the display registry; round-trip gated
2026-07-14 — Pane framework — data-composed display surfaces (viewport/output/response), semantic-action input funnel, character generation recomposed onto it; behavioral-equivalence gated
2026-07-14 — Display registry rider — piece parameters and font-size keys added to the theme vocabulary
2026-07-14 — Event registry milestone — character generation event set serialized to and loaded from the engine's registry format by a production loader; the game now runs from data files end to end
