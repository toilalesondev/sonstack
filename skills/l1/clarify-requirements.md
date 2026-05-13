---
name: clarify-requirements
layer: L1
source: oh-my-hermes
description: 7-question intake interview to capture project requirements and save answers to Hermes memory
version: 1.0.0
tags: [intake, requirements, planning, memory]
---

## When to Use

Use this skill at the **very start** of any new project or feature when Sir says something like:
- "I want to build…"
- "Let's start a new app…"
- "Help me plan…"
- Any vague idea that needs scope before touching code

Do **not** skip this skill. A 3-minute interview saves hours of rework.

## Prerequisites

- Hermes memory is writable (`memory` tool available)
- Telegram notification channel is configured (Sir's chat ID in env)
- Conversation context with Sir is active

## Procedure

1. **Open the intake session** — announce the 7-question intake directly in chat:

   > "Starting requirements intake for [project name]. I'll ask 7 quick questions — answer each in your own words, EN or VI is fine. 🚀"

2. **Ask Q1 — Problem statement** — write directly in chat:

   > "Q1/7 — **Problem:** What problem does this solve? (one sentence — who feels the pain and what are they doing without your product?)"

   Wait for reply. Note the answer as `intake.problem`.

3. **Ask Q2 — Target users** — write directly in chat:

   > "Q2/7 — **Users:** Who is the first user? (a specific person or a concrete persona — their role, habits, tech comfort level)"

   Wait for reply. Note the answer as `intake.users`.

4. **Ask Q3 — Status quo** — write directly in chat:

   > "Q3/7 — **Status quo:** What does the current solution look like? How is this problem being solved today?"

   Wait for reply. Note the answer as `intake.status_quo`.

5. **Ask Q4 — Core action** — write directly in chat:

   > "Q4/7 — **Core action:** What's the single most important thing a user does in the app? (the one action that makes the whole product worth using)"

   Wait for reply. Note the answer as `intake.core_action`.

6. **Ask Q5 — Stack preference** — write directly in chat:

   > "Q5/7 — **Stack:** Any strong preferences? (framework, language, DB, hosting) Or just say \"no preference\" and I'll use Sonstack defaults: Next.js + Supabase + Vercel."

   Wait for reply. Note the answer as `intake.stack`.

7. **Ask Q6 — Monetization** — write directly in chat:

   > "Q6/7 — **Monetization:** How will this make money — or won't it? (free / paid / not thinking about it yet)"

   Wait for reply. Note the answer as `intake.monetization`.

8. **Ask Q7 — Success metric** — write directly in chat:

   > "Q7/7 — **Success:** What does success look like in 30 days? (one measurable outcome — e.g. '100 sign-ups', 'team uses it daily', 'I can demo it on Friday')"

   Wait for reply. Note the answer as `intake.success_30d`.

9. **Save all answers to memory:**

   ```
   memory(
     action="add",
     target="memory",
     content="Project {project_slug} intake: Q1_problem={intake.problem} | Q2_users={intake.users} | Q3_status_quo={intake.status_quo} | Q4_core_action={intake.core_action} | Q5_stack={intake.stack} | Q6_monetization={intake.monetization} | Q7_success_30d={intake.success_30d} | captured_at={ISO timestamp}"
   )
   ```

10. **Confirm to Sir in chat:**

    > "✅ Intake complete! Answers saved to memory. Next step: run `product-brief` skill to generate PRODUCT_BRIEF.md when ready."

## Sir's Defaults

| Setting | Value |
|---|---|
| Notification channel | Telegram (never Slack) |
| Default stack (if Q5 is blank) | Next.js 14 + Supabase (`bhssclapikzlpyzolzjy`) + Vercel |
| DB project | `bhssclapikzlpyzolzjy` |
| Runtime | VPS (`karpathy` user) + macOS local |
| GitHub account | `toilalesondev`, SSH auth |
| Language | Accept EN or VI answers; store as-is |
| PAT location | 1Password → Hermes Shared vault |

If Sir answers Q5 with "defaults" or blank, auto-fill:
```
stack: "Next.js 14 App Router, TypeScript, Supabase (bhssclapikzlpyzolzjy), Vercel, Tailwind CSS, shadcn/ui"
```

## Pitfalls

- **Don't batch all 7 questions in one message** — Sir will give shallow answers. One question at a time.
- **Don't proceed to `product-brief` automatically** — Sir may want to adjust answers first.
- **Don't assume the project slug** — derive it from the project name (lowercase, hyphenated). Ask if ambiguous.
- **If Sir answers in Vietnamese**, store the answer as-is; do not auto-translate. The `product-brief` skill will normalize.
- **If memory write fails**, fall back to writing a local file `intake_raw.json` in the project root and warn Sir in chat.
- **Don't skip Q6 (monetization)** — it changes scope dramatically (auth, billing, usage limits).

## Verification

After step 9, confirm in chat that the memory entry was saved, e.g.:

> "Intake saved ✅ — Project {project_slug} intake has 7 answers in memory. Ready for product-brief."

If the `memory` call returned an error, tell Sir and offer to retry or save to a local file instead.
