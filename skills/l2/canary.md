---
name: canary
layer: L2
source: gstack
description: Post-deploy canary watch — monitor production for errors in the first 5 minutes after deploy.
version: 1.0.0
tags: [canary, monitoring, post-deploy, production]
---

## When to Use
Immediately after every production deployment (G13). Runs for 5 minutes, then exits.

## Prerequisites
- App deployed and live URL available
- browser_navigate available for visual checks

## Procedure

1. Read live URL from memory
2. Run 3 rounds of checks spaced 90s apart:
   - `curl -s -o /dev/null -w "%{http_code}" <URL>` — HTTP status
   - `browser_navigate(url)` + `browser_vision` — visual sanity
   - Check `/api/health` if exists
3. On any failure: immediately notify Sir via Telegram, stop canary
4. After 3 clean rounds: report "Canary passed — app stable after deploy"

## Sir's Defaults
- Watch window: 5 minutes (3 checks × 90s)
- On failure: Telegram alert immediately
- On pass: one Telegram message "✅ Canary passed"

## Pitfalls
- Vercel cold start on first request — allow up to 10s timeout
- JS errors in console don't always surface in HTTP status — always do browser_vision
- If canary fails, do NOT rollback automatically — notify Sir and wait for instruction

## Verification
Pass: 3 consecutive clean checks with HTTP 200 + visual render correct
Fail: any non-200, blank page, error modal, or JS exception
