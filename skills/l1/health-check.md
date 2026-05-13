---
name: health-check
layer: L1
source: oh-my-hermes
description: Verify deployed app is alive — hits all core endpoints, checks response codes and content.
version: 1.0.0
tags: [health, monitoring, post-deploy, verification]
---

## When to Use
After every deployment. On-demand when Sir asks "is it live?". Before sending merge approval.

## Prerequisites
- Live URL in Hermes memory or passed as argument
- App deployed to Vercel

## Procedure

1. Read live URL from memory
2. Hit root endpoint: `curl -s -o /dev/null -w "%{http_code}" <URL>`
3. Hit `/api/health` if exists
4. Use `browser_navigate` to load the page visually
5. Use `browser_vision` to verify it renders correctly (not blank, not error page)
6. Report: URL, status code, visual pass/fail

## Sir's Defaults
- Target: Vercel production URL (from memory)
- Timeout: 10s per endpoint
- Notify Sir on Telegram if health check fails

## Pitfalls
- Cold start on Vercel free tier — first request may be slow, retry once
- 200 doesn't mean working — always do visual check with browser_vision
- Check both / and a core API route

## Verification
Pass: HTTP 200 + browser renders page without error state
Fail: non-200, blank page, JS error in console, "Application error" text visible
