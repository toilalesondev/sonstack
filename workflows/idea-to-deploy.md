---
name: idea-to-deploy
layer: workflow
description: Master 14-gate orchestrator. Sir texts an idea — Karpathy ships it. No skipping gates.
version: 1.0.0
tags: [workflow, orchestrator, 14-gate, end-to-end]
---

# idea-to-deploy — Master Workflow

Sir texts an idea. Karpathy ships it. 14 gates, no exceptions.

## The Hard Rule

**No gate skips. Ever.** If Sir asks to skip — say no and explain why.
Gate status tracked in `docs/gate-status.md` per project.

## Gate Execution

### G01 — Requirements (`skills/l1/clarify-requirements.md`)
- Load skill, run 7-question intake
- Save answers to Hermes memory
- **Sir does:** answers 7 questions
- **Output:** memory updated, requirements clear
- **Gate passes when:** all 7 answered, saved to memory

### G02 — Product Brief (`skills/l1/product-brief.md`)
- Load skill, read from memory
- Write `PRODUCT_BRIEF.md`
- **Output:** PRODUCT_BRIEF.md committed
- **Gate passes when:** file exists, no open blockers

### G03 — Stress-Test (`skills/l2/office-hours.md`)
- Load skill, run YC 6-question stress-test
- Write `docs/office-hours-stress-test.md`
- **Output:** stress-test doc, verdict (SHIP / PIVOT / KILL)
- **Gate passes when:** verdict is SHIP, or Sir overrides with reason

### G04 — Plan
- Karpathy writes `docs/plans/YYYY-MM-DD-<project>.md`
- Format: problem, solution, tech stack, tasks (day-by-day), risks
- **Gate passes when:** plan.md exists and committed

### G05 — CEO Review (`skills/l2/plan-ceo-review.md`)
- Load skill, read plan.md
- Score ambition 1-10, flag what's missing
- **Gate passes when:** ambition score ≥ 7, or plan revised

### G06 — Eng Review (`skills/l2/plan-eng-review.md`)
- Load skill, read plan.md
- Check data model, API design, edge cases, tests
- **Output:** `docs/arch-decisions.md`
- **Gate passes when:** no blocking architecture issues

### G07 — Design (UI projects only)
- **On VPS:** load `sketch` skill → 3 HTML variants → screenshot → Sir picks
- **On Mac:** load `design-shotgun` → DALL-E variants → Sir picks
- **Sir does:** picks design variant
- **Gate passes when:** approved design committed, DESIGN.md written

### G08 — Build (`skills/l1/choose-engine.md`)
- Load skill, score task complexity
- Route to: claude-code (complex) / codex (targeted) / hermes direct (simple)
- Build against approved plan + design
- **Gate passes when:** build compiles, core flows work

### G09 — Review (`skills/l2/review.md` + `skills/l2/cso.md` + `skills/l2/qa.md`)
- Run in sequence: code review → security scan → browser QA
- Fix all blockers before proceeding
- **Gate passes when:** no blocking issues in any of the three

### G10 — Ship (`skills/l2/ship.md` + `skills/l1/create-github-pr.md`)
- Run tests → typecheck → lint → commit → push → open PR
- **Gate passes when:** PR open on GitHub, CI green

### G11 — Merge Approval (`skills/l1/await-merge-approval.md`)
- Send PR link + summary to Sir on Telegram
- Block until Sir replies YES or NO
- **Sir does:** YES / NO
- **Gate passes when:** Sir replies YES

### G12 — Deploy (`skills/l1/deploy-to-vercel.md` + `skills/l1/connect-supabase.md`)
- Deploy to Vercel prod, push Supabase migrations
- Capture live URL, update memory
- **Gate passes when:** live URL returns 200

### G13 — Monitoring (`skills/l2/canary.md` + `skills/l1/health-check.md`)
- Run post-deploy canary watch (5 min)
- Verify all core endpoints healthy
- **Gate passes when:** no errors in canary window

### G14 — Report (`skills/l1/cto-status-report.md`)
- Write status report, send to Sir on Telegram
- Update docs (README, CHANGELOG)
- **Gate passes when:** Sir notified, docs committed

## Gate Status Template

Write to `docs/gate-status.md` at project start:

```
# Gate Status — <project>

| Gate | Name               | Status | Notes |
|------|--------------------|--------|-------|
| G01  | Requirements       | ❌     |       |
| G02  | Product Brief      | ❌     |       |
| G03  | Stress-Test        | ❌     |       |
| G04  | Plan               | ❌     |       |
| G05  | CEO Review         | ❌     |       |
| G06  | Eng Review         | ❌     |       |
| G07  | Design             | ❌     |       |
| G08  | Build              | ❌     |       |
| G09  | Review             | ❌     |       |
| G10  | Ship               | ❌     |       |
| G11  | Merge Approval     | ❌     |       |
| G12  | Deploy             | ❌     |       |
| G13  | Monitoring         | ❌     |       |
| G14  | Report             | ❌     |       |
```

Update status to ✅ as each gate passes.
