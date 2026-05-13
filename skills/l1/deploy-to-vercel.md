---
name: deploy-to-vercel
layer: L1
source: oh-my-hermes
description: Runs vercel deploy --prod, captures the live URL, and saves it to memory + notifies Sir via Telegram
version: 1.0.0
tags: [deploy, vercel, production, release]
---

## When to Use

Run this skill when it's time to push a new build to production. Use it:
- After `await-merge-approval` has returned YES and the PR is merged to `main`
- When Sir says "deploy this", "push to prod", "go live"
- As the final step in a feature release flow

Do **not** run this on an unmerged branch unless Sir explicitly requests a preview deploy.

## Prerequisites

- Vercel CLI is installed: `vercel --version` returns a version
- Vercel project is linked to the repo (`.vercel/project.json` exists in project root)
- All environment variables are set in Vercel dashboard (not just `.env.local`)
- GitHub repo `toilalesondev/{repo}` has latest changes merged to `main`
- Local workdir is clean (`git status` shows no uncommitted changes) — or deploy from CI
- Vercel token available (either via `vercel login` session or `VERCEL_TOKEN` env var from 1Password)

## Procedure

1. **Confirm deploy target** — read project context from memory:

   ```
   tool: memory_read
   key: "project/{project_slug}/context"
   ```

   Verify `vercel_project_id` or `vercel_team` is set. If not, run Vercel link first (see Pitfalls).

2. **Pull latest from main** before deploying (if deploying from VPS):

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "git checkout main && git pull origin main --ff-only"
   ```

   If `git pull` fails (conflict or diverged), **stop** and notify Sir:
   ```
   tool: send_telegram_message
   text: "⚠️ git pull failed on main — there may be a conflict. Check {project_slug} before deploying."
   ```

3. **Run pre-deploy checks:**

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "npm run build 2>&1"
   ```

   If build fails:
   ```
   tool: send_telegram_message
   text: "❌ Build failed for {project_slug}. Fix errors before deploying.\n\nError:\n```\n{last 20 lines of build output}\n```"
   ```
   **Stop.** Do not deploy a broken build.

4. **Run vercel deploy --prod:**

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "vercel deploy --prod --yes 2>&1"
   timeout: 300
   ```

   Capture full output. Extract the production URL from the last line matching:
   `https://{project}.vercel.app` or custom domain if configured.

5. **Verify deploy succeeded** — check for success signal in output:

   Look for: `✅ Production` or `Deployed to production` in Vercel CLI output.
   
   Also confirm via Vercel API:
   ```
   tool: run_shell
   command: "vercel ls --prod 2>&1 | head -5"
   ```

   If deploy failed (exit code ≠ 0 or error in output):
   ```
   tool: send_telegram_message
   text: "❌ Vercel deploy failed for {project_slug}.\n\nError:\n```\n{error excerpt}\n```\n\nCheck Vercel dashboard: https://vercel.com/toilalesondev/{project_slug}"
   ```
   **Stop.** Do not update memory with a failed deploy URL.

6. **Save live URL to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/live_url"
   value: "{production_url}"
   ```

   ```
   tool: memory_write
   key: "project/{project_slug}/last_deploy"
   value: {
     url: "{production_url}",
     deployed_at: "<ISO timestamp>",
     git_sha: "{current HEAD sha}",
     git_branch: "main",
     deploy_triggered_by: "hermes"
   }
   ```

7. **Run a smoke test** — curl the live URL:

   ```
   tool: run_shell
   command: "curl -o /dev/null -s -w '%{http_code}' {production_url}"
   ```

   If response code is not `200`:
   ```
   tool: send_telegram_message
   text: "⚠️ Deploy succeeded but smoke test returned HTTP {status_code} at {production_url}. Check the app manually."
   ```

8. **Notify Sir via Telegram:**

   On success:
   ```
   tool: send_telegram_message
   text: "🚀 **{project_slug} is live!**\n\n🔗 {production_url}\n\n📦 Deployed: {git_sha_short} on main\n⏱ {deploy_timestamp}\n\nSmoke test: ✅ HTTP 200"
   ```

   Include a direct link to the Vercel deployment dashboard:
   `https://vercel.com/toilalesondev/{project_slug}/deployments`

## Sir's Defaults

- **Deploy target:** Vercel production (`--prod` flag always used)
- **Deploy account:** `toilalesondev` on Vercel (same as GitHub account)
- **VPS workdir:** `/home/karpathy/projects/{project_slug}` (karpathy user)
- **macOS local:** `~/projects/{project_slug}`
- **Vercel token:** If CLI session expired, retrieve `VERCEL_TOKEN` from 1Password → Hermes Shared vault:
  ```
  tool: run_shell
  command: "export VERCEL_TOKEN=$(op read 'op://Hermes Shared/Vercel Token/credential') && vercel deploy --prod --yes --token $VERCEL_TOKEN"
  ```
- **Notifications:** Telegram only (never Slack)
- **Post-deploy:** Always run smoke test (curl) and report HTTP status to Sir
- **Environment variables:** Never put secrets in `.env.local` for production — they must be set in Vercel dashboard or via `vercel env add`
- **Supabase connection:** If app uses Supabase project `bhssclapikzlpyzolzjy`, verify `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are set in Vercel env vars before deploying

## Pitfalls

- **Don't deploy from a feature branch to `--prod`** without Sir's explicit instruction. Always deploy from `main`.
- **Don't skip the build check (step 3)** — Vercel will build again in the cloud, but catching errors locally saves time and avoids a failed deploy notification.
- **If `.vercel/project.json` is missing** (fresh clone), link the project first:
  ```
  tool: run_shell
  workdir: "{project_root}"
  command: "vercel link --yes"
  ```
  If `vercel link` prompts for team/project selection, the CLI may hang — use `--project {project_name}` and `--scope toilalesondev` flags.
- **Don't capture a preview URL as the production URL** — Vercel outputs multiple URLs. Only the one marked `Production:` or the `.vercel.app` canonical URL is the production URL.
- **Custom domains:** If the project has a custom domain, update memory with the custom domain, not the `.vercel.app` URL.
- **If deploy takes > 5 minutes**, check Vercel dashboard — it may be stuck. Send Sir an interim update:
  ```
  tool: send_telegram_message
  text: "⏳ Deploy of {project_slug} is taking longer than expected. Monitoring... ETA unknown."
  ```

## Verification

After deploy (step 8):

1. **Live URL responds:**
   ```
   tool: run_shell
   command: "curl -I {production_url} 2>&1 | head -3"
   ```
   Expected: `HTTP/2 200`

2. **Memory is updated:**
   ```
   tool: memory_read
   key: "project/{project_slug}/last_deploy"
   ```
   Expected: `url` and `deployed_at` are set.

3. **Vercel dashboard shows green:**
   `https://vercel.com/toilalesondev/{project_slug}/deployments`
   — Latest deployment should show `Ready` status.
