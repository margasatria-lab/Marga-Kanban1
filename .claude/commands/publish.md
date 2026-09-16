---
description: Security-scan, then push this repo to a GitHub URL, wire up GitHub Pages, and update the README + repo About link
argument-hint: <github-repo-url> [branch-name]
allowed-tools: Bash(git:*), Bash(node:*), Bash(grep:*), Read, Write, Edit, Glob, Grep, mcp__github__get_file_contents, mcp__github__create_or_update_file, mcp__github__push_files, mcp__github__get_me
---

## Inputs

- `$1` — the destination GitHub repository URL the user provided (e.g. `https://github.com/<owner>/<repo>`). If missing, ask the user for it before doing anything else — never guess a URL.
- `$2` — optional branch name to push to. Default to the current branch (`git branch --show-current`) if not given.

Parse `<owner>` and `<repo>` out of `$1` for later steps (Pages URL, About link).

## Step 0 — Preconditions

1. `git status` to confirm the working tree state. If there are uncommitted changes the user hasn't asked you to commit, stop and ask — never silently commit unrelated work.
2. Confirm this really is the repo the user wants published (check `git remote -v`; if `origin` already points somewhere else, tell the user and confirm before changing it).

## Step 1 — Mandatory security scan (never skip, never push around a failure)

Before anything touches the network, scan the full working tree (tracked + staged + about-to-be-pushed files) for sensitive data. At minimum check for:

- Hardcoded credentials / secrets: API keys, tokens, passwords, private keys (`-----BEGIN.*PRIVATE KEY-----`), AWS-style keys (`AKIA[0-9A-Z]{16}`), generic `Bearer <token>` strings, connection strings with embedded passwords.
- `.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, credential JSON files, or any file that looks like local secrets that shouldn't be public.
- Personal data beyond what's expected for this project (real emails/phone numbers/addresses that aren't the intentional demo config, e.g. anything outside the documented `FORMSUBMIT_ENDPOINT` constant in `index.html` per `CLAUDE.md`).
- Any accidental internal-only info (internal hostnames, internal URLs, staging credentials) that shouldn't be public on GitHub Pages.

Use `Grep` across the repo (not just `git diff`) for patterns like:
```
(api[_-]?key|secret|password|passwd|token|private[_-]?key)\s*[:=]\s*['"][^'"]{8,}
AKIA[0-9A-Z]{16}
-----BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY-----
```
Also run `git status --porcelain` / `git ls-files` and flag any suspicious filenames (`.env`, `*.pem`, `secrets*`, `credentials*`).

**If anything suspicious is found:** stop, do not push, and report exactly what was found and where (file:line) so the user can decide (redact, remove, or confirm it's intentional demo data like the `FORMSUBMIT_ENDPOINT` constant, which is expected and fine per `CLAUDE.md`). Only proceed past this step once the tree is clean or the user has explicitly confirmed a flagged item is safe to publish.

## Step 2 — Push to the destination repo

1. If `origin` isn't already `$1`, set it: `git remote set-url origin <url>` (or `git remote add origin <url>` if none exists) — confirm with the user first if this changes an existing remote.
2. Push the current (or `$2`) branch: `git push -u origin <branch>`.
3. Follow the repo's standard git push retry policy (retry on network errors only, exponential backoff), never force-push without explicit user confirmation.

## Step 3 — GitHub Pages via GitHub Actions

1. Check for `.github/workflows/deploy-pages.yml`. If it doesn't exist, create it; if it exists, update it so its trigger branch matches the branch actually being pushed in Step 2 (Pages only deploys from pushes to that branch).
2. Use this pattern (vanilla static site, no build step — matches this repo's "no bundler" constraint):
   ```yaml
   name: Deploy to GitHub Pages

   on:
     push:
       branches:
         - <branch-name>
     workflow_dispatch:

   permissions:
     contents: read
     pages: write
     id-token: write

   concurrency:
     group: pages
     cancel-in-progress: false

   jobs:
     deploy:
       environment:
         name: github-pages
         url: ${{ steps.deployment.outputs.page_url }}
       runs-on: ubuntu-latest
       steps:
         - name: Checkout
           uses: actions/checkout@v4
         - name: Configure Pages
           uses: actions/configure-pages@v5
           with:
             enablement: true
         - name: Upload artifact
           uses: actions/upload-pages-artifact@v3
           with:
             path: .
         - name: Deploy to GitHub Pages
           id: deployment
           uses: actions/deploy-pages@v4
   ```
3. Tell the user they must set **Settings → Pages → Source: GitHub Actions** once in the repo UI if it isn't already (the API/workflow alone can't flip that toggle on a brand-new repo).
4. Compute the resulting Pages URL: `https://<owner>.github.io/<repo>/` (project page) — this is what Steps 4 and 5 link to.

## Step 4 — README

Create or update `README.md` at the repo root with:
- Project name/one-line description (pull from `CLAUDE.md` — keep to the same neutral, non-branded framing already established there; don't invent claims about the project).
- A "Live demo" link to the computed Pages URL from Step 3.
- Brief usage notes (open `index.html` directly, or visit the Pages link) and the "no persistence — refresh resets the board" note from `CLAUDE.md`, since that's user-facing behavior worth documenting.
- Keep it accurate to what the code actually does — verify claims against `index.html` rather than assuming.

## Step 5 — Repo "About" section + homepage link

The GitHub "About" panel's homepage link is repo metadata, not something a normal `contents: write` workflow permission can touch — it needs the repository's admin API (`PATCH /repos/{owner}/{repo}` with `homepage` and optionally `description`).

Preferred approach — ask the user which they want:
- **One-time, done now:** if a GitHub MCP tool for updating repository metadata is available, use it to set `description` and `homepage` (the Step 3 Pages URL) directly. Check `ToolSearch` for something like `mcp__github__update_repository` before assuming it's unavailable.
- **Automated on every deploy:** add a step to the Pages workflow (or a small separate workflow) that calls the GitHub REST API with `curl` using `secrets.GITHUB_TOKEN`, gated behind `permissions: administration: write`, e.g.:
  ```yaml
  - name: Update repo About link
    run: |
      curl -sS -X PATCH \
        -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
        -H "Accept: application/vnd.github+json" \
        https://api.github.com/repos/${{ github.repository }} \
        -d '{"homepage":"${{ steps.deployment.outputs.page_url }}"}'
  ```
  This keeps the About link self-updating without a PAT, at the cost of needing that extra permission scope on the token.

Confirm with the user which route they want before adding a workflow step that changes repo settings automatically — that's a standing, low-visibility change to how the repo behaves on every push.

## Step 6 — Commit and push the changes from Steps 3–4

1. `git status` / `git diff` to show the user what changed (workflow file, README).
2. Commit with a clear message (e.g. "Set up GitHub Pages deployment and README").
3. Push per the repo's push policy from Step 2.

## Step 7 — Report back

Summarize for the user:
- What was pushed and to which branch/URL.
- The security scan result (clean, or what was flagged and how it was resolved).
- The Pages URL, and whether they still need to flip Settings → Pages → Source themselves.
- Whether the About/homepage link was set now, or wired to auto-update, or still needs a manual step.

Never perform Step 2 (the push) if Step 1 turned up an unresolved finding.
