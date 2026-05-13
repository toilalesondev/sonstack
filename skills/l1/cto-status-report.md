---
name: cto-status-report
layer: L1
source: oh-my-hermes
description: Reads git log + open PRs + memory context, writes a plain-English CTO-style status summary, and sends it to Sir via Telegram
version: 1.0.0
tags: [reporting, status, telegram, git, summary]
---

## When to Use

Run this skill to give Sir a high-level status update on a project. Use it:
- At the end of a work session ("wrap up and report")
- On a daily or weekly schedule (cron job)
- When Sir asks "what's the status?", "where are we?", "give me a summary"
- Before a demo or investor meeting
- After a major deployment to confirm everything is healthy

## Prerequisites

- Git repo exists at `{project_root}` with a remote on `github.com/toilalesondev/{repo}`
- `gh` CLI is authenticated (or PAT in 1Password → Hermes Shared vault)
- Hermes memory is readable for project context
- Telegram notification channel is configured

## Procedure

1. **Load project context from memory:**

   ```
   tool: memory_read
   key: "project/{project_slug}/context"
   ```

   Also read:
   ```
   tool: memory_read
   key: "project/{project_slug}/last_deploy"
   ```
   ```
   tool: memory_read
   key: "project/{project_slug}/supabase"
   ```
   ```
   tool: memory_read
   key: "project/{project_slug}/live_url"
   ```

2. **Get recent git activity (last 7 days):**

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "git log --oneline --since='7 days ago' --all 2>&1 | head -30"
   ```

   Also get:
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "git log --oneline -1 2>&1"
   ```
   (last commit SHA + message for the header)

   Count commits:
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "git log --oneline --since='7 days ago' | wc -l"
   ```

3. **Get open pull requests:**

   ```
   tool: run_shell
   command: "gh pr list --repo toilalesondev/{repo} --state open --json number,title,author,createdAt,url 2>&1"
   ```

   Format into a short list.

4. **Get merged PRs in the last 7 days:**

   ```
   tool: run_shell
   command: "gh pr list --repo toilalesondev/{repo} --state merged --json number,title,mergedAt,url --limit 10 2>&1 | jq '[.[] | select(.mergedAt > \"{7_days_ago_iso}\")]'"
   ```

5. **Check live URL health** (if deployed):

   ```
   tool: run_shell
   command: "curl -o /dev/null -s -w '%{http_code}' {live_url}"
   ```

   Record: `200` = healthy, anything else = degraded.

6. **Check for any failed GitHub Actions in the last 24h:**

   ```
   tool: run_shell
   command: "gh run list --repo toilalesondev/{repo} --limit 5 --json status,conclusion,name,createdAt 2>&1"
   ```

   Flag any `conclusion: failure`.

7. **Compose the plain-English status report:**

   Write a short, scannable report using this structure:

   ```markdown
   ## {Project Name} — Status Report
   {date, e.g. "Mon 13 May 2025"}

   ### 🚀 Deployment
   - Live URL: {url or "not deployed"}
   - Last deploy: {relative time, e.g. "2 hours ago"} — commit {sha_short}: "{message}"
   - Health: ✅ HTTP 200 / ⚠️ {issue}

   ### 📦 Recent Work (last 7 days)
   - {N} commits merged
   - Highlights:
     • {commit or PR summary 1}
     • {commit or PR summary 2}
     • {commit or PR summary 3}

   ### 🔀 Open PRs ({count})
   {list of open PRs with title + link, or "None"}

   ### 🗄 Database
   - Supabase: bhssclapikzlpyzolzjy
   - Last migration push: {relative time or "never"}
   - Status: ✅ Linked / ⚠️ Issue

   ### ⚠️ Issues / Blockers
   {Any failing CI, failed deploys, degraded health, or open questions from memory}
   {Or: "None — all green ✅"}

   ### 📋 Next Steps
   {Read from memory: project/{project_slug}/next_steps if set, otherwise derive from open PRs and context}
   ```

8. **Write the report to a file** (optional, for audit trail):

   ```
   tool: write_file
   path: "{project_root}/reports/status-{YYYY-MM-DD}.md"
   content: {report content}
   ```

9. **Save report summary to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/last_status_report"
   value: {
     generated_at: "<ISO timestamp>",
     commits_last_7d: {count},
     open_prs: {count},
     live_url: "{url}",
     health_status: "healthy" | "degraded" | "down" | "not_deployed",
     report_path: "{path}"
   }
   ```

