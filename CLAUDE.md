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
- `PLAYED` — hardcoded group match results already played; add results here as games finish (`"G:pi": [scoreA, scoreB]`)
- `KO_PLAYED` — hardcoded knockout results, keyed by official match number (73–104). The score is the **after-extra-time** result, excluding penalties: `"73": [a, b]`, or `"74": [a, b, "A"|"B"]` when level after ET (3rd value = who advanced on penalties), or `"74": [a, b, "A", penA, penB]` to also record the shootout (shown in parentheses).
- **Knockout bracket — mirrors the official FIFA 2026 bracket directly** (decoupled from the app's group model). Matches 73–104, verified against Wikipedia + aggregators:
  - `KO_R32` — 16 Round-of-32 matchups (73–88); each slot is a **fixed real team**: `{code:"RSA"}`. (Not derived from group standings, and no best-thirds table.)
  - `KO_R16`/`KO_QF`/`KO_SF` — later rounds; each slot is `{src:n}` = winner of match `n`. `KO_THIRD` (103) uses `{srcLoser:n}`; `KO_FINAL` (104).
  - `KO_ALL` / `KO_BY_ID` — all knockout defs flattened and indexed by id. `KO_ROUNDS` — render-order columns + labels (R16 / R8 / QF / SF / Final; third place "3.º puesto").
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
- `resolveSlot(slot)` — resolves an R32 slot (`winner`/`runner`/`best3rd`) to a team row, or `null` if not yet determined
- `isSlotConfirmed(slot)` — true when the slot's group(s) are complete; drives `r32-ghost` opacity on unconfirmed slots
- `tally()` / `ledgerHTML()` — scan group **and** knockout matches with actual results and compute hit rates (yours vs. model), exact scores, and Brier score, shown in the "Mis pronósticos" tab (`view-dash` / `renderDash()`). Knockout uses the after-ET score (penalties shown as `(p-p)` in the ledger). Total denominator is 104 (72 groups + 32 knockout).

#### Knockout advancement (Dieciseisavos → Final)
**Advancement is driven only by the real result, never by the user's prediction.** Each knockout match is stored at `STATE.matches["KO:<n>"]`; the user keeps predicting (`predA/predB`, the after-ET score) but who advances comes from `actA/actB` (+ `advance` for ties decided on penalties), fed by `KO_PLAYED` and the live fetch. A later-round team only appears once its feeder match has a real result.
- `koRec(id)` — `STATE.matches["KO:"+id]`
- `resolveKOSlot(slot)` — `{code,name}` for fixed R32 teams; `resolveKOWinner`/`resolveKOLoser` for `{src}`/`{srcLoser}`
- `resolveKOTeams(def)` — `{teamA, teamB}` (either may be `null` until decided)
- `koAdvancer(id)` — `'A'|'B'|null`; from the after-ET `actA/actB`, falling back to `advance` when level (penalties)
- `resolveKOWinner(id)` / `resolveKOLoser(id)` — recursive; the advancing / eliminated team row, or `null`
- `isKOConfirmed(id)` / `isKOSlotConfirmed(slot)` — decided? (R32 fixed teams are always confirmed); drives downstream `r32-ghost`
- `matchupLabel(n)` / `koSideLabel` — placeholder for an undecided slot, **referencing the feeder match's teams** ("Ganador de Alemania–Paraguay"; deeper rounds collapse to codes "GER/PAR – FRA/SWE"), never "Ganador #n"

### Persistence (`Store`)
Async wrapper with two-tier fallback: `localStorage` first, then `window.storage` (Claude preview environment). Writes are debounced 300 ms via `saveTimer`. Storage key is `"wc26"`. Export/import is available in the header via `exportJSON()` / `importJSON()` for cross-device portability.

### Boot sequence
`boot()` is the async entry point: `load()` → `seedState()` → `buildTabs()` → `renderGroups()` → wire header buttons. `seedState()` uses `??=` throughout, so saved `STATE` values always win over `PLAYED` entries and initial defaults — localStorage is the source of truth once the app has ever run.

### Render strategy
- `renderGroups()` — full re-render of all group blocks (standings + match cards); called on rating/adj changes
- `refreshCard(id)` — surgical DOM patch of a single match card (probabilities, verdict, team λ labels) without touching score inputs, preserving keyboard focus during live score entry
- `refreshStandings(g)` — misleadingly named; delegates to the full `renderGroups()`, not a partial update
- `renderBracket()` — full re-render of the Llaves tab as a left-to-right knockout tree (columns R16 → R8 → QF → SF → Final, plus a 3.º puesto block); uses `koMatchHTML(def)` per match and CSS elbow connectors (`.ko-pair::before/::after`). Calls `bindBracketInputs()`. Called on tab switch and by `refreshLlavesIfOpen()`. A finished match (`hasScore`) renders **only the two team modules** (flag, code, name, goals + penalties `(p)`), highlighting the winner — no central score block. The predictable state always shows a `Guardar pronóstico` button, `disabled` until both scores are entered.
- `renderDash()` / `renderModel()` — rendered on tab switch
- `refreshDashIfOpen()` / `refreshLlavesIfOpen()` — keep Dashboard and Llaves live while entering scores in Groups

### Event binding
All interactive elements use `data-*` attributes (`data-pred`, `data-act`, `data-venue`, `data-alt`, `data-rating`, `data-adj`) as hooks. `bindGroupInputs()` wires these after every `renderGroups()` call. Score inputs are sanitised by `clean()`: strips non-digits, caps at 20. Group headers (`data-grp`) toggle collapse state and persist it to `STATE.collapsed`.

Knockout cards reuse `data-pred="KO:<n>:A|B"` for predictions and `data-ko-confirm` / `data-ko-edit` for save/edit; `bindBracketInputs()` wires these after each `renderBracket()`. To preserve input focus, prediction edits do not re-render until saved.

### Disagree highlight
A match card gets the `.disagree` CSS class (amber border) when the user has entered a prediction whose outcome (H/D/A) differs from the model's top pick. Knockout cards apply the same highlight once a prediction is confirmed.

### Tabs
Four views: `view-groups`, `view-llaves`, `view-dash`, `view-model`. Active tab toggled via `.on` class on both `<button>` and `<section>`. Tab IDs are `groups`, `llaves`, `dash`, `model`; labels are `Grupos`, `Llaves`, `Precisión`, `Modelo`.

### Language
The entire UI is in Spanish. All labels, messages, and footer copy are Spanish. Keep any new UI text in Spanish.

## Key IDs and Data Keys

- Group match ID format: `"G:pi"` — group letter + colon + pairing index (0–5), e.g. `"A:0"`
- Knockout match ID format: `"KO:<n>"` — official match number 73–104, e.g. `"KO:73"`
- `STATE.matches[id]` (group): `{ venue: "N"|"A"|"B", alt: bool, predA, predB, actA, actB }`
- `STATE.matches["KO:<n>"]` (knockout): `{ predA, predB, predConfirmed, actA, actB, advance: "A"|"B"|null, penA, penB, status, minute, kickoff }` — `actA/actB` = after-ET score; `penA/penB` = shootout (or `null`)
- `STATE.rating[code]` — editable FIFA rating; `STATE.adj[code]` — per-team adjustment (form, injuries)
- `STATE.collapsed[g]` — boolean per group letter; persisted so collapsed state survives reload

## What to touch when updating results

**Group matches:**
1. Add to `PLAYED` for the hardcoded baseline (`"G:pi": [scoreA, scoreB]`), or
2. Let the live fetch fill `STATE.matches[id].actA/actB` automatically.

**Knockout matches** (drive the bracket's advancement). The score is the **after-extra-time** result, excluding penalties:
1. Add to `KO_PLAYED` keyed by official match number: `"73": [a, b]`; `"74": [a, b, "A"|"B"]` when level after ET (who advanced on penalties); `"74": [a, b, "A", penA, penB]` to also store the shootout (rendered in parentheses, e.g. `1–1 (4–2)`). The live fetch (`fetchLiveResults`) fills these automatically — `actA/actB` from `score.fullTime`, `advance` from `score.winner`, `penA/penB` from `score.penalties`. The user's prediction never affects who advances.
