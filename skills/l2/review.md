---
name: review
layer: L2
source: gstack
description: Senior code review of a feature branch — finds bugs that pass CI but break in prod
version: 1.0.0
tags: [code-review, git, quality, security, testing]
---

## When to Use

- Before merging any PR to main
- After a feature is "done" but before QA
- When CI is green but something feels off
- When a junior engineer's PR needs a senior read
- Anytime the change touches auth, payments, or data deletion

## Prerequisites

- Feature branch exists and has commits relative to `main`
- Git is available in the terminal
- Source files are readable
- `main` branch is up to date locally

## Procedure

1. **Get the diff**
   ```
   terminal("git diff main...HEAD")
   terminal("git log main...HEAD --oneline")
   terminal("git diff main...HEAD --stat")
   ```
   If the diff is very large (> 500 lines), also get the file list first and review file by file:
   ```
   terminal("git diff main...HEAD --name-only")
   ```

2. **Read changed files in full context** (not just the diff)
   For each changed file, read the full file to understand surrounding context:
   ```
   read_file("<changed_file_path>")
   ```
   The diff shows what changed; the file shows whether it makes sense.

3. **Run the 5 review lenses:**

   **Lens 1 — Bugs that pass CI but break in prod**
   Look for:
   - Logic that works in tests (mocked data, happy path) but fails with real data
   - Race conditions that only appear under concurrent load
   - Off-by-one errors in pagination, array access, date ranges
   - Floating point used for currency or percentages
   - Timezone assumptions (`new Date()` without timezone spec)
   - Missing `await` on async calls (code continues before data arrives)
   - Silent failures: `try/catch` that swallows errors with no logging

   **Lens 2 — Missing error handling**
   Every external call must have an error path:
   - Network requests: what happens on timeout? on 500? on malformed response?
   - DB queries: what happens on connection failure? on constraint violation?
   - File system: what happens if the file doesn't exist?
   - User input parsing: what happens on unexpected shape?
   - Third-party APIs: are rate limits handled?

   **Lens 3 — Security holes**
   - User-controlled input going into SQL/shell/eval without sanitization
   - Missing authorization checks (is this user allowed to access this resource?)
   - Secrets or API keys in code or logs
   - CORS headers that are too permissive (`*`)
   - Insecure direct object references (IDOR) — IDs passed in URL/body without ownership check
   - Mass assignment: is every field that gets updated explicitly allowlisted?

   **Lens 4 — Missing tests**
   For every new function, ask:
   - Is there a unit test for the happy path?
   - Is there a test for the primary failure mode?
   - Are new API routes integration-tested?
   - If the PR fixes a bug: is there a regression test?
   Absence of tests is a comment, not a blocker — but it must be called out.

   **Lens 5 — Dead code and quality**
   - Commented-out code (unless explicitly marked as "keep for reference: ...")
   - Unused imports, variables, or functions
   - Functions longer than ~50 lines (probably needs to be split)
   - Inconsistent naming (camelCase vs snake_case in the same file)
   - `console.log` or debug statements left in
   - TODO comments without a ticket number

4. **Write inline review comments** in this format:
   ```
   **[FILE: src/app/api/users/route.ts, LINE: 42]**
   🐛 BUG: This `await` is missing. The function returns before `result` is populated.
   ```
   Prefix convention:
   - `🐛 BUG:` — will break in production, must fix before merge
   - `🔒 SECURITY:` — security concern, must fix before merge
   - `⚠️ WARNING:` — likely problem, should fix
   - `🧹 CLEANUP:` — code quality, nice to fix
   - `❓ QUESTION:` — unclear intent, needs clarification
   - `✅ NICE:` — worth calling out what's done well

5. **Render a summary**
   ```
   ## Review Summary
   - **Blockers (must fix):** N
   - **Warnings (should fix):** N
   - **Cleanup (optional):** N
   - **Verdict:** APPROVE / REQUEST CHANGES / BLOCK
   ```
   - APPROVE: zero blockers
   - REQUEST CHANGES: 1+ warnings, zero blockers
   - BLOCK: 1+ blockers (bug or security)

6. **Optionally write to file** if the review is long:
   ```
   write_file("docs/review-<branch-name>.md", <full review>)
   ```

## Sir's Defaults

- Default branch comparison: `main...HEAD`
- If no branch name is given: `terminal("git branch --show-current")` to get it
- Always check auth-related files even if not directly changed (a refactor may have moved auth middleware)
- Always check `.env.example` — if secrets were added to code but not listed here, flag immediately

## Pitfalls

- **Don't just review the diff.** The diff shows lines; the bug lives in the context. Read full files.
- **CI green ≠ correct.** CI tests the happy path. Your job is to find the unhappy paths CI doesn't know about.
- **Don't skip Lens 3 because it's "internal tooling."** Internal tools get breached. The blast radius is often larger.
- **"Missing tests" is not optional feedback.** It's a risk statement. Call it out every time, even for "small" changes.
- **Don't approve if you don't understand something.** An `❓ QUESTION:` is a valid review action. Understanding is required.

## Verification

- All 5 lenses have been explicitly applied (not assumed)
- Every blocker has a file + line number
- Summary section with verdict is present
- No APPROVE verdict if any `🐛 BUG:` or `🔒 SECURITY:` comments exist
