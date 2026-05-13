---
name: await-merge-approval
layer: L1
source: oh-my-hermes
description: Sends Sir a Telegram message with PR link + summary, then blocks the merge until Sir replies YES
version: 1.1.0
tags: [github, pr, approval, telegram, merge, gating]
---

## When to Use

Run this skill **before merging any pull request** to a protected branch (`main`, `production`). Use it when:
- A PR is ready for review and merge
- After all CI checks have passed
- Before running `deploy-to-vercel` (approval gate)
- Any time Sir should have final say before code goes to production

Never merge to `main` without going through this skill.

## Prerequisites

- A GitHub PR exists with a URL (e.g., `https://github.com/toilalesondev/{repo}/pull/{number}`)
- CI checks are green (don't request approval for a failing PR)
- Telegram notification channel is configured with Sir's chat ID
- GitHub token available in 1Password → Hermes Shared vault (for merge API call)

## Procedure

1. **Get PR status from GitHub**

   ```bash
   gh pr view <PR_NUMBER> --json title,body,url,statusCheckRollup
   ```

   Extract: `title`, `url`, and `statusCheckRollup` (array of CI check results).

2. **Verify CI is green before sending to Sir**

   Inspect `statusCheckRollup`. All checks must be `SUCCESS` or `NEUTRAL`.

   If any check is `FAILURE` or `PENDING`:
   ```
   tool: send_message
   target: telegram
   message: "⏳ PR #<NUMBER> has failing/pending CI. Waiting for green before requesting approval."
   ```
   Retry after 60s, max 10 retries (~10 min). If still not green after 10 min, notify Sir and **stop**:
   ```
   tool: send_message
   target: telegram
   message: "❌ PR #<NUMBER> CI did not pass after 10 min. Manual review needed.\n\n<url>"
   ```

3. **Send merge approval request to Sir**

   ```
   tool: send_message
   target: telegram
   message: "🔀 Ready to merge: <title>\n<url>\n\nReply YES to merge or NO to reject."
   ```

   Keep this message short — title + URL + the YES/NO prompt. No tables, no long summaries.

4. **⛔ BLOCKING GATE — Karpathy stops here**

   Karpathy **cannot poll Telegram** in a skill. This is a hard stop.

   - Do NOT proceed with any merge
   - Do NOT assume Sir's intent from prior context
   - Sir must reply in **this same Telegram conversation**
   - Karpathy reads Sir's **next message** as the YES/NO response

   When Sir's reply arrives (in the next conversation turn):

   **If YES (or affirmative: "ok", "go ahead", "merge it"):**
   ```bash
   gh pr merge <NUMBER> --merge --delete-branch
   ```
   Then confirm:
   ```bash
   gh pr view <NUMBER> --json state,mergedAt
   ```
   Expected: `state: "MERGED"`.

   **If NO (or rejection: "don't", "cancel", "stop"):**
   Log rejection to memory:
   ```
   tool: memory_write
   key: "project/{project_slug}/pr_approvals/<NUMBER>"
   value: { decision: "rejected", decided_at: "<ISO timestamp>", pr_url: "<url>" }
   ```
   Reply to Sir: "❌ Merge cancelled. PR #<NUMBER> remains open. Let me know what changes are needed."

5. **Save outcome to memory**

   ```
   tool: memory_write
   key: "project/{project_slug}/pr_approvals/<NUMBER>"
   value: {
     pr_url: "<url>",
     pr_title: "<title>",
     decision: "approved" | "rejected",
     decided_at: "<ISO timestamp>",
     merged_at: "<ISO timestamp or null>"
   }
   ```

## Sir's Defaults

- **GitHub account:** `toilalesondev` (SSH auth)
- **Merge strategy:** `--merge` (default); use `--squash` only if Sir explicitly requests
- **Branch deletion after merge:** always `--delete-branch`
- **Protected branches:** `main`, `production` — always require approval
- **Notifications:** Telegram only

## Pitfalls

- **Karpathy cannot auto-detect Sir's reply** — Sir must reply in the SAME Telegram conversation. Karpathy reads Sir's next message as the YES/NO. There is no background polling.
- **Never auto-merge without an explicit YES** — not even if Sir seems obviously in favor from prior context
- **Do not request approval for failing CI** — fix the build first (step 2 gates this)
- **Do not merge to `production` directly** — deploy flow is `feat/*` → `main` → Vercel
- **If `gh` CLI not authenticated**, retrieve PAT from 1Password:
  ```bash
  op read 'op://Hermes Shared/GitHub PAT/credential' | gh auth login --with-token
  ```
- **Natural language YES is valid**: "Yes please", "ok merge", "go ahead" all count as YES
- **Do not delete branch before confirming merge succeeded** — verify `state: "MERGED"` first

## Verification

After merge:
```bash
gh pr view <NUMBER> --json state,mergedAt
```
Expected: `state: "MERGED"` + `mergedAt` populated.

Branch deleted:
```bash
git ls-remote --heads origin <head_branch>
```
Expected: empty output.

Memory check:
```
tool: memory_read
key: "project/{project_slug}/pr_approvals/<NUMBER>"
```
Expected: `decision: "approved"` + `merged_at` set.
