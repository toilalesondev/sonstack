---
name: health-check
layer: L1
source: oh-my-hermes
description: Verify deployed app is alive — hits all core endpoints, checks response codes and content.
version: 1.1.0
tags: [health, monitoring, post-deploy, verification]
---

## When to Use
After every deployment. On-demand when Sir asks "is it live?". Before sending merge approval.

## Prerequisites
- Live URL in Hermes memory or passed as argument
- App deployed to Vercel

## Procedure

1. **Read live URL from memory**

   ```
   tool: memory_read
   key: "project/{project_slug}/live_url"
   ```

   If not in memory, ask Sir for the URL before proceeding.

2. **Cold-start HTTP hit — root endpoint**

   ```bash
   curl -s -o /dev/null -w "%{http_code}" --max-time 15 <URL>
   ```

   - If response code is `200`: proceed.
   - If timeout or non-200 on first attempt: **wait 5 seconds**, then retry once.
   - If second attempt also fails: skip to step 7 (failure path) immediately.

3. **Check `/api/health` endpoint**

   ```bash
   curl -s -o /dev/null -w "%{http_code}" --max-time 15 <URL>/api/health
   ```

   - Expected: `200`. A `404` here is acceptable only if the app has no health endpoint (note it).
   - Any `5xx` → failure path (step 7).

4. **Load page visually in browser**

   ```
   tool: browser_navigate
   url: <URL>
   ```

   Wait for page to fully load (networkidle or DOMContentLoaded).

5. **Visual integrity check**

   ```
   tool: browser_vision
   question: "Is the page rendering correctly? Any error messages, blank screens, or broken layouts?"
   ```

   - Pass: page shows expected content, no error banners, no blank white/black screen.
   - Fail: "Application error", Next.js error overlay, blank screen, broken CSS → failure path.

6. **Check browser console for JS errors**

   ```
   tool: browser_console
   ```

   - Ignore benign warnings (e.g., cookie SameSite notices, minor deprecations).
   - Fail on: uncaught exceptions, failed network requests to own API routes, React render errors.

7. **On any failure — notify immediately**

   ```
   tool: send_message
   target: telegram
   message: "❌ Health check failed: <URL> — <error detail>"
   ```

   Include the specific check that failed (curl code, visual issue, console error). Do NOT continue silently.

8. **On full pass — report inline only**

   Reply in the current conversation with a brief status: URL, HTTP code, visual pass, console clean.
   **Do NOT send a Telegram notification on success** — only failures warrant a ping.

## Sir's Defaults
- Target: Vercel production URL (from memory)
- Timeout: 15s per curl request (with 5s retry gap)
- Notify Sir on Telegram **only** on failure
- Check both `/` and `/api/health`

## Pitfalls
- Cold start on Vercel free tier — first request may be slow; the 5s retry in step 2 handles this
- HTTP 200 does not mean working — always complete the visual check (step 5)
- Do not skip browser_console — silent JS errors break features without surfacing in HTTP codes
- Do not spam Telegram on every health check — success is silent, failure is loud

## Verification
Pass: HTTP 200 + browser renders page without error state + no uncaught JS errors in console
Fail: non-200 after retry, blank page, JS error in console, "Application error" text visible
