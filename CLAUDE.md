# Proactive Supply Chain Group — Org Restructuring Tool

## Purpose

An interactive, drag-and-drop tool for modelling proposed org restructures at
Proactive Supply Chain Group. Leadership drags employee cards between teams to
visualize a new structure, then exports/prints the result to share. It is meant
to look presentation-ready, not like an internal prototype.

## File structure

```
proactive-restructure/
├── index.html          # The ENTIRE application: HTML + embedded <style> + <script>.
├── CLAUDE.md           # This file.
└── .github/
    └── workflows/
        └── claude.yml  # GitHub Action for @claude mentions (unrelated to the app).
```

Everything the app needs lives in `index.html`. There is no separate CSS or JS file.

## Conventions

- **Single self-contained file.** All HTML, CSS, and JavaScript live in `index.html`.
- **No dependencies, no build step, no npm.** No frameworks and no external/CDN
  links — the file must open by double-clicking and work fully offline.
- **Vanilla JS only**, using native HTML5 drag-and-drop. No jQuery, React, etc.
- **No innerHTML with user data.** Employee names/roles are rendered with
  `textContent` to avoid injection.
- Keep the code readable and the styling driven by the CSS custom properties in
  `:root` (the Proactive brand palette). Use the brand red sparingly, for accents
  and hover/focus states.

## Data model

State is a single flat list of employees; each employee records which drop zone
it currently sits in.

```jsonc
// employee
{ "id": "emp-1", "name": "Jane Doe", "role": "VP Operations", "section": "c-suite" }
```

Valid `section` (drop-zone) ids:

| Area        | Zone ids                                                                       |
|-------------|--------------------------------------------------------------------------------|
| C-Suite     | `c-suite`                                                                      |
| Warehousing | `warehousing-toronto`, `warehousing-montreal`, `warehousing-la`, `warehousing-houston` |
| Freight     | `freight-canada`, `freight-us`, `freight-mexico`                               |
| Terminated  | `terminated`                                                                   |

- **Persistence:** the full state is saved to `localStorage` under the key
  `proactive-restructure-state` on every change, so a refresh keeps your work.
- **Export JSON** downloads `{ app, version, exportedAt, employees }` as
  `proactive-org-YYYY-MM-DD.json`.
- **Import JSON** validates that an `employees` array is present, then coerces each
  record into the shape above (unknown sections fall back to `terminated`).
- **Reset to Original** restores the hardcoded `ORIGINAL_STATE` seed (confirmed
  first via a dialog).
- **Terminate (×)** moves an employee to the `terminated` zone — it is recoverable
  by dragging the card back out.

## How to test

There is no test suite and no server. To test:

1. Open `index.html` in a browser (double-click it, or `open index.html` on macOS).
2. Verify:
   - All four sections render with the seeded placeholder employees.
   - Dragging a card between zones works and the counts update.
   - **Add Employee** opens the modal and creates a card in the chosen section.
   - The **edit (✎)** icon updates a card's name/role.
   - The **× / Terminate** button moves a card to Terminated.
   - Refreshing the page preserves the current layout (localStorage).
   - **Export JSON** downloads a file; **Import JSON** loads it back.
   - **Reset to Original** restores the seed data after confirmation.
   - **Print / PDF** shows a clean layout (toolbar and Terminated section hidden).

Quick JS syntax check (does not run the app, just parses it):

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);new (require('vm').Script)(m[1]);console.log('JS syntax OK');"
```
