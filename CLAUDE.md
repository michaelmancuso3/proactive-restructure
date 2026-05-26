# Proactive Supply Chain Group — Org Chart Tool

## Purpose

An interactive org chart for modelling proposed restructures at Proactive Supply
Chain Group. The full ~220-person hierarchy renders as a true tree with
Visio-style orthogonal connectors; leadership drags nodes to change reporting
lines, color-codes/filters by division, terminates or adds people, and
exports/prints the result. Presentation-ready.

## Architecture

Built on **d3-org-chart** (MIT, by bumbeishvili — the d3-based library used by
Coca-Cola, Microsoft, etc.). Replaced earlier BalkanGraph and dabeng attempts.

Dependencies (all CDN; the app needs internet on first load, then caches data in
localStorage and works offline):

```
d3            7      https://cdn.jsdelivr.net/npm/d3@7
d3-flextree   2.1.2  https://cdn.jsdelivr.net/npm/d3-flextree@2.1.2/build/d3-flextree.js
d3-org-chart  3.1.1  https://cdn.jsdelivr.net/npm/d3-org-chart@3.1.1
jsPDF         2.5.1  https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js
```

**d3-flextree is required** — d3-org-chart calls `d3.flextree(...)` for layout
and throws `d3.flextree is not a function` without it. Load it after d3 and
before d3-org-chart. Keep jsPDF at 2.x (3.x/4.x changed the global/API).
d3-org-chart 3.1.1 is the current latest.

Data is kept separate from code: `index.html` `fetch()`es `employees.json` on
boot — the roster is never inlined.

## File structure

```
proactive-restructure/
├── index.html          # App: HTML + CSS + JS. Loads d3-org-chart from CDN, fetches data.
├── employees.json      # Roster wrapper: { app, version, schema, exportedAt, source, ceo_id, employees[] }.
├── CLAUDE.md
└── .github/workflows/claude.yml
```

## Data model

```jsonc
{ "id": "000293", "pid": "", "name": "Salvatore Mancuso",
  "title": "Chief Executive Officer", "dept": "OWNERS", "location": "Ontario",
  "status": "Active", "terminated": false, "division": "C-Suite" }
```

- Flat `id`/`pid` is the source of truth. At render time it's transformed into the
  flat `{id, _parent}` shape d3-org-chart consumes (it stratifies internally).
  Exactly one root (CEO, `_parent: null`); any node whose manager is
  terminated/missing is re-homed to the root so stratify never sees a dangling
  parent or multiple roots.
- `status`: `"Active"` | `"Leave"` (Leave shows an "On Leave" badge).
- `terminated: true` hides the node and lists it in the Terminated sidebar.
- `division` drives the colored left border and the filter.

### Important: source ids are NOT unique

The ADP dump reuses `id` across different people (220 records, 174 distinct ids,
46 collisions). Ids are uniquified on load (first occurrence kept, collisions get
an `-N` suffix). Node cards carry `data-emp-id`, which the drag/click/search code
reads. Treat `id` as a chart key, not a canonical ADP number.

### Divisions (4 — no "Specialized")

Derived on load from dept code + title (existing valid `division` is kept;
legacy "Specialized" falls through to re-derivation → Freight). Order, first
match wins:

| Division     | Rule                                                          | Border  | Hex     |
|--------------|---------------------------------------------------------------|---------|---------|
| C-Suite      | dept `OWNERS`, or title has Chief/CEO/COO/CFO/CCO/President/VP | red     | #D32F2F |
| Warehousing  | dept contains `WHS`/`WAR`/`WPL`, or `NLFPRO`                  | charcoal| #1F2937 |
| Freight      | dept contains `FRT`/`TRA`/`HOU`, `ATLOPS`, or starts with `SPE`| slate   | #6B7280 |
| Support      | everything else (`ACC`, `ITD`, `HUMRES`, `BOOKKP`, `OFF`, …)  | teal    | #0F766E |

