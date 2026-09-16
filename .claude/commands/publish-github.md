---
description: Refresh the README screenshot, commit, push, and open/update a PR for this branch
---

Publish the current state of this repo to GitHub. Do the following in order:

1. **Refresh the screenshot.** Use Playwright (headless Chromium) to open `index.html` directly from the filesystem (`file://` URL, no server needed), wait for the board to render, and save a screenshot to `screenshot.png` at the repo root, overwriting the existing one. Use a desktop-sized viewport (around 1440x960) so the full four-column board is visible.
2. **Update the README.** Ensure `README.md` embeds `screenshot.png` (e.g. `![Screenshot of the UOB IT PMO Kanban board](screenshot.png)`) and that its description still matches the current app. Update the copy if the app has changed since the last publish.
3. **Validate.** Run the syntax-check from `CLAUDE.md`'s "Developing / testing changes" section against `index.html` before committing.
4. **Commit** the changes (screenshot, README, and any other pending work) with a clear, descriptive message.
5. **Push** to the current branch with `git push -u origin <branch-name>`.
6. **Open or update a pull request** for the branch against the repo's default branch, following the repo's PR template if one exists. If a PR already exists for this branch, just leave the push to update it rather than opening a duplicate.

Skip steps that have nothing to do (e.g. don't re-screenshot if `index.html` hasn't changed since the last screenshot and the README already reflects it — but on an explicit `/publish-github` invocation, prefer refreshing the screenshot anyway so it never goes stale).
