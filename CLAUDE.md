# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`index.html` is a single-file Kanban board web app for an internal "UOB IT PMO" demo/training tool. It is **not** an official UOB system — it uses a neutral text wordmark and generic corporate-blue styling only, never real UOB branding.

## Hard constraints (do not violate)

- **Vanilla HTML/CSS/JS only.** No frameworks (React, Vue, jQuery, Tailwind), no build step, no bundler, no npm dependencies at runtime.
- **Single file.** All markup, the `<style>` block, and the `<script>` block live in `index.html`. It must run by double-clicking the file in a browser — no server or build required.
- **No persistence.** Board state lives only in the in-memory `state` object in the page's JS. Do not add `localStorage`, `sessionStorage`, `IndexedDB`, cookies, or any other storage API — a page refresh is *meant* to reset the board to the seeded demo data, and the UI shows a note saying so.
- **No external resources.** No CDN scripts/styles, no Google Fonts, no image files. Icons are inline SVG or Unicode glyphs; fonts come from the system font stack.
- **Outbound network calls go only through FormSubmit** (`formsubmit.co`), used purely to email-notify on new task creation. Never add any other backend call, and never send the user's email anywhere except that one endpoint.

## Architecture (all inside `index.html`)

- **State**: a single `state = { tasks: [], filters: {}, nextId }` object is the source of truth. All mutations (`addTask`, `moveTask`, `requestDelete`/`cancelDelete`/`deleteTask`) update `state` and then call `renderBoard()` — there is no direct DOM mutation of card contents outside the render functions.
- **Rendering**: `renderBoard()` rebuilds all four columns from `getFilteredTasks()`; `renderCard()` builds one card's HTML; `renderSummary()` builds the live total/per-status/overdue counts in the header. Every render pass re-attaches drag-and-drop listeners (`attachDragAndDropHandlers`) since the DOM is regenerated each time.
- **Columns are fixed**: `COLUMNS = ["Backlog", "In Progress", "Blocked", "Done"]` — column identity is the task's `status` field, matched by exact string.
- **Drag-and-drop** uses the native HTML5 DnD API (`draggable`, `dragstart`/`dragover`/`drop`) and updates `task.status` on drop. A parallel keyboard-accessible "Move ▸" `<select>` on every card calls the same `moveTask()` function, so DnD is never the only way to change a task's column.
- **Delete confirmation** is inline per-card (`confirmingDelete` flag on the task object toggles a "Delete? Yes/No" block), not a native `confirm()` dialog.
- **Add Task flow**: modal form → `validateForm()` (client-side only, inline error text per field, no `alert()`) → `addTask()` adds the task to `state.tasks` and re-renders immediately (optimistic UI) → `notifyNewTask()` fires the FormSubmit AJAX call in parallel and is wrapped in try/catch, so a network failure only shows a warning toast and never removes the card or blocks the board.
- **All user-supplied strings are escaped** via the `escapeHtml()` helper before being interpolated into template strings — never bypass this when adding new rendered fields.
- **Config constant**: `FORMSUBMIT_ENDPOINT` at the top of the `<script>` block is the one place to change the notification email address. Note FormSubmit requires a one-time confirmation click on the target address before it actually delivers emails.

## Developing / testing changes

There is no build, lint, or test tooling in this repo. To validate a change:

- **Syntax-check the embedded script** (fast, no browser needed):
  ```bash
  node -e "
  const fs = require('fs');
  const html = fs.readFileSync('index.html', 'utf8');
  const m = html.match(/<script>([\s\S]*)<\/script>/);
  new Function(m[1]);
  console.log('Script syntax OK');
  "
  ```
- **Visually/functionally verify in a real browser** — open `index.html` directly (or use the `run` skill / a headless Chromium via Playwright if available) and exercise: seeded board render, drag-and-drop between columns, the keyboard "Move ▸" fallback, Add Task validation and optimistic add, inline delete confirm, filter bar, and the sub-768px stacked layout.
- Do not add a package.json, test runner, or linter config unless explicitly asked — that would contradict the "no build step" constraint above.

## Deployment

`.github/workflows/deploy-pages.yml` deploys the repo root to GitHub Pages via `actions/upload-pages-artifact` + `actions/deploy-pages` on every push to `claude/uob-it-pmo-kanban-yjv6vd` (or manual `workflow_dispatch`). GitHub Pages must be set to "Source: GitHub Actions" in repo settings for this to take effect. Update the trigger branch here if the default branch changes.