C-Suite is checked first, so an SPE employee with a "VP" title (e.g. "VP, PSL CA
& US") classifies as C-Suite, not Freight. Seed distribution: Warehousing 122,
Support 54, Freight 34, C-Suite 10. Editable per-employee in the modal.

## Persistence

- **State:** `proactive-restructure-state-v3` (new key so old broken state never
  loads). Full wrapper saved on every change.
- **Original snapshot:** `proactive-restructure-original-v3` (for offline Reset).
- **Filter:** `proactive-restructure-filter-v3`.
- **Boot:** localStorage → else `fetch(employees.json)` → else in-app banner.

## Features

- Tree org chart, orthogonal connectors. Spacing: `nodeWidth 220`, `nodeHeight
  110`, `childrenMargin 40`, `siblingsMargin 20`, `compactMarginBetween 12` (note
  d3-org-chart wants these as **functions**, e.g. `.nodeWidth(()=>220)`).
- **Reparenting is click-based** (drag was removed). Drag-to-reparent was tried on
  3 libraries and is unreliable here: d3-org-chart puts `d3.zoom` on the same SVG
  as the cards, so node-drag fights chart-pan, and at fit zoom each card is only
  ~12×6 px. Two reliable paths instead:
  - **Click a card (or its ⇄ icon) → edit modal → "Reports to"** searchable
    autocomplete. It lists all employees except the person, their descendants, and
    terminated staff; pick one (chip + × to clear); Save sets `pid` and redraws.
    Same-person / descendant selections are blocked with an inline error.
  - **Move Mode** toolbar button: click it, click the person to move, then click
    their new manager. CEO can't be a source; descendant targets are rejected.
    Esc or a background click cancels.
  `reparentNode()` enforces the CEO/cycle rules for both paths.
- **Zoom:** `scaleExtent([0.1, 2.5])`, `duration(400)`; wheel sensitivity tamed
  ~5× via `chart.zoomBehavior().wheelDelta(...)`. Floating controls (bottom-left):
  Zoom In/Out → `zoomIn()`/`zoomOut()`, Fit → `fit()`, Reset → `expandAll()+fit()`
  (the library has no `reset()`).
- **Click a node** → edit modal (name/title/dept/location/division), Terminate,
  Delete. Terminating/deleting a manager re-homes their reports upward.
- **Division filter bar** ([All] + 4 divisions): dims non-matching cards to ~22%;
  persisted.
- **Search** (name+title+dept, fuzzy): instant yellow-highlight + count on every
  keystroke; centers on the first match (`setCentered`+`setHighlighted`) after a
  short pause; clear button.
- **Terminated sidebar** (toggle): click to reinstate.
- **Export JSON** (same wrapper schema, round-trips) / **Import JSON**.
- **Print/PDF:** `chart.exportImg({full:true,...})` → the PNG is wrapped in a PDF
  via jsPDF (fits A4, orientation by aspect); falls back to `window.print()`.

## Conventions

- Single `index.html` for app code; data in `employees.json`. CDN deps only, no build.
- d3-org-chart re-renders rebuild all node DOM, so re-attach card click handlers
  (`attachCardHandlers`, idempotent via `onclick=`) + re-apply filter/search/move
  classes after every render (see `postRender()`).
- No `innerHTML` with unescaped user data — `nodeContent` HTML strings run through
  `escapeHtml()`.
- Brand palette in `:root` (no amber after Specialized was dropped). Division
  border colors are CSS classes `.oc-card.division-*`.

## How to run / test

The app `fetch()`es a local file, so `file://` is blocked. Serve the folder:

```bash
cd proactive-restructure
python3 -m http.server 8000   # then open http://localhost:8000
```

Manual checks: 220 nodes render under the CEO with orthogonal connectors; click a
card → modal → "Reports to" autocomplete reparents and persists on reload; Move
Mode reparents in two clicks; CEO/descendant moves are blocked; search highlights
+ centers; division filters dim; zoom feels smooth via wheel + buttons; Print/PDF
produces a usable PDF. The click-reparent flow + Move Mode are verified end-to-end
in headless Chromium (puppeteer); drag/zoom/PDF visuals still need a real browser.

Headless checks (no browser):

```bash
# JS parses
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>\s*\"use strict\"[\s\S]*?<\/script>/);new (require('vm').Script)(m[0].replace(/^<script>/,'').replace(/<\/script>$/,''));console.log('JS OK');"
# data is 220 rows
node -e "console.log('employees:', require('./employees.json').employees.length);"
```

A deeper headless render check (jsdom + d3 + d3-flextree + d3-org-chart, with a
stubbed canvas) confirms all 220 cards render and division classes apply; it
cannot exercise drag/zoom/PDF/visual layout — those need a real browser.