10. **Send to Sir via Telegram:**

    For the Telegram message, use a condensed version (Telegram has a 4096 char limit):

    ```
    tool: send_telegram_message
    text: "📊 **{Project Name} — Status Report**\n{date}\n\n🚀 **Deploy:** {url} · {relative time} · {sha_short}\n💚 Health: {status}\n\n📦 **Last 7 days:** {N} commits\n• {highlight 1}\n• {highlight 2}\n• {highlight 3}\n\n🔀 **Open PRs:** {count}\n{pr list or 'None'}\n\n⚠️ **Blockers:** {blockers or 'None'}\n\n📋 **Next:** {next step 1}"
    ```

    If the full report is long, also send the file:
    ```
    tool: send_telegram_file
    path: "{project_root}/reports/status-{YYYY-MM-DD}.md"
    caption: "Full status report — {project_name} {date}"
    ```

## Sir's Defaults

- **Telegram only** — never send status reports to Slack or email
- **GitHub account:** `toilalesondev` — all `gh` commands use this account
- **Report language:** English by default; if Sir has set `language: vi` in memory, write the narrative sections in Vietnamese but keep section headers in English
- **Supabase project:** `bhssclapikzlpyzolzjy` — always reference this in the DB section
- **VPS workdir:** `/home/karpathy/projects/{project_slug}`
- **Report file location:** `{project_root}/reports/` (create if missing)
- **Schedule:** Can be triggered manually or via cron — store cron schedule in memory under `project/{project_slug}/report_schedule`
- **PAT for `gh`:** 1Password → Hermes Shared vault if `gh auth status` fails

**Tone guidelines:**
- Write like a CTO briefing a founder: direct, no fluff, highlight blockers
- Use relative time ("2 hours ago", "yesterday") not raw timestamps
- Flag anything that needs Sir's attention with ⚠️
- Keep the Telegram message under 3000 characters

## Pitfalls

- **Don't fabricate status** — if git log is empty or `gh` fails, report that honestly rather than making up activity.
- **Don't include raw commit hashes in the Telegram message** — use the first 7 chars and the commit message.
- **If the project has no Vercel deployment yet**, skip the deployment section and note "Not yet deployed."
- **If `gh pr list` returns an error** (auth issue), retrieve the PAT from 1Password before failing:
  ```
  tool: run_shell
  command: "op read 'op://Hermes Shared/GitHub PAT/credential' | gh auth login --with-token"
  ```
- **Don't write sensitive info to the report file** — no env vars, no API keys, no passwords. The `reports/` directory may be in the repo.
- **Add `reports/` to `.gitignore`** if it isn't already — status reports are operational, not source:
  ```
  tool: run_shell
  workdir: "{project_root}"
  command: "grep -q 'reports/' .gitignore || echo 'reports/' >> .gitignore"
  ```
- **If the project has no activity in 7 days**, don't generate a fake summary. Report:
  `"📦 Last 7 days: No commits. Last activity: {relative time of last commit}."`

## Verification

After step 10:

1. **Telegram message delivered:**
   Check for delivery confirmation from Telegram API (message ID returned).

2. **Report file written:**
   ```
   tool: run_shell
   command: "ls -la {project_root}/reports/ | grep status-{YYYY-MM-DD}"
   ```
   Expected: file exists with non-zero size.

3. **Memory updated:**
   ```
   tool: memory_read
   key: "project/{project_slug}/last_status_report"
   ```
   Expected: `generated_at` is within the last 5 minutes.

4. **No sensitive data leaked:**
   ```
   tool: run_shell
   command: "grep -i 'service_role\\|password\\|secret\\|token' {project_root}/reports/status-{YYYY-MM-DD}.md"
   ```
   Expected: no matches.
