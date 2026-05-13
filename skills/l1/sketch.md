---
name: sketch
layer: L1
source: hermes-native
description: Generate 3 HTML design variants, screenshot with Playwright, send to Sir for pick. VPS-safe (no external APIs).
version: 1.0.0
tags: [design, sketch, html, mockup, variants, playwright]
---

## When to Use
Gate 7 (design step) on VPS projects. Replaces `design-shotgun` when no external design APIs are available.
Use whenever Sir needs to pick a UI direction before implementation begins.

## Prerequisites
- Python 3 available on VPS
- Playwright installed: `pip install playwright && python -m playwright install chromium`
- Project directory writable (for `sketches/` folder)

## Procedure

1. **Ask Sir for design brief** — get answers to these three questions before writing any HTML:
   - What screen or flow is this? (e.g., "login page", "dashboard hero", "onboarding step 2")
   - What feel? Give me adjectives. (e.g., "clean minimal", "bold high-contrast", "playful warm")
   - What is the core action? (the one thing the user does on this screen)

2. **Write 3 standalone HTML variant files**

   Paths:
   - `sketches/001-<name>/index.html`
   - `sketches/002-<name>/index.html`
   - `sketches/003-<name>/index.html`

   Replace `<name>` with a slug of the screen (e.g., `login`, `dashboard`, `onboarding`).

   Each file must be:
   - **Fully self-contained** — all CSS inline or via CDN (Tailwind CDN is fine)
   - **Real content** — no lorem ipsum. Use actual copy, real button labels, real field names
   - **Interactive** — hover states on buttons/links; at least one state transition (e.g., button press, input focus, menu open)
   - **A distinct design stance** — the 3 variants must feel clearly different, not just color swaps:
     - Variant 001: e.g., **minimal** — lots of space, one dominant action, quiet typography
     - Variant 002: e.g., **bold** — strong color, large type, high visual weight
     - Variant 003: e.g., **playful** — rounded corners, friendly tone, subtle animation or gradient

   Dark theme by default unless Sir specified otherwise in step 1.

3. **Start local HTTP server**

   ```
   tool: terminal
   command: "python3 -m http.server 8899"
   background: true
   workdir: "<project root>"
   ```

   This must be run from the project root so paths like `/sketches/001-.../index.html` resolve correctly.

4. **Screenshot all 3 variants with Playwright**

   Create `sketches/screenshots/` directory, then run:

   ```python
   import asyncio
   from playwright.async_api import async_playwright

   async def shot():
       async with async_playwright() as p:
           b = await p.chromium.launch(args=[
               '--no-sandbox',
               '--disable-setuid-sandbox',
               '--disable-dev-shm-usage'
           ])
           page = await b.new_page()
           await page.set_viewport_size({'width': 390, 'height': 844})
           variants = ['001-<name>', '002-<name>', '003-<name>']
           for name in variants:
               await page.goto(
                   f'http://localhost:8899/sketches/{name}/index.html',
                   wait_until='networkidle'
               )
               await page.screenshot(
                   path=f'sketches/screenshots/{name}.png',
                   full_page=True
               )
               print(f'✅ screenshotted {name}')
           await b.close()

   asyncio.run(shot())
   ```

   Replace `'001-<name>'` etc. with the actual folder names from step 2.

5. **Send screenshots to Sir**

   After Playwright finishes, include all 3 in your reply:

   ```
   MEDIA:/absolute/path/to/sketches/screenshots/001-<name>.png
   MEDIA:/absolute/path/to/sketches/screenshots/002-<name>.png
   MEDIA:/absolute/path/to/sketches/screenshots/003-<name>.png
   ```

   Add brief labels: "001 — Minimal", "002 — Bold", "003 — Playful" (or whatever the stances are).

6. **Wait for Sir to pick a variant**

   Sir replies with a number (1, 2, or 3) or a name. This is a **blocking gate** — do not proceed
   until Sir has explicitly chosen.

7. **Save approved design to memory**

   ```
   tool: memory_write
   key: "project/{project_slug}/approved_sketch"
   value: {
     variant: "00X-<name>",
     path: "sketches/00X-<name>/index.html",
     stance: "<minimal|bold|playful>",
     approved_at: "<ISO timestamp>"
   }
   ```

   After saving: commit the winner, delete the other two variant folders.

   ```bash
   git add sketches/00X-<name>/
   git commit -m "chore: add approved sketch (variant 00X)"
   rm -rf sketches/001-<name> sketches/002-<name>  # delete the two rejected ones
   ```

## Sir's Defaults
- **Viewport:** 390×844 (iPhone 14 Pro, mobile-first)
- **Theme:** dark unless Sir explicitly says light
- **Always 3 variants**, always clearly different design stances — not just palette swaps
- **After pick:** commit the winner, delete the others

## Pitfalls
- **Kill the HTTP server after screenshots** — don't leave it running:
  ```bash
  terminal("pkill -f 'http.server 8899'")
  ```
- **`install-deps` needs sudo on VPS — skip it.** Use `--no-sandbox` + `--disable-setuid-sandbox` in Playwright args. This is already in the script above.
- **Always serve via HTTP, not `file://`** — `file://` causes CORS issues with Tailwind CDN and any relative asset loads. The `python3 -m http.server` in step 3 handles this.
- **Multi-screen SPA:** if a variant has multiple states to capture, use `page.click()` + `page.wait_for_timeout(300)` between interactions before calling `screenshot()`.
- **No lorem ipsum** — Sir evaluates real copy. Placeholder text makes designs look unfinished and wastes the review.
- **Don't start HTTP server if port 8899 is already in use** — check first: `lsof -i :8899`. Kill any existing process before starting.

## Verification
Pass: 3 PNG screenshots sent to Sir, Sir has replied with a variant choice, memory entry written
Fail: blank screenshots (page didn't load — check HTTP server is running), Playwright crashes (missing `--no-sandbox`), CORS error in console (serving via `file://` instead of HTTP)
