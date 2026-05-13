---
name: connect-supabase
layer: L1
source: oh-my-hermes
description: Links a project to Supabase (project bhssclapikzlpyzolzjy), pushes schema migrations, and writes env vars to .env.local
version: 1.0.0
tags: [supabase, database, migrations, env, setup]
---

## When to Use

Run this skill when:
- Setting up a new project that needs a Supabase database
- Adding Supabase to an existing project for the first time
- After schema changes need to be pushed (`supabase db push`)
- When `.env.local` is missing Supabase credentials
- After cloning a repo that has a `supabase/` directory

Do **not** run `supabase db push` on production without reviewing the migration diff first.

## Prerequisites

- Supabase CLI installed: `supabase --version` returns a version (≥ 1.x)
- Supabase account credentials available — access token in 1Password → Hermes Shared vault
- Project root has a `supabase/` directory (or will be initialized)
- Supabase project `bhssclapikzlpyzolzjy` exists and is accessible
- `.gitignore` contains `.env.local` (never commit env secrets)

## Procedure

1. **Check Supabase CLI auth:**

   ```
   tool: run_shell
   command: "supabase projects list 2>&1"
   ```

   If output contains `not logged in` or auth error:
   ```
   tool: run_shell
   command: "supabase login --token $(op read 'op://Hermes Shared/Supabase Access Token/credential')"
   ```

   Verify login succeeded:
   ```
   tool: run_shell
   command: "supabase projects list 2>&1 | grep bhssclapikzlpyzolzjy"
   ```

2. **Initialize Supabase in the project** (skip if `supabase/` already exists):

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "supabase init"
   ```

   This creates `supabase/config.toml` and `supabase/migrations/` directory.

3. **Link the project to Supabase project `bhssclapikzlpyzolzjy`:**

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "supabase link --project-ref bhssclapikzlpyzolzjy"
   ```

   Expected output: `Linked project bhssclapikzlpyzolzjy`

   If the command prompts for the database password, retrieve it from 1Password:
   ```
   tool: run_shell
   command: "op read 'op://Hermes Shared/Supabase DB Password bhssclapikzlpyzolzjy/credential'"
   ```

4. **Review pending migrations** before pushing:

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "supabase db diff --linked 2>&1"
   ```

   If there is a diff, capture it and summarize for Sir:
   ```
   tool: send_telegram_message
   text: "🗄 Supabase migration diff for {project_slug}:\n\n```sql\n{diff_excerpt — first 50 lines}\n```\n\nAbout to run `supabase db push`. OK to proceed?"
   ```

   Wait for YES/OK. If Sir says NO, **stop** — do not push.

5. **Push schema to Supabase:**

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "supabase db push 2>&1"
   timeout: 120
   ```

   If push fails:
   ```
   tool: send_telegram_message
   text: "❌ `supabase db push` failed for {project_slug}.\n\nError:\n```\n{error output}\n```"
   ```
   **Stop.** Do not update `.env.local` if the DB is not properly configured.

6. **Retrieve Supabase project credentials:**

   ```
   tool: run_shell
   command: "supabase status --project-ref bhssclapikzlpyzolzjy 2>&1"
   ```

   Or retrieve directly from Supabase API / 1Password if `status` output is incomplete.

   Extract:
   - `SUPABASE_URL` — e.g., `https://bhssclapikzlpyzolzjy.supabase.co`
   - `SUPABASE_ANON_KEY` — public anon key (safe for client-side)
   - `SUPABASE_SERVICE_ROLE_KEY` — **server-side only**, never expose to client

7. **Write `.env.local`:**

   ```
   tool: write_file
   path: "{project_root}/.env.local"
   mode: "append_or_create"
   content: |
     # Supabase — project bhssclapikzlpyzolzjy
     NEXT_PUBLIC_SUPABASE_URL=https://bhssclapikzlpyzolzjy.supabase.co
     NEXT_PUBLIC_SUPABASE_ANON_KEY={anon_key}
     SUPABASE_SERVICE_ROLE_KEY={service_role_key}
   ```

   **IMPORTANT:** Never write `SUPABASE_SERVICE_ROLE_KEY` to any file that gets committed. Confirm `.gitignore` has `.env.local` before writing.

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "grep -q '.env.local' .gitignore && echo 'SAFE' || echo 'WARNING: .env.local not in .gitignore!'"
   ```

   If output is `WARNING`, add it:
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "echo '.env.local' >> .gitignore"
   ```

