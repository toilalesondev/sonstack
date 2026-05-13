---
name: plan-eng-review
layer: L2
source: gstack
description: Senior-engineer review of plan.md — locks architecture decisions and surfaces technical debt before it's written
version: 1.0.0
tags: [engineering, planning, architecture, review, technical]
---

## When to Use

- After CEO review passes (strategy is locked), before any code is written
- When the plan describes a data model, API, or system interaction — even loosely
- Before a sprint starts to ensure engineers aren't making up architecture on the fly
- When technical debt from a previous sprint is being carried forward

## Prerequisites

- `plan.md` (or `docs/plan.md`) exists and has passed CEO review, or CEO review is waived
- The plan includes at minimum: what needs to be built, who uses it, and rough data/interaction model
- Read access to existing codebase if this is not a greenfield project

## Procedure

1. **Read the plan**
   ```
   read_file("plan.md")
   // fallback:
   read_file("docs/plan.md")
   ```

2. **Read existing architecture context** (if not greenfield)
   ```
   search_files("schema", path=".")          // find schema files
   search_files("*.prisma", path=".")        // Prisma schema
   search_files("types.ts", path="src")      // type definitions
   read_file("<schema_file>")                // read each found
   ```
   Also check:
   ```
   read_file("src/lib/db.ts")               // or equivalent DB layer
   read_file("src/app/api/**")              // existing API routes
   ```

3. **Run the 5 engineering review questions:**

   **Q1 — Is the data model correct?**
   - Does the plan imply a data shape? Map it out explicitly (even in pseudocode).
   - Are the relationships correct? (1:1, 1:N, N:M — name them)
   - Are there nullable fields that shouldn't be? Required fields that will be optional in practice?
   - Does this model make queries expensive at scale? (e.g., N+1 queries hiding in the design)
   - Does it conflict with existing schema? Flag any migration required.

   **Q2 — Is the API design clean?**
   - Are the endpoints RESTful or RPC-style? Is that consistent with the existing API?
   - Are there endpoints that will need to be split later because they do too much?
   - Are there missing endpoints (e.g., plan describes a UI flow but no API backs it)?
   - Authentication: is every endpoint explicitly auth-gated or explicitly public? No ambiguity.
   - Response shapes: consistent error format? Consistent pagination?

   **Q3 — Are edge cases covered?**
   - What happens when the user has no data yet? (empty state)
   - What happens when a request is made twice? (idempotency)
   - What happens under load? (rate limiting planned?)
   - What happens if a downstream service is down? (graceful degradation?)
   - Concurrent writes: is there a race condition in the proposed flow?

   **Q4 — Are tests planned?**
   - Does the plan mention test coverage at all? If not, flag it.
   - Unit tests: which pure functions need them?
   - Integration tests: which API routes touch the DB and need end-to-end coverage?
   - E2E tests: which user flows are critical enough to warrant browser tests?
   - Is there a testing strategy or just "we'll write tests"?

   **Q5 — What is the deployment strategy?**
   - Is there a DB migration? Is it backward-compatible (can old code read the new schema)?
   - Feature flag needed? Or is this a clean cut-over?
   - Any environment variables or secrets needed that aren't provisioned?
   - Rollback plan: if this ships and breaks, how do we revert?
   - Performance: any new indexes needed? Any slow queries introduced?

4. **Flag architecture smells** — any of these auto-flag as BLOCKER:
   - Schema changes without a migration plan
   - API endpoints with no auth mentioned
   - No error handling described for external service calls
   - "We'll figure out the data model during implementation"
   - N+1 queries implied by the design
   - Secrets hardcoded or unmanaged

5. **Output locked architecture decisions**
   ```
   write_file("docs/plan-eng-review.md", <review>)
   ```
   Format:
   ```markdown
   # Eng Review — {plan title} — {DATE}

   ## Status: APPROVED / APPROVED WITH CONDITIONS / BLOCKED

   ## Data Model
   ```
   // Explicit schema in pseudocode or actual Prisma/SQL
   ```

   ## API Contracts
   | Method | Path | Auth | Request | Response |
   |--------|------|------|---------|----------|
   | ...    | ...  | ...  | ...     | ...      |

   ## Edge Cases Addressed
   - [ ] Empty state: ...
   - [ ] Idempotency: ...
   - [ ] Failure mode: ...

   ## Test Plan
   - Unit: ...
   - Integration: ...
   - E2E: ...

   ## Deployment Checklist
   - [ ] Migration: ...
   - [ ] Feature flag: ...
   - [ ] Env vars: ...
   - [ ] Rollback: ...

   ## Architecture Smells / Blockers
   - ...

   ## Locked Decisions
   These are final. Do not re-litigate during implementation.
   1. ...
   2. ...
   ```

6. **If status is BLOCKED**, explicitly state: *"Engineering must not start until blocked items are resolved."* List each blocker with an owner and expected resolution.

## Sir's Defaults

- Default output: `docs/plan-eng-review.md`
- Assume Next.js + Prisma + TypeScript stack unless context says otherwise
- If no test plan exists in the plan: auto-add a test plan section with minimum required coverage
- "Locked Decisions" section is permanent — once written, it takes a new eng-review to change them

## Pitfalls

- **Don't skip the data model even if the plan doesn't mention it.** Every feature has a data model. If the plan doesn't, you write it.
- **"We'll handle edge cases later" is not an answer.** Later never comes. Name the edge cases and assign handling now.
- **Migration safety is critical.** A backward-incompatible migration on a live system is an outage. Flag every migration.
- **Auth is not optional.** If an endpoint doesn't explicitly say "authenticated" or "public," it's a security hole waiting to happen.
- **Don't approve a plan with no test strategy.** "We'll write tests" is not a strategy. At minimum: name which flows get E2E coverage.

## Verification

- `docs/plan-eng-review.md` exists with all 5 sections populated
- Data model is written out explicitly (not described in prose)
- Every API endpoint has an auth designation
- Deployment checklist is complete with no "TBD" items
- "Locked Decisions" section is non-empty
- Any BLOCKED status has concrete resolution criteria
