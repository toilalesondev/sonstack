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

- Hermes memory is writable (`memory_write` tool available)
- Telegram notification channel is configured (Sir's chat ID in env)
- Conversation context with Sir is active

## Procedure

1. **Open the intake session** — send Sir a Telegram message (or reply in-thread) announcing the 7-question intake:

   ```
   tool: send_telegram_message
   text: "Starting requirements intake for [project name]. I'll ask 7 quick questions — answer each in your own words, EN or VI is fine. 🚀"
   ```

2. **Ask Q1 — Problem statement:**

   ```
   tool: send_telegram_message
   text: "Q1/7 — Problem: What exact problem does this solve? Who feels the pain today and how are they solving it without your product?"
   ```

   Wait for reply. Log raw answer as `intake.problem`.

3. **Ask Q2 — Target users:**

   ```
   tool: send_telegram_message
   text: "Q2/7 — Users: Who are the primary users? Describe one real person (persona) — their job, habits, tech comfort level."
   ```

   Wait for reply. Log raw answer as `intake.users`.

4. **Ask Q3 — Data & integrations:**

   ```
   tool: send_telegram_message
   text: "Q3/7 — Data: What data does the app store or read? Any third-party APIs, OAuth providers, or file uploads involved?"
   ```

   Wait for reply. Log raw answer as `intake.data`.

5. **Ask Q4 — Stack preferences:**

   ```
   tool: send_telegram_message
   text: "Q4/7 — Stack: Any strong preferences? (framework, language, DB, hosting) Or shall I use the Sonstack defaults: Next.js + Supabase + Vercel?"
   ```

   Wait for reply. Log raw answer as `intake.stack`.

6. **Ask Q5 — Monetization:**

   ```
   tool: send_telegram_message
   text: "Q5/7 — Money: How will this make money (or not)? Free tool, SaaS subscription, one-time purchase, internal only?"
   ```

   Wait for reply. Log raw answer as `intake.monetization`.

7. **Ask Q6 — Timeline:**

   ```
   tool: send_telegram_message
   text: "Q6/7 — Timeline: What's the deadline or milestone? MVP in a weekend? Ship in 2 weeks? No rush?"
   ```

   Wait for reply. Log raw answer as `intake.timeline`.

8. **Ask Q7 — Success metric:**

   ```
   tool: send_telegram_message
   text: "Q7/7 — Success: How do we know this worked? One measurable outcome (e.g., '100 sign-ups', 'team uses it daily', 'I can demo it on Friday')."
   ```

   Wait for reply. Log raw answer as `intake.success_metric`.

9. **Save all answers to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/intake"
   value: {
     problem: <Q1 answer>,
     users: <Q2 answer>,
     data: <Q3 answer>,
     stack: <Q4 answer>,
     monetization: <Q5 answer>,
     timeline: <Q6 answer>,
     success_metric: <Q7 answer>,
     captured_at: <ISO timestamp>
   }
   ```

10. **Confirm to Sir via Telegram:**

    ```
    tool: send_telegram_message
    text: "✅ Intake complete! Answers saved. Next step: I'll generate the PRODUCT_BRIEF.md — run `product-brief` skill when ready."
    ```

## Sir's Defaults

| Setting | Value |
|---|---|
| Notification channel | Telegram (never Slack) |
| Default stack (if Q4 is blank) | Next.js 14 + Supabase (`bhssclapikzlpyzolzjy`) + Vercel |
| DB project | `bhssclapikzlpyzolzjy` |
| Runtime | VPS (`karpathy` user) + macOS local |
| GitHub account | `toilalesondev`, SSH auth |
| Language | Accept EN or VI answers; store as-is |
| PAT location | 1Password → Hermes Shared vault |

If Sir answers Q4 with "defaults" or blank, auto-fill:
```
stack: "Next.js 14 App Router, TypeScript, Supabase (bhssclapikzlpyzolzjy), Vercel, Tailwind CSS, shadcn/ui"
```

## Pitfalls

- **Don't batch all 7 questions in one message** — Sir will give shallow answers. One question at a time.
- **Don't proceed to `product-brief` automatically** — Sir may want to adjust answers first.
- **Don't assume the project slug** — derive it from the project name (lowercase, hyphenated). Ask if ambiguous.
- **If Sir answers in Vietnamese**, store the answer as-is; do not auto-translate. The `product-brief` skill will normalize.
- **If memory_write fails**, fall back to writing a local file `intake_raw.json` in the project root and warn Sir.
- **Don't skip Q5 (monetization)** — it changes scope dramatically (auth, billing, usage limits).

## Verification

After step 9, verify:

```
tool: memory_read
key: "project/{project_slug}/intake"
```

Expected: JSON object with all 7 keys populated and `captured_at` timestamp present.

Send Sir:
```
tool: send_telegram_message
text: "Intake saved ✅ — {project_slug}/intake has 7 answers. Ready for product-brief."
```