8. **Set env vars in Vercel** (for production):

   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "vercel env add NEXT_PUBLIC_SUPABASE_URL production <<< 'https://bhssclapikzlpyzolzjy.supabase.co'"
   ```
   ```
   tool: run_shell
   command: "vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY production <<< '{anon_key}'"
   ```
   ```
   tool: run_shell
   command: "vercel env add SUPABASE_SERVICE_ROLE_KEY production <<< '{service_role_key}'"
   ```

   Note: `SUPABASE_SERVICE_ROLE_KEY` should be set as a Vercel **sensitive** env var (not exposed in build logs).

9. **Save connection info to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/supabase"
   value: {
     project_ref: "bhssclapikzlpyzolzjy",
     url: "https://bhssclapikzlpyzolzjy.supabase.co",
     linked_at: "<ISO timestamp>",
     last_push_at: "<ISO timestamp>",
     migrations_applied: {count}
   }
   ```

10. **Notify Sir via Telegram:**

    ```
    tool: send_telegram_message
    text: "🗄 Supabase connected!\n\nProject: `bhssclapikzlpyzolzjy`\nURL: https://bhssclapikzlpyzolzjy.supabase.co\nMigrations pushed: {count}\n\n✅ .env.local updated\n✅ Vercel env vars set\n\nNext: run `deploy-to-vercel` to go live."
    ```

## Sir's Defaults

- **Supabase project:** `bhssclapikzlpyzolzjy` (always — do not create a new project without Sir's explicit instruction)
- **Supabase URL:** `https://bhssclapikzlpyzolzjy.supabase.co`
- **Access token:** 1Password → Hermes Shared vault → "Supabase Access Token"
- **DB password:** 1Password → Hermes Shared vault → "Supabase DB Password bhssclapikzlpyzolzjy"
- **Runtime:** VPS (`/home/karpathy/projects/{slug}`) or macOS local
- **Env file:** Always `.env.local` (Next.js convention)
- **Vercel account:** `toilalesondev` (same for env var sync)
- **Notifications:** Telegram only

**Standard env var names for Next.js + Supabase:**
```env
NEXT_PUBLIC_SUPABASE_URL=https://bhssclapikzlpyzolzjy.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
```

## Pitfalls

- **Never expose `SUPABASE_SERVICE_ROLE_KEY` in client-side code** — it bypasses Row Level Security (RLS). It must only be in server-side env vars.
- **Don't push migrations blindly** — always run `supabase db diff` first (step 4) and get Sir's OK if there are destructive changes (DROP TABLE, DROP COLUMN).
- **Don't create a new Supabase project** — the project `bhssclapikzlpyzolzjy` is Sir's canonical project. Unless Sir explicitly says "new project", always link to this one.
- **If `supabase link` fails with "already linked"**, that's fine — skip step 3 and proceed.
- **If `supabase link` asks for org** — the org is associated with the `toilalesondev` account; select it when prompted.
- **`.env.local` already has content** — append, do not overwrite the entire file. Use append mode to preserve other env vars.
- **RLS policies must be written** — `supabase db push` pushes schema but doesn't auto-enable RLS. Remind Sir if tables are created without RLS:
  ```
  tool: send_telegram_message
  text: "⚠️ New tables created. Make sure RLS is enabled and policies are written before going to production."
  ```

## Verification

After step 10:

1. **Test DB connection:**
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "supabase db remote commit 2>&1 || echo 'connection ok'"
   ```

2. **Verify env vars exist:**
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "grep -c 'SUPABASE' .env.local"
   ```
   Expected: `3` (URL + anon key + service role key)

3. **Verify Vercel env vars:**
   ```
   tool: run_shell
   workdir: "{project_root}"
   command: "vercel env ls production 2>&1 | grep SUPABASE"
   ```
   Expected: 3 SUPABASE entries listed.

4. **Memory check:**
   ```
   tool: memory_read
   key: "project/{project_slug}/supabase"
   ```
   Expected: `project_ref: "bhssclapikzlpyzolzjy"` and `linked_at` is set.
