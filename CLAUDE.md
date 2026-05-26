# Proactive Supply Chain Group — Org Chart Tool

## Purpose

An interactive org chart for modelling proposed restructures at Proactive Supply
Chain Group. The full ~220-person hierarchy renders as a true org chart with
reporting lines; leadership drags nodes to change who reports to whom, terminates
or adds people, color-codes and filters by division, and exports/prints the
result. Presentation-ready.

## Architecture

Org chart built on **dabeng/OrgChart** (MIT licensed jQuery plugin) loaded from
CDN. This replaced an earlier BalkanGraph attempt, whose free Community edition
gated drag-to-reparent and search behind the paid tier.

Dependencies (all CDN, pinned; the app needs internet on first load, then caches
data in localStorage and works offline):

```
jQuery        3.7.1   https://cdn.jsdelivr.net/npm/jquery@3.7.1/dist/jquery.min.js
OrgChart CSS  4.0.1   https://cdn.jsdelivr.net/npm/orgchart@4.0.1/dist/css/jquery.orgchart.min.css
OrgChart JS   4.0.1   https://cdn.jsdelivr.net/npm/orgchart@4.0.1/dist/js/jquery.orgchart.min.js
html2canvas   1.4.1   (used by OrgChart export)
jsPDF         2.5.1   (used by OrgChart export)
```

**Do not bump these to npm "latest".** orgchart 5.x dropped jQuery (no
`$().orgchart()`), jQuery 4.x is a breaking major, and jsPDF 3.x/4.x changed the
global/API that OrgChart 4.x's export expects. 4.0.1 is the last jQuery-plugin
release. (Note: the originally requested `orgchart@4.6.1` does not exist on npm;
4.0.1 is the correct latest 4.x.)

Data is kept **separate from code**: `index.html` `fetch()`es `employees.json` on
boot — the roster is never inlined.

## File structure

```
proactive-restructure/
├── index.html          # App: HTML + CSS + JS. Loads dabeng/OrgChart from CDN, fetches data.
├── employees.json      # Roster wrapper: { app, version, schema, exportedAt, source, ceo_id, employees[] }.
├── CLAUDE.md
└── .github/workflows/claude.yml
```

## Data model

Each employee record:

```jsonc
{ "id": "000293", "pid": "", "name": "Salvatore Mancuso",
  "title": "Chief Executive Officer", "dept": "OWNERS", "location": "Ontario",
  "status": "Active", "terminated": false, "division": "C-Suite" }
```

- Flat `id`/`pid` is the source of truth. At render time it's converted to the
  **nested tree** (`{ id, name, children[] }`) that dabeng requires; on changes
  the flat array is updated and the tree rebuilt.
- `status`: `"Active"` | `"Leave"` (Leave shows an "On Leave" badge).
- `terminated: true` hides the node and lists it in the Terminated sidebar.
- `division` (see below) drives the colored accent and the filter.

### Important: source ids are NOT unique

The ADP dump reuses `id` across different people (220 records, 174 distinct ids,
46 collisions). dabeng needs unique node ids, so on load ids are **uniquified**:
first occurrence keeps its id, later collisions get an `-N` suffix (`000052-2`).
The DOM node also carries `data-emp-id`, which is what the click/drag/search code
reads — so it never depends on dabeng's own id handling or on CSS id selectors
(many ids start with a digit). Treat `id` as a chart key, not an ADP number.

### Divisions

`division` is derived on load from the dept code + title (existing `division` in
imported data is kept; otherwise derived), persisted on each record, included in
export, and editable in the modal. Derivation order (first match wins):

| Division     | Rule                                                      | Accent  | Hex      |
|--------------|-----------------------------------------------------------|---------|----------|
| C-Suite      | dept `OWNERS`, or title has Chief/CEO/COO/CFO/CCO/President| red     | #D32F2F  |
| Specialized  | dept starts with `SPE`                                    | amber   | #D97706  |
| Warehousing  | dept contains `WHS`/`WAR`/`WPL`, or `NLFPRO`              | charcoal| #1F2937  |
| Freight      | dept contains `FRT`/`TRA`/`HOU`, or `ATLOPS`              | slate   | #6B7280  |
| Support      | everything else (`ACC`, `ITD`, `HUMRES`, `BOOKKP`, `OFF`) | teal    | #0F766E  |

(Specialized is checked before Warehousing/Freight so `SPEOFF` → Specialized, not
Support; Warehousing before Freight so `TRAWAR` → Warehousing.)

## Persistence

- **State key:** `proactive-restructure-state-v2` — full wrapper saved on every change.
- **Original snapshot:** `proactive-restructure-original-v2` (written on first load
  so Reset works offline).
- **Filter:** `proactive-restructure-filter-v2` (active division filter persists).
- **Boot order:** localStorage → else `fetch(employees.json)` → else in-app banner.

## Features

- Org chart of all non-terminated employees; node card = name (bold) / title /
  dept · location / "On Leave" badge, with a division-colored left border.
- **Drag a node onto another** to reparent (`draggable: true`); the `nodedrop`
  handler updates `pid` and auto-saves. **pan**/**zoom** enabled.
- **Click a node** (delegated `$('#chart-container').on('click','.node')`) → modal
  to edit name/title/dept/location/division, Terminate, or Delete. Terminating or
  deleting a manager re-homes their reports to the manager above.
- **Division filter bar** ([All] + 5 divisions): dims non-matching nodes to ~25%
  opacity (structure stays visible); persisted in localStorage.
- **Search** (name + title + dept, fuzzy/subsequence): highlights matches with a
  yellow outline and scrolls to the first match; clear button resets.
- **Terminated sidebar** (toggle): click an entry to reinstate (under original
  manager, or CEO if gone).
- **Export JSON** (same wrapper schema, round-trippable) / **Import JSON**.
- **Print / PDF** via OrgChart's `export()` (html2canvas + jsPDF); falls back to
  the browser print dialog if unavailable.

## Conventions

- Single `index.html` for app code; data in `employees.json`.
- CDN deps only; no npm/build step. jQuery is available; use it for chart/node DOM,
  vanilla JS elsewhere. Never bind chart node handlers directly — use delegation on
  `#chart-container` so they survive re-renders.
- No `innerHTML` with user data (names/titles set via jQuery `.text()` / `textContent`).
- Brand palette in `:root`. dabeng node colors come from these via CSS classes
  (`.division-*`, `.node`, `.node-sub`, `.node-badge`).

## How to run / test

The app `fetch()`es a local file, so opening `index.html` directly (`file://`) is
blocked. Serve the folder:

```bash
cd proactive-restructure
python3 -m http.server 8000   # then open http://localhost:8000
```

Manual checks: all 220 nodes render under the CEO; drag a node onto another and
reload (reparent persists); click opens the edit modal; search highlights + scrolls;
division filter dims correctly and persists across reload; colored left borders match
divisions; Print/PDF produces a file.

Headless checks (no browser; can't verify rendering/drag/PDF):

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>\s*\"use strict\"[\s\S]*?<\/script>/);new (require('vm').Script)(m[0].replace(/^<script>/,'').replace(/<\/script>$/,''));console.log('JS OK');"
node -e "console.log('employees:', require('./employees.json').employees.length);"
```
