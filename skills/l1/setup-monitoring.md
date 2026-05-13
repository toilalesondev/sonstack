---
name: setup-monitoring
layer: L1
source: oh-my-hermes
description: Wire error tracking and uptime monitoring for a newly deployed app.
version: 1.0.0
tags: [monitoring, sentry, uptime, post-deploy]
---

## When to Use
First deploy of a new project (G13). When Sir asks "set up monitoring".

## Prerequisites
- App deployed and live
- Sentry account (or use Vercel's built-in analytics)

## Procedure

1. Check if Sentry already configured: `grep -r "SENTRY_DSN" .env* 2>/dev/null`
2. If not: add `@sentry/nextjs` (for Next.js projects): `npm install @sentry/nextjs`
3. Add `SENTRY_DSN` to Vercel env vars via `vercel env add SENTRY_DSN production`
4. For uptime: use Better Uptime or UptimeRobot free tier — create monitor for live URL
5. Test: trigger a test error, verify it appears in Sentry
6. Notify Sir via Telegram: "Monitoring live. Errors → Sentry. Uptime → <service>."

## Sir's Defaults
- Error tracking: Sentry (free tier)
- Uptime: Better Uptime or UptimeRobot
- Alert destination: Telegram

## Pitfalls
- Skip if project is a private tool (Sir only) — monitoring overkill
- Vercel Analytics covers basic usage without Sentry setup cost
- Don't block deploy on monitoring setup — it's optional on first ship

## Verification
Pass: test error appears in Sentry dashboard within 30s
