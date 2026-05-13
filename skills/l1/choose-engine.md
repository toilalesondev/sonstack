---
name: choose-engine
layer: L1
source: oh-my-hermes
description: Routes a coding task to the correct execution engine (claude-code, codex, or Hermes direct) based on complexity and scope
version: 1.0.0
tags: [routing, engine, claude-code, codex, planning]
---

## When to Use

Run this skill **before** starting any coding task to select the right tool. Use it when:
- Sir says "build this", "fix this bug", "add this feature"
- After `product-brief` is written and it's time to write code
- Any time the coding approach is unclear

Do not run this skill for purely conversational or research tasks — it's for **code execution routing only**.

## Prerequisites

- The task is clearly defined (run `clarify-requirements` if not)
- PRODUCT_BRIEF.md or a clear task description is available
- All three engines are available: `claude-code`, `codex`, `hermes_direct`
- Memory is readable for project context

## Procedure

1. **Load task context** — read from memory or accept from Sir's message:

   ```
   tool: memory_read
   key: "project/{project_slug}/context"
   ```

   Also inspect the current task description for keywords.

2. **Score the task against the decision matrix:**

   Evaluate each criterion and assign a score:

   | Criterion | Score +1 if true |
   |---|---|
   | Touches **3 or more files** | +1 |
   | Requires **creating a new feature** (not patching) | +1 |
   | Involves **database schema changes** | +1 |
   | Requires **running shell commands** (install, migrate, build) | +1 |
   | Needs **web browsing** or live URL fetching | +1 |
   | Task description is **>50 words** or multi-part | +1 |
   | Involves **auth, payments, or security** | +1 |
   | Touches **1 file only** | -2 |
   | Task is a **typo/rename/small refactor** | -3 |
   | Task is **purely a question** (no file writes needed) | -5 |

3. **Apply routing rules:**

   ```
   if total_score >= 3:
     engine = "claude-code"
     reason = "Complex multi-file work requiring full repo context"
   
   elif total_score >= 1:
     engine = "codex"
     reason = "Single-file or targeted fix, no broad context needed"
   
   else:
     engine = "hermes_direct"
     reason = "Simple task / question / no file writes required"
   ```

   **Override rules** (take precedence over score):
   - If task explicitly says "browse", "fetch URL", "research" → always `claude-code` (has browser tool)
   - If task is "explain this code" with no edits → always `hermes_direct`
   - If task touches `supabase/migrations/` → always `claude-code` (migration safety)
   - If task touches `vercel.json` or deployment config → always `claude-code`

4. **Log routing decision to memory:**

   ```
   tool: memory_write
   key: "project/{project_slug}/last_engine_decision"
   value: {
     task_summary: "{one-line task description}",
     engine: "{chosen engine}",
     score: {total_score},
     reason: "{reason string}",
     decided_at: "<ISO timestamp>"
   }
   ```

5. **Notify Sir via Telegram:**

   ```
   tool: send_telegram_message
   text: "🔧 Engine selected: **{ENGINE}**\n\nTask: {task summary}\nReason: {reason}\n\nStarting now — I'll update you when done."
   ```

6. **Execute with the chosen engine:**

   **If `claude-code`:**
   ```
   tool: spawn_claude_code
   workdir: "{project_root}"
   prompt: "{full task description with context from PRODUCT_BRIEF.md}"
   ```

   **If `codex`:**
   ```
   tool: spawn_codex
   workdir: "{project_root}"
   file_hint: "{target file path if known}"
   prompt: "{focused single-file task description}"
   ```

   **If `hermes_direct`:**
   ```
   # Handle inline — no spawning needed
   # Respond directly to Sir's question or perform simple task
   ```

7. **On engine completion**, update memory with outcome:

   ```
   tool: memory_write
   key: "project/{project_slug}/last_task_outcome"
   value: {
     engine: "{engine used}",
     status: "completed" | "failed" | "needs_review",
     output_summary: "{brief summary of what changed}",
     files_modified: ["{file1}", "{file2}"],
     completed_at: "<ISO timestamp>"
   }
   ```

## Sir's Defaults

- **Default engine for new projects:** `claude-code` (full scaffold = multi-file by definition)
- **Default engine for "quick fix" requests:** `codex`
- **Never use Slack** for engine routing notifications — Telegram only
- **VPS workdir pattern:** `/home/karpathy/projects/{project_slug}`
- **macOS local workdir pattern:** `~/projects/{project_slug}`
- **Supabase migrations** always route to `claude-code` regardless of score (project: `bhssclapikzlpyzolzjy`)
- **GitHub push** after claude-code tasks: always use `toilalesondev` account with SSH auth

**Engine capability summary:**

| Capability | claude-code | codex | hermes_direct |
|---|---|---|---|
| Multi-file edits | ✅ | ❌ | ❌ |
| Web browsing | ✅ | ❌ | ❌ |
| Shell commands | ✅ | ✅ | ❌ |
| Single-file fix | ✅ | ✅ | ❌ |
| Answer questions | ✅ | ❌ | ✅ |
| Speed | Slow | Medium | Fast |
| Cost | High | Medium | Low |

## Pitfalls

- **Don't always default to `claude-code`** — it's slower and more expensive. Use `codex` for small fixes and `hermes_direct` for questions.
- **Don't ignore the override rules** — especially for Supabase migrations. A wrong migration can corrupt the production DB (`bhssclapikzlpyzolzjy`).
- **Don't start the engine before notifying Sir** — always send the Telegram message first (step 5) so Sir knows work has begun.
- **Score is a guide, not a law** — if Sir explicitly says "just use claude-code" or "quick codex fix", respect that override.
- **If project_root is unknown**, check memory first. If not in memory, ask Sir before running any engine:
  ```
  tool: send_telegram_message
  text: "⚠️ Project root not found in memory. What's the absolute path to {project_slug}?"
  ```

## Verification

After routing decision (step 3), verify before executing:

1. Engine tool is available (not rate-limited or down)
2. Project root path exists and is accessible
3. For `claude-code`: PRODUCT_BRIEF.md or equivalent context is in the prompt
4. For Supabase tasks: confirm project ID `bhssclapikzlpyzolzjy` is in env before running migrations

After engine completes (step 7), verify:
- Memory key `last_task_outcome` has `status: "completed"`
- Modified files are committed or staged (check with `git status` in workdir)
- No new lint errors introduced
