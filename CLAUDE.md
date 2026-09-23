# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Kanban board (`index.html`) for an internal IT PMO demo/training tool. All markup, CSS (`<style>`) and JS (`<script>`) live in that one file. There is no build step, no package manager, and no tests.

## Hard constraints (from the project brief — do not break these)

- Vanilla HTML/CSS/JS only: no frameworks, libraries, bundlers, npm, CDN scripts, web fonts or image files. Icons are Unicode glyphs or inline SVG; fonts are a system stack.
- Must work when opened directly from disk (`file://`), with no server.
- **No persistence.** Do not use localStorage, sessionStorage, IndexedDB or cookies. A refresh resets the board to the seed data by design, and the header note tells users so.
- The only network call is to FormSubmit (`FORMSUBMIT_ENDPOINT`). No other backend, and user data goes nowhere else.
- No `alert()`/`confirm()`, and no `!important`. Every user-supplied string rendered into HTML goes through `escapeHtml()`.
- Branding: "UOB IT PMO" text wordmark and a corporate blue palette only. No real logos or imitation of official systems.

## Commands

```bash
# Run: open the file in a browser
open index.html

# Syntax-check the inline script
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/app.js && node --check /tmp/app.js

# Headless render: dump the DOM after JS runs (smoke-test counts, badges)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --virtual-time-budget=2000 --dump-dom "file://$PWD/index.html"

# Screenshot (headless Chrome has a minimum window width of about 500px, so for
# true mobile width, load the page in a 390px-wide iframe instead)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --window-size=1440,1100 --screenshot=/tmp/desk.png "file://$PWD/index.html"

# Guard against forbidden APIs/URLs (should only match the FormSubmit URL)
grep -nE "localStorage|sessionStorage|indexedDB|document\.cookie|https?://|!important|alert\(|confirm\(" index.html
```

## Architecture (inside the `<script>` block)

- **CONFIG at the top.** `FORMSUBMIT_ENDPOINT` is the one place to change the notification email. `STATUSES`, `PROJECTS`, `CATEGORIES` and `PRIORITIES` drive the columns, the `<select>` options and validation. Add values there, not in markup.
- **Single state object**: `state = { tasks, filters, nextId, pendingDelete, openMoveMenu, sendingCount }`. UI toggles such as the inline "Delete? Yes / No" confirm and the "Move ▸" menu are state flags, not DOM tweaks.
- **Render from state.** `buildColumns()` builds the four column shells once. After that, `renderBoard(focusSelector?)` regenerates every card list from `applyFilters(state.tasks)` using `renderCard()` HTML strings, and refreshes the column badges and `renderSummary()`. Never mutate card DOM directly. Change `state`, then call `renderBoard()`.
- **Focus restoration.** Because each render replaces the card DOM, actions pass a CSS selector (built with `selectorFor(action, id)`) to `renderBoard()`, which refocuses that element afterwards. This keeps keyboard users in place. Keep this pattern when adding card controls.
- **Event delegation.** One listener set sits on `#board`. Clicks dispatch on `data-action` (`move-toggle`, `move-to`, `delete`, `delete-yes`, `delete-no`). Drag and drop uses native `dragstart`/`dragover`/`drop`/`dragend`, carrying the task id in `dataTransfer`.
- **Actions**: `addTask`, `moveTask` and `deleteTask` mutate `state.tasks`, then re-render.
- **Column counts and the summary strip differ.** Column badges count the *filtered* tasks. The header summary strip always counts *all* tasks.
- **Add Task flow** (native `<dialog>`, `novalidate` form):
  - `validateForm()` returns an errors map, which `showFormErrors()` displays inline via `f-<key>-error` elements.
  - Field ids follow `f-<key>`, matching `FIELD_KEYS`.
  - A valid submit is optimistic: `addTask()` renders the card immediately, the dialog closes, and `notifyNewTask()` runs in the background in try/catch.
  - A failed notification keeps the card and shows a warning toast.
  - `setSending()` tracks in-flight requests on both the submit and header buttons.
- **Task IDs** are `UOB-ITPM-####` from `state.nextId`. The 8 seed tasks use 0001–0008. Seed due dates are relative to today (`daysFromToday`) so some tasks are always overdue.
- **Overdue** is computed, never stored: `status !== "Done" && dueDate < todayISO()`. Dates are local-time `YYYY-MM-DD` strings compared lexically.

## FormSubmit

- The first submission to a new address triggers an activation email, and nothing is delivered until its link is clicked.
- FormSubmit can return HTTP 200 with `success: "false"`. `notifyNewTask()` treats that as a failure.
- The placeholder address `YOUR_EMAIL@example.com` is intentional in the repo.

## Git & deployment

`.github/workflows/pages.yml` deploys to GitHub Pages (https://ramki2k3.github.io/Claude_Project/) on every push to `main`. It publishes only `index.html`, so any new asset the app needs must be copied into `_site` in that workflow.

The remote is `git@github.com:ramki2k3/Claude_Project.git`. Use SSH: HTTPS pushes on this machine authenticate as a different GitHub account and fail with 403.
