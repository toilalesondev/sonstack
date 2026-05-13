---
name: ship
layer: L2
source: gstack
description: Gated ship sequence — tests → typecheck → lint → commit → push → PR. Blocks on any failure.
version: 1.0.0
tags: [git, deploy, ship, ci, pr, commit]
---

## When to Use

- Feature is implemented, reviewed, and QA has passed
- Ready to create a PR for human review or auto-merge
- Any time code needs to go from local branch to remote
- Do NOT use to "just push a quick fix" — the gates exist for a reason

## Prerequisites

- Git repo initialized, remote configured
- `gh` CLI installed and authenticated (`gh auth status`)
- `npm` scripts available: `test`, `typecheck` (or `tsc`), `lint`
- On a feature branch (not `main` directly)
- QA has passed (or waived with explicit acknowledgment)

## Procedure

**These steps are sequential and gated. If any step fails, STOP. Do not proceed.**

### Step 1 — Run tests
```
terminal("npm test -- --run")          // Vitest
// or:
terminal("npm test -- --passWithNoTests")  // Jest
```
- **If tests fail**: print the failure output. DO NOT continue to step 2. Fix the failures or explicitly get confirmation from the user to proceed (and document the waiver).
- **If no tests exist**: warn *"No test suite found — proceeding, but this is a risk."* Log it.

### Step 2 — Run typecheck
```
terminal("npm run typecheck")
// fallback if no typecheck script:
terminal("npx tsc --noEmit")
```
- **If typecheck fails**: print the errors. DO NOT continue. TypeScript errors are bugs with extra steps.
- Zero errors required to proceed.

### Step 3 — Run lint
```
terminal("npm run lint")
```
- **If lint fails with errors**: stop. Fix the lint errors.
- **If lint produces only warnings**: proceed, but note the warnings in the commit message or PR description.
- If no lint script: `terminal("npx eslint src --ext .ts,.tsx")` as fallback.

### Step 4 — Stage changes
```
terminal("git status")              // review what's changed
terminal("git diff --stat")         // confirm scope
terminal("git add -A")              // stage all changes
terminal("git status")              // confirm staging
```
Review the staged files before committing. If anything looks unexpected (unrelated files, build artifacts, secrets), unstage and investigate:
```
terminal("git reset HEAD <suspicious-file>")
terminal("git diff HEAD -- <file>")   // inspect the specific file
```

### Step 5 — Write the commit message
Follow Conventional Commits format:
```
<type>(<scope>): <short description>

[optional body — what changed and why, not how]

[optional footer — breaking changes, issue refs]
```
Types:
- `feat:` — new feature
- `fix:` — bug fix
- `refactor:` — code change that's not a fix or feature
- `test:` — adding or updating tests
- `chore:` — build, deps, config
- `docs:` — documentation only
- `perf:` — performance improvement
- `security:` — security fix (use this, don't bury security fixes in `fix:`)

Rules:
- Subject line ≤ 72 characters
- Subject line is imperative mood: "add user auth" not "added user auth"
- If this fixes an issue: add `Closes #<issue-number>` in footer
- If this is a breaking change: add `BREAKING CHANGE:` in footer

Commit:
```
terminal('git commit -m "<type>(<scope>): <description>"')
// for multi-line commit:
terminal('git commit -m "<type>: <subject>" -m "<body>"')
```

### Step 6 — Push to remote
```
terminal("git push origin HEAD")
```
- If the branch doesn't exist on remote yet, it will be created automatically.
- If push is rejected (non-fast-forward), do NOT force push without understanding why:
  ```
  terminal("git log origin/main..HEAD --oneline")  // see what we have
  terminal("git fetch origin")
  terminal("git rebase origin/main")               // rebase, then retry push
  ```
- Never `git push --force` on a branch someone else might be reviewing.

### Step 7 — Create the PR
```
terminal("gh pr create --title '<type>: <description>' --body '<pr body>' --base main")
```
PR body template:
```markdown
## What
[1-3 sentences: what does this change do?]

## Why
[1-2 sentences: why is this change needed?]

## How
[optional: notable implementation decisions]

## Testing
- [ ] Unit tests pass
- [ ] Typecheck passes
- [ ] Lint passes
- [ ] Manual QA: <what was tested>

## Screenshots
[if UI changes: before/after screenshots]
```
After creating:
```
terminal("gh pr view --web")   // open in browser to verify it looks right
```

### Step 8 — Confirm
Print the PR URL to the user. Ship is complete.

## Sir's Defaults

- Default base branch: `main` (not `master`)
- Default test command: `npm test -- --run` (Vitest) or `npm test` (Jest)
- Default typecheck: `npm run typecheck` with fallback to `npx tsc --noEmit`
- Always run `git status` before `git add -A` — no surprises
- PR title should match the commit subject line
- Auto-add `Closes #<n>` if the branch name contains an issue number (e.g., `feat/123-add-auth`)

## Pitfalls

- **Never skip the tests step.** "Tests were passing earlier" is not the same as tests passing now.
- **Never force push without understanding why the push was rejected.** A rejected push often means someone else has commits on that branch.
- **Conventional commits matter.** They power changelogs, semantic versioning, and PR search. Enforce the format.
- **Don't commit secrets.** Before `git add -A`, always scan for `.env` files, API keys in new files:
  ```
  terminal("git diff --cached | grep -i 'secret\\|key\\|password\\|token'")
  ```
  If any match: unstage and investigate immediately.
- **`gh pr create` requires `gh` auth.** If not authed: `terminal("gh auth login")` first.
- **Don't create the PR if tests failed and were waived.** If steps 1–3 were skipped for any reason, the PR description must explicitly say so and why.

## Verification

- `terminal("npm test -- --run")` exit code 0
- `terminal("npm run typecheck")` exit code 0
- `terminal("npm run lint")` exit code 0
- `terminal("git log origin/main..HEAD --oneline")` shows exactly the expected commits
- PR exists on GitHub: `terminal("gh pr view")` returns the PR details
- PR URL has been shared with the user
