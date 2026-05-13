---
name: qa
layer: L2
source: gstack
description: Real browser QA — navigate localhost:3000, click through core flows, find bugs, fix them, re-verify
version: 1.0.0
tags: [qa, browser, testing, verification, e2e]
---

## When to Use

- After a feature is implemented and unit tests pass
- Before creating a PR / running `ship`
- When a bug report comes in and you need to reproduce it
- After a fix to verify the fix works and nothing regressed
- Any time the phrase "works on my machine" has been uttered

## Prerequisites

- Dev server is running on `localhost:3000` (or known port)
- The feature has a defined user flow (from plan.md, ticket, or description)
- `browser_navigate` and `browser_vision` tools are available
- If the app requires auth: test credentials are known

## Procedure

1. **Confirm the server is running**
   ```
   terminal("curl -s -o /dev/null -w '%{http_code}' http://localhost:3000")
   ```
   If not 200: try `localhost:3001`, `localhost:8080`. If none respond, start the server:
   ```
   terminal("npm run dev", background=true)
   // wait for "ready" signal in output
   ```

2. **Read the feature spec** to know what flows to test
   ```
   read_file("plan.md")           // what was built
   read_file("docs/plan.md")      // fallback
   ```
   Extract: list of user-facing flows. If no spec: derive flows from recent git changes:
   ```
   terminal("git diff main...HEAD --stat")
   ```

3. **Navigate to the app**
   ```
   browser_navigate("http://localhost:3000")
   browser_vision()                   // screenshot — confirm page loaded correctly
   ```

4. **Execute core flows** — for each flow:

   **Pattern for every flow:**
   ```
   // a) Navigate to the starting point
   browser_navigate("http://localhost:3000/<path>")
   browser_vision()                   // confirm starting state
   
   // b) Interact (click, type, submit)
   browser_click(<selector or coordinates>)
   browser_type(<input>, <value>)
   browser_vision()                   // confirm UI responded correctly
   
   // c) Submit / trigger action
   browser_click(<submit button>)
   browser_vision()                   // confirm result
   ```

   **Minimum flows to always test:**
   - **Happy path**: complete the primary action successfully end-to-end
   - **Empty state**: what does the UI show with no data?
   - **Error state**: submit a form with invalid/missing data — does the error message appear and make sense?
   - **Auth gate**: try to access a protected route while logged out — does it redirect to login?
   - **Loading state**: on slow networks or slow APIs, is there a loading indicator?

5. **For each bug found:**
   a. Screenshot the bug:
      ```
      browser_vision()   // capture the broken state
      ```
   b. Note: file/component likely responsible, what was expected, what actually happened
   c. Find the relevant source file:
      ```
      search_files("<component name>", path="src")
      read_file("<file path>")
      ```
   d. Fix the bug:
      ```
      // Use patch or write_file to apply fix
      ```
   e. Re-navigate and verify the fix:
      ```
      browser_navigate("http://localhost:3000/<path>")
      browser_vision()   // confirm fix resolved the issue
      ```

6. **Run a regression check** after any fix — re-test all flows, not just the fixed one:
   ```
   browser_navigate("http://localhost:3000")
   browser_vision()
   // re-execute each flow from step 4
   ```

7. **Write the QA report**
   ```
   write_file("docs/qa-report.md", <report>)
   ```
   Format:
   ```markdown
   # QA Report — {feature name} — {DATE}

   ## Verdict: PASS / FAIL / PASS WITH KNOWN ISSUES

   ## Environment
   - URL: localhost:3000
   - Branch: <git branch>
   - Server: <Next.js dev / production build>

   ## Flows Tested
   - [x] Happy path: <description> — PASS
   - [x] Empty state — PASS
   - [x] Error handling — PASS
   - [x] Auth gate — PASS
   - [ ] Loading state — SKIPPED (reason)

   ## Bugs Found
   ### Bug 1 — <title>
   **Severity:** Critical / High / Medium / Low
   **Reproduction:** Step 1 → Step 2 → ...
   **Expected:** ...
   **Actual:** ...
   **Status:** Fixed / Open

   ## Bugs Fixed
   - Bug 1: Fixed in src/components/... by ...

   ## Remaining Issues
   - ...

   ## Screenshots
   (reference browser_vision outputs taken during session)
   ```

## Sir's Defaults

- Default port: `3000`. Fallback order: `3001`, `8080`, `5000`
- Always test empty state — it's the most consistently broken thing
- Test on the actual running dev server, not mocked API responses
- If auth is required: use `test@example.com` / `password123` or whatever's in `.env.example`
- After fixing a bug: always re-run the *full* flow, not just the specific step that failed

## Pitfalls

- **Don't QA against mocked data only.** The real server, real DB, real API — that's where bugs live.
- **Don't skip empty state.** Almost every app ships with broken empty states because devs always have sample data locally.
- **Screenshot before AND after each bug fix.** "It worked after the fix" without a screenshot is not evidence.
- **One fix can break another flow.** Always regression-test after fixing. Don't assume a fix is isolated.
- **"Works in dev" ≠ "works in production."** If the app has a staging environment, run QA there too before ship.
- **Don't mark a bug as "Low" just because it's in a corner case.** Estimate frequency × severity, not just severity.

## Verification

- Every core flow was exercised (not just the happy path)
- Every bug found has: reproduction steps, expected, actual, severity, and status
- Every bug fixed has been re-verified with a browser_vision screenshot
- `docs/qa-report.md` exists with a clear PASS / FAIL verdict
- No open Critical or High bugs if verdict is PASS
