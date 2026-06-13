# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Type

**static** — single self-contained HTML file. No build step, no package manager, no backend, no secrets, no Doppler. All persistence is via `localStorage` (key: `wc26`).

The entire application lives in `index.html`.

## How to Run

Open `index.html` directly in any browser. No server required.

## Architecture

### Data layer (top of `<script>`)
- `TEAMS` — 12 groups (A–L), 4 teams each: `[name, code, fifaPts]`
- `PAIRINGS` — the 6 round-robin pairs per group by team index (`[[0,1],[2,3],[0,2],[3,1],[3,0],[1,2]]`)
- `GROUP_LETTERS` — `Object.keys(TEAMS)`, used for iteration throughout
- `VENUE_PRESETS` — pre-set home advantage for Mexico, Canada, USA (host nations)
- `PLAYED` — hardcoded match results already played as of the current date; add results here as games finish
- `R32_BRACKET` — 16 Round-of-32 matchups, each slot typed as `winner`, `runner`, or `best3rd`
- `BRACKET_SECTIONS` — groups R32 matches into 4 display quadrants (Cuadrante A–D)
- `DEFAULTS` — baseline model parameters (`base`, `sens`, `hfa`, `alt`)
- `STATE` — the mutable runtime object: `{ rating, adj, matches, settings, collapsed }`, persisted to `localStorage`

### Prediction model (`model()`)
Poisson with independent goal lambdas:
- `effRating(code) = STATE.rating[code] + STATE.adj[code]`
- `Δ = (effRatingA + homeBonus) − (effRatingB + homeBonus)`
- `λA = base × e^(sens × Δ)`, `λB = base × e^(−sens × Δ)`
- Probabilities computed over a 9×9 score matrix (0–8 goals each side), then normalised
- `FACT` is a precomputed factorial array (indices 0–9) used by `pois(k, l)`

### Standings and bracket logic
- `computeStandings(g)` — returns sorted array of team rows for a group, accumulating actual match results from `STATE.matches`
- `isGroupComplete(g)` — returns true only when all 6 group matches have actual scores entered
- `getBest8Thirds()` — collects the 3rd-place team from every group, sorts by Pts/GD/GF, returns top 8
- `resolveSlot(slot)` — resolves a bracket slot (`winner`/`runner`/`best3rd`) to a team row, or `null` if not yet determined
- `isSlotConfirmed(slot)` — true when the slot's group(s) are complete; drives `r32-ghost` opacity on unconfirmed slots
- `tally()` — scans all matches with actual results and computes hit rates (yours vs. model), exact scores, and Brier score

### Persistence (`Store`)
Async wrapper with two-tier fallback: `localStorage` first, then `window.storage` (Claude preview environment). Writes are debounced 300 ms via `saveTimer`. Storage key is `"wc26"`. Export/import is available in the header via `exportJSON()` / `importJSON()` for cross-device portability.

### Boot sequence
`boot()` is the async entry point: `load()` → `seedState()` → `buildTabs()` → `renderGroups()` → wire header buttons. `seedState()` uses `??=` throughout, so saved `STATE` values always win over `PLAYED` entries and initial defaults — localStorage is the source of truth once the app has ever run.

### Render strategy
- `renderGroups()` — full re-render of all group blocks (standings + match cards); called on rating/adj changes
- `refreshCard(id)` — surgical DOM patch of a single match card (probabilities, verdict, team λ labels) without touching score inputs, preserving keyboard focus during live score entry
- `refreshStandings(g)` — misleadingly named; delegates to the full `renderGroups()`, not a partial update
- `renderBracket()` — full re-render of the Llaves tab; called on tab switch and by `refreshLlavesIfOpen()`
- `renderDash()` / `renderModel()` — rendered on tab switch
- `refreshDashIfOpen()` / `refreshLlavesIfOpen()` — keep Dashboard and Llaves live while entering scores in Groups

### Event binding
All interactive elements use `data-*` attributes (`data-pred`, `data-act`, `data-venue`, `data-alt`, `data-rating`, `data-adj`) as hooks. `bindGroupInputs()` wires these after every `renderGroups()` call. Score inputs are sanitised by `clean()`: strips non-digits, caps at 20. Group headers (`data-grp`) toggle collapse state and persist it to `STATE.collapsed`.

### Disagree highlight
A match card gets the `.disagree` CSS class (amber border) when the user has entered a prediction whose outcome (H/D/A) differs from the model's top pick.

### Tabs
Four views: `view-groups`, `view-llaves`, `view-dash`, `view-model`. Active tab toggled via `.on` class on both `<button>` and `<section>`. Tab IDs are `groups`, `llaves`, `dash`, `model`; labels are `Grupos`, `Llaves`, `Precisión`, `Modelo`.

### Language
The entire UI is in Spanish. All labels, messages, and footer copy are Spanish. Keep any new UI text in Spanish.

## Key IDs and Data Keys

- Match ID format: `"G:pi"` — group letter + colon + pairing index (0–5), e.g. `"A:0"`
- `STATE.matches[id]`: `{ venue: "N"|"A"|"B", alt: bool, predA, predB, actA, actB }`
- `STATE.rating[code]` — editable FIFA rating; `STATE.adj[code]` — per-team adjustment (form, injuries)
- `STATE.collapsed[g]` — boolean per group letter; persisted so collapsed state survives reload

## What to touch when updating results

1. Add to `PLAYED` if the result should be the hardcoded baseline (permanent fixtures)
2. Or enter results via the UI (stored in `STATE.matches[id].actA/actB` → persisted to localStorage)

When adding new played matches, use the existing `PLAYED` format: `"G:pairingIndex": [scoreA, scoreB]`.
