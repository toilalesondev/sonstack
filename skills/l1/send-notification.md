---
name: send-notification
layer: L1
source: oh-my-hermes
description: Send a status notification to Sir on Telegram.
version: 1.0.0
tags: [telegram, notification, status, alert]
---

## When to Use
After deployment completes. When health check fails. When Sir needs a status update.
Any important event that Sir should know about.

## Prerequisites
- Hermes Telegram integration configured
- Sir's chat ID in config

## Procedure

1. Compose message — plain English, short, include:
   - What happened (deploy complete / health fail / PR ready)
   - Live URL or PR link if applicable
   - Any action needed from Sir
2. Send via `send_message(target="telegram", message="...")`
3. Log notification sent to memory

## Sir's Defaults
- Platform: Telegram only (never Slack, never Discord for prod events)
- Format: short plain English — no markdown tables, no long reports
- Include URL whenever applicable

## Pitfalls
- Don't spam — one notification per event
- Don't send on every intermediate step — only final outcomes
- If health check fails, include the error and what Karpathy tried

## Verification
Pass: Sir receives Telegram message within 30s
