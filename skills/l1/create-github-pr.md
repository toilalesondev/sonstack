---
name: create-github-pr
layer: L1
source: oh-my-hermes
description: Open a pull request on GitHub after implementation is complete on a feature branch.
version: 1.1.0
tags: [github, pr, pull-request, ship]
---

## When to Use
After G08 (build) passes. Before G11 (merge approval). Implementation complete on feature branch.

## Prerequisites
- `gh` CLI authenticated as toilalesondev
- Feature branch pushed to GitHub
- Tests passing

## Procedure

1. **Confirm on feature branch — never PR from main**

   ```bash
   git branch --show-current
   ```

   If output is `main` or `production`: **stop**. Switch to the correct feature branch first.

2. **List commits to include in PR**

   ```bash
   git log main..HEAD --oneline
   ```

   Collect these commits — they inform the PR title and body.

3. **Show files changed**

   ```bash
   git diff main...HEAD --stat
   ```

   Review scope: how many files, which directories. This informs the risk level.

4. **Compose PR title and body**

   **Title** — conventional commits format:
   - `feat: <description>` — new feature
   - `fix: <description>` — bug fix
   - `chore: <description>` — tooling, deps, config
   - `refactor: <description>` — no behavior change
   - Keep under 72 chars

   **Body** template:
   ```markdown
   ## What changed
   <1-3 sentences describing what was built or fixed>

   ## Commits
   <paste git log output from step 2>

   ## How to test
   1. <step 1>
   2. <step 2>
   3. Expected: <what success looks like>

   ## Screenshots
   <include if UI changed — use browser_screenshot if needed>
   ```

5. **Create the PR**

   ```bash
   gh pr create --title '<title from step 4>' --body '<body from step 4>' --base main
   ```

   Capture the PR URL printed to stdout (e.g., `https://github.com/toilalesondev/{repo}/pull/{number}`).

6. **Save PR URL to memory**

   ```
   tool: memory_write
   key: "project/{project_slug}/current_pr_url"
   value: "<PR URL>"
   ```

7. **Wait for CI — poll statusCheckRollup**

   ```bash
   gh pr view --json statusCheckRollup
   ```

   Poll every 30 seconds, max 5 minutes (10 polls). On each poll, check if all checks are `SUCCESS`.

   - If green within 5 min: proceed to step 8.
   - If still pending after 5 min: report to Sir with current status and wait for instruction.
   - If any check fails: report failure to Sir immediately; do not proceed to `await-merge-approval`.

8. **Report to Sir**

   Reply inline with:
   - PR URL
   - CI status (green ✅ / pending ⏳ / failed ❌)
   - Files changed count (from step 3)
   - Reminder: "Run `await-merge-approval` when ready to merge"

## Sir's Defaults
- Base branch: `main`
- GitHub account: toilalesondev
- PR title format: conventional commits (`feat:` / `fix:` / `chore:`)
- Always include "How to test" in PR body
- Include screenshots in body if any UI changed

## Pitfalls
- Never open PR from `main` → `main`
- Do not skip step 3 (`git diff --stat`) — scope review prevents surprise large PRs
- Always include "How to test" — Sir cannot review a PR without knowing how to verify it
- If CI fails after PR open, fix before running `await-merge-approval` (that skill also gates on green CI)
- `gh pr create` will open your `$EDITOR` if body is empty — always provide `--body`

## Verification
Pass: `gh pr view` returns PR in `OPEN` state, CI checks started, PR URL saved in memory
