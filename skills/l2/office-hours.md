---
name: office-hours
layer: L2
source: gstack
description: YC-style office hours stress-test that forces brutal clarity on what you're building and why
version: 1.0.0
tags: [strategy, product, yc, stress-test, clarity]
---

## When to Use

- Before writing a single line of code on a new idea
- When the team is fuzzy on "what problem we're solving exactly"
- When a feature has grown scope-creep and you've lost the thread
- Quarterly product review to pressure-test existing direction
- Any time you catch yourself saying "it's hard to explain"

## Prerequisites

- A product idea, feature spec, or existing product description (can be rough)
- Access to `docs/` directory for output
- 30 minutes of uninterrupted focus — this is a thinking exercise, not a checkbox

## Procedure

1. **Read the product brief or spec**
   ```
   read_file("docs/plan.md")          // or wherever the brief lives
   read_file("README.md")             // fallback if no plan.md
   ```
   If neither exists, ask the user: *"Give me 2–3 sentences on what this does."*

2. **Run the 6 YC forcing questions** (answer each in 3–5 sentences, no hedging):

   **Q1 — What does this do?**
   Compress the product into one sentence a smart 12-year-old could repeat back. No jargon. No "platform." No "ecosystem." If you can't, the idea is not clear yet.

   **Q2 — Who desperately needs it?**
   Not "anyone who…" — name a *specific* person. What is their job title? What does their Tuesday afternoon look like? Why does the current situation cause them pain that feels personal, not just inconvenient?

   **Q3 — What does the status quo look like?**
   What are they doing *right now* without your product? Excel? Slack threads? Nothing? Hiring a person? Quantify the cost — time, money, or embarrassment. If the status quo is "nothing," that's a red flag (demand signal is weak).

   **Q4 — Narrowest possible wedge**
   What is the *single* workflow or moment where you can win undeniably, even if everything else is mediocre? This is your beachhead. It must be small enough that you can own it completely with one sprint.

   **Q5 — Personal observation**
   Why does *this team* have an unfair advantage here? Lived experience? Domain access? Proprietary data? Existing relationships? If the answer is "we don't," that's critical information.

   **Q6 — Feature or standalone?**
   Is this a product people will open a new tab for, or is it a button inside another product? Be honest. Features rarely become companies. If it's a feature, name the host product and consider a different distribution strategy.

3. **Score each answer** on a 1–5 scale:
   - 5 = razor sharp, no ambiguity
   - 3 = roughly right but needs tightening
   - 1 = hand-wavy, would not survive a YC partner
   
   Flag any answer scoring ≤ 2 as **BLOCKER**.

4. **Write the stress-test report**
   ```
   write_file("docs/office-hours-stress-test.md", <report>)
   ```
   Report format:
   ```markdown
   # Office Hours Stress-Test — {DATE}

   ## Verdict
   PASS / CONDITIONAL PASS / FAIL (and why in one sentence)

   ## Q1: What does this do?
   **Answer:** ...
   **Score:** X/5
   **Notes:** ...

   ## Q2: Who desperately needs it?
   ...

   ## Q3: Status quo
   ...

   ## Q4: Narrowest wedge
   ...

   ## Q5: Personal observation / unfair advantage
   ...

   ## Q6: Feature or standalone?
   ...

   ## Blockers (score ≤ 2)
   - [ ] ...

   ## Recommended next step
   One concrete action to take this week.
   ```

5. **Surface blockers to the user** inline — do not bury them in the document.

## Sir's Defaults

- Default output path: `docs/office-hours-stress-test.md`
- Default date: today's date via `terminal("date +%Y-%m-%d")`
- If plan.md has no clear "who" section, assume Q2 and Q5 will score ≤ 2 — pre-flag
- Always end with the "recommended next step" — actionability is the whole point

## Pitfalls

- **Don't be nice.** The value is in the brutal honest read, not the encouraging one. If an answer is weak, say so clearly.
- **Don't let Q1 be compound.** If the one-sentence answer has "and" in it, the idea is two ideas.
- **Q3 "nothing" is suspicious.** If users truly have no workaround, either the problem is fake or the market is too early.
- **Q6 feature trap.** Teams frequently mistake "would be a cool integration" for "standalone product." Push hard on this one.
- **Don't write the report until all 6 answers are resolved.** Partial stress-tests are worse than none — they create false confidence.

## Verification

- `docs/office-hours-stress-test.md` exists and has all 6 sections populated
- Verdict is explicitly stated (not implied)
- Any blocker (score ≤ 2) has a corresponding action item
- "Recommended next step" is a concrete, time-bound action — not a platitude
