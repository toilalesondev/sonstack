---
name: await-merge-approval
layer: L1
source: oh-my-hermes
description: Sends Sir a Telegram message with PR link + summary, then blocks the merge until Sir replies YES
version: 1.0.0
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
- Hermes can receive and read incoming Telegram messages (polling or webhook)
- GitHub token available in 1Password → Hermes Shared vault (for merge API call)

## Procedure

1. **Gather PR metadata** — collect from GitHub API or local git:

   ```
   tool: run_shell
   command: "gh pr view {pr_number} --repo toilalesondev/{repo} --json title,body,url,additions,deletions,changedFiles,headRefName,baseRefName"
   ```

   Extract:
   - `pr_url` — full GitHub URL
   - `pr_title` — PR title
   - `pr_body` — first 200 chars of description
   - `changed_files` — count of changed files
   - `additions` / `deletions` — line counts
   - `head_branch` → `base_branch` — e.g., `feat/login` → `main`

2. **Check CI status** — do not proceed if checks are failing:

   ```
   tool: run_shell
   command: "gh pr checks {pr_number} --repo toilalesondev/{repo}"
   ```

   If any check is `fail` or `pending`:
   ```
   tool: send_telegram_message
   text: "⏳ PR #{pr_number} has failing/pending CI checks. Waiting for green before requesting approval."
   ```
   Retry after 60 seconds (max 10 retries). If still not green after 10 minutes, notify Sir and **stop**:
   ```
   tool: send_telegram_message
   text: "❌ PR #{pr_number} CI checks did not pass after 10 min. Manual review needed.\n\n{pr_url}"
   ```

3. **Write a plain-English PR summary** (2-4 sentences max):

   Based on `pr_title`, `pr_body`, `changed_files`, `additions`, `deletions`:
   ```
   Summary: "{What this PR does, what problem it solves, what changed at a high level}"
   Risk: "Low / Medium / High" (based on file count and whether it touches auth/DB/payments)
   ```

4. **Send approval request to Sir via Telegram:**

   ```
   tool: send_telegram_message
   text: "🔀 **Merge approval needed**\n\nPR: [{pr_title}]({pr_url})\nBranch: `{head_branch}` → `{base_branch}`\nChanges: +{additions} / -{deletions} across {changed_files} file(s)\n\n📝 Summary: {summary}\n⚠️ Risk: {risk_level}\n\nReply **YES** to merge, **NO** to cancel, or **HOLD** to defer."
   ```

5. **Wait for Sir's reply** — poll Telegram for incoming message:

   ```
   tool: wait_for_telegram_reply
   timeout_minutes: 60
   valid_responses: ["YES", "NO", "HOLD", "yes", "no", "hold"]
   ```

   While waiting:
   - Do NOT proceed with any merge
   - Do NOT time out silently — re-ping Sir after 30 minutes:
     ```
     tool: send_telegram_message
     text: "⏰ Still waiting on merge approval for PR #{pr_number}. Reply YES / NO / HOLD."
     ```
   - If timeout (60 min) is reached with no reply, **abort and notify**:
     ```
     tool: send_telegram_message
     text: "⏰ No reply after 60 min — PR #{pr_number} merge has been **cancelled**. Re-run await-merge-approval when ready."
     ```

6. **Handle Sir's response:**

   **If YES:**
   ```
   tool: run_shell
   command: "gh pr merge {pr_number} --repo toilalesondev/{repo} --squash --delete-branch"
   ```
   Then notify:
   ```
   tool: send_telegram_message
   text: "✅ PR #{pr_number} merged to {base_branch}! Branch `{head_branch}` deleted.\n\nNext: run `deploy-to-vercel` to push live."
   ```

   **If NO:**
   ```
   tool: send_telegram_message
   text: "❌ Merge cancelled by Sir. PR #{pr_number} remains open. Let me know what changes are needed."
   ```
   Log outcome to memory as `status: "rejected"`.

   **If HOLD:**
   ```
   tool: send_telegram_message
   text: "⏸ PR #{pr_number} is on hold. I'll remind you in 24 hours. Let me know when to re-request approval."
   ```
   Schedule a reminder. Log outcome to memory as `status: "on_hold"`.

7. **Save outcome to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/pr_approvals/{pr_number}"
   value: {
     pr_url: "{pr_url}",
     pr_title: "{pr_title}",
     decision: "approved" | "rejected" | "on_hold" | "timed_out",
     decided_at: "<ISO timestamp>",
     merged_at: "<ISO timestamp or null>"
   }
   ```

## Sir's Defaults

- **GitHub account:** `toilalesondev` (SSH auth) — all `gh` commands use this account
- **Merge strategy:** `--squash` by default (keeps `main` history clean); use `--merge` only if Sir explicitly requests
- **Branch deletion after merge:** always `--delete-branch`
- **Protected branches:** `main`, `production` — always require approval
- **Feature branches** (`feat/*`, `fix/*`, `chore/*`) — no approval needed for merging to a staging branch
- **Notifications:** Telegram only (not Slack, not email)
- **PAT for `gh` CLI:** Retrieve from 1Password → Hermes Shared vault if `gh auth status` fails
- **Timeout:** 60 minutes before auto-cancelling
- **Re-ping interval:** 30 minutes

## Pitfalls

- **Never auto-merge without a YES reply** — not even if Sir seems obviously in favor from context. Always wait for explicit YES.
- **Don't request approval for failing CI** — fix the build first (step 2). Sir should only see green PRs.
- **Don't merge to `production` directly** — deploy flow is `feat/*` → `main` → Vercel. There is no separate `production` branch by default.
- **Don't use `--merge` (no-ff) by default** — squash keeps history readable. Only switch if Sir asks.
- **If `gh` CLI is not authenticated**, retrieve the PAT from 1Password before failing:
  ```
  tool: run_shell
  command: "op read 'op://Hermes Shared/GitHub PAT/credential' | gh auth login --with-token"
  ```
- **Telegram reply matching is case-insensitive** but also watch for "Yes please", "ok merge", "go ahead" — treat affirmative natural language as YES.
- **Don't delete the branch before confirming merge succeeded** — check `gh pr view` status post-merge.

## Verification

After merge (step 6, YES path):

```
tool: run_shell
command: "gh pr view {pr_number} --repo toilalesondev/{repo} --json state,mergedAt"
```

Expected: `state: "MERGED"` and `mergedAt` is populated.

Also verify branch is deleted:
```
tool: run_shell
command: "git ls-remote --heads origin {head_branch}"
```

Expected: empty output (branch deleted).

Memory check:
```
tool: memory_read
key: "project/{project_slug}/pr_approvals/{pr_number}"
```

Expected: `decision: "approved"` and `merged_at` is set.
