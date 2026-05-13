---
name: send-notification
layer: L1
source: oh-my-hermes
description: Send a status notification to Sir on Telegram.
version: 1.1.0
tags: [telegram, notification, status, alert]
---

## When to Use
After deployment completes. When health check fails. When Sir needs a status update.
Any important event that Sir should know about.

## Prerequisites
- Hermes Telegram integration configured
- Sir's chat ID in config

## Procedure

1. **Identify the event type** and select the matching message template below.

2. **Compose the message** using the exact template — fill in placeholders:

   **Deploy success:**
   ```
   ✅ <project> deployed — <URL>
   ```

   **Health check failure:**
   ```
   ❌ Health check failed — <URL> — <error>
   ```
   Include what specifically failed (HTTP code, visual error, JS exception).

   **PR ready for review:**
   ```
   🔀 PR ready for review — <PR_URL> — <summary>
   ```
   Summary: 1 sentence max — what the PR does.

   **Blocked / needs input:**
   ```
   🚫 Blocked on <issue> — needs Sir's input
   ```
   Use when Karpathy cannot proceed without a decision or credential from Sir.

   **Canary pass:**
   ```
   ✅ Canary passed — app stable
   ```
   Send after a canary deployment has been live for its soak period with no errors.

3. **Send via `send_message`:**

   ```
   tool: send_message
   target: telegram
   message: "<composed message from step 2>"
   ```

4. **Log to memory** — always record that the notification was sent:

   ```
   tool: memory_write
   key: "notifications/{project_slug}/{event_type}/{timestamp}"
   value: {
     message: "<full message text>",
     sent_at: "<ISO timestamp>",
     event: "<event_type>"
   }
   ```

## Sir's Defaults
- Platform: Telegram only (never Slack, never Discord for prod events)
- Format: short plain English — no markdown tables, no long reports
- Include URL whenever applicable
- One message per event, no follow-ups unless Sir replies

## Pitfalls
- **Never send more than 1 message per event** — deduplicate by checking memory before sending; if a notification for this event was already sent, skip
- Don't send on every intermediate step — only final outcomes (deploy complete, not "starting deploy")
- If health check fails, include the specific error and what Karpathy tried (don't just say "failed")
- Don't send success notifications for health checks — only failures (health-check skill handles this gate)
- Always log to memory after sending — this prevents duplicate sends if the skill is re-run

## Verification
Pass: Sir receives Telegram message within 30s + memory log entry created
