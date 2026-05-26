# Proactive Supply Chain Group — Org Chart Tool

## Purpose

An interactive org chart for modelling proposed restructures at Proactive Supply
Chain Group. The full ~220-person hierarchy renders as a true org chart with
reporting lines; leadership drags nodes to change who reports to whom, terminates
or adds people, and exports/prints the result to share. Presentation-ready.

## Architecture (v2)

This replaced the original v1 column/drag-board layout. It is now an org chart
built on **BalkanGraph OrgChart JS**, loaded from CDN.

- **Library:** `https://cdn.balkan.app/orgchart-community.js` (free Community
  edition — chosen to avoid a trial watermark). Because it loads from a CDN, the
  app needs internet **the first time**. After a successful load the data is
  cached in `localStorage`, so later opens work offline.
- **Data is kept separate from code.** `index.html` does NOT inline the roster;
  it `fetch()`es `employees.json` on boot so the file can be re-exported/replaced.

## File structure

```
proactive-restructure/
├── index.html          # App: HTML + embedded CSS + JS. Loads OrgChart from CDN, fetches data.
├── employees.json      # Roster (wrapper object: { app, version, schema, exportedAt, source, ceo_id, employees[] }).
├── CLAUDE.md           # This file.
└── .github/workflows/claude.yml
```

## Data model

`employees.json` is a wrapper object whose `employees` array holds the records:

```jsonc
{ "id": "000293", "pid": "", "name": "Salvatore Mancuso",
  "title": "Chief Executive Officer", "dept": "OWNERS",
  "location": "Ontario", "status": "Active", "terminated": false }
```

- `id` / `pid` drive the chart (OrgChart uses them natively). Root = CEO, `pid: ""`.
- `status` is `"Active"` or `"Leave"` (Leave shows an "On Leave" badge on the node).
- `terminated: true` hides the node from the chart and lists it in the Terminated sidebar.

### Important: source ids are NOT unique

The ADP dump reuses `id` values across different people — 220 records, only 174
distinct ids (46 collisions). OrgChart requires unique ids, so on load the app
**uniquifies** them: the first occurrence of an id keeps it; later collisions get
an `-N` suffix (e.g. `000052-2`). The first occurrence is always preserved, so
existing `pid` references still resolve. This re-keying is applied to exports too,
so a round-trip stays internally consistent. Treat `id` as a chart key, not a
canonical ADP employee number.

## Persistence

- **Storage key:** `proactive-restructure-state-v2` (separate from the retired v1
  key so the old column data never clashes). The whole wrapper object is saved on
  every change.
- An original snapshot is also stored under `proactive-restructure-original-v2`
  on first load so **Reset to Original** works offline.
- **Boot order:** localStorage (if present) → else `fetch(employees.json)` → else
  show an in-app banner explaining how to load the data.

## Features

- Org chart of all non-terminated employees with reporting lines.
- **Drag a node onto another** to change its manager (`enableDragDrop`); auto-saved.
- **Search box** centers the chart on the first name match.
- **Add Employee** modal (name/title/dept/location + "Reports to" manager picker).
- **Click a node** → modal to edit name/title/dept/location, Terminate, or Delete.
  Terminating/deleting a manager re-homes their direct reports to the manager above.
- **Terminated sidebar** (toggle button) lists terminated people; click to reinstate
  (returns under their original manager, or the CEO if that manager is gone).
- **Export JSON** downloads the current state in the same wrapper schema as
  `employees.json` (round-trippable).
- **Import JSON** loads a previously exported file (wrapper object or bare array).
- **Print / PDF** uses OrgChart's `exportToPDF()` when available, else falls back to
  the browser print dialog (which can "Save as PDF").

## Conventions

- Single `index.html` for all app code; data lives in `employees.json`.
- One external dependency: the BalkanGraph CDN script. No npm/build step.
- Vanilla JS. Employee names/titles render via OrgChart bindings / `textContent`
  (no `innerHTML` with user data).
- Brand palette lives in `:root` CSS custom properties. OrgChart SVG nodes can't
  use `var()` in attributes, so node colors are the same hex values inlined in the
  templates / `.oc-*` classes — keep them in sync with `:root` if the palette changes.

## How to run / test

Because the app `fetch()`es a local file, opening `index.html` directly
(`file://`) is blocked by browsers. Serve the folder:

```bash
cd proactive-restructure
python3 -m http.server 8000
# then open http://localhost:8000
```

Manual checks: chart renders all 220 nodes under the CEO (none are terminated in
the seed; 9 are flagged "On Leave"); drag a node to a new
manager and reload (change persists); search jumps to a person; Add/Edit/Terminate/
Delete work; terminated people appear in the sidebar and reinstate correctly;
Export then Import round-trips; Print/PDF produces output.

Quick headless checks (no browser; can't verify rendering):

```bash
# JS parses
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);new (require('vm').Script)(m[1]);console.log('JS OK');"
# data is 220 rows
node -e "const d=require('./employees.json');console.log('employees:',d.employees.length);"
```
