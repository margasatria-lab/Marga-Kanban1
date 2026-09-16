# UOB IT PMO — Kanban Board (Demo)

A single-file Kanban board web app built for internal demo/training use. This is **not** an official UOB system — it uses a neutral text wordmark and generic corporate-blue styling only, no real UOB branding.

**Live demo:** https://margasatria-lab.github.io/Marga-Kanban1/

## What it does

- Four fixed columns — Backlog, In Progress, Blocked, Done — with drag-and-drop cards, plus a keyboard-accessible "Move ▸" dropdown on every card as a non-drag alternative.
- Add Task modal with client-side validation and optimistic UI (the card appears immediately; a background email notification is sent via FormSubmit and failure only shows a warning toast).
- Inline per-card delete confirmation (no native browser dialogs).
- A filter bar and a live summary of totals, per-status counts, and overdue tasks.
- Responsive layout that stacks into a single column below 768px.

## Running it

No build, server, or install step is required — it's vanilla HTML/CSS/JS in one file.

- Open `index.html` directly in a browser, or
- Visit the [live demo](https://margasatria-lab.github.io/Marga-Kanban1/) hosted via GitHub Pages.

## Demo data — nothing is saved

Board state lives only in the page's in-memory JavaScript. **Refreshing the page resets the board to the seeded sample data** — there is no `localStorage`, database, or backend beyond a single outbound notification call. This is intentional: see `CLAUDE.md` for the full list of hard constraints this project maintains.
