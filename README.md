# Sonstack

**Sir's unified skill pack for Hermes Agent.**

gstack intelligence + oh-my-hermes wiring — rewritten in Hermes-native language, Sir-specific defaults.

## Architecture

```
Layer 1 (L1) — Project wiring          ← from oh-my-hermes, Hermes-native
Layer 2 (L2) — Quality gates           ← from gstack, rewritten for Hermes tools
```

**L1** runs once per project setup: wires Supabase, Vercel, GitHub, kanban, Telegram notifications, engine routing.

**L2** runs on every task/sprint: product stress-test, plan reviews, code review, QA, security, ship discipline.

## Sir's defaults (baked in)

- **Notifications:** Telegram (not Slack)
- **Deploy:** Vercel
- **DB:** Supabase
- **Runtime:** VPS (`karpathy@vps`) + macOS
- **Design:** `sketch` skill on VPS (Playwright HTML), `design-shotgun` on Mac (DALL-E)
- **Auth:** GitHub SSH, PAT in 1Password
- **Language:** EN + VI

## 14-Gate Flow

```
G01  /clarify-requirements     L1   7 questions → memory
G02  /product-brief            L1   PRODUCT_BRIEF.md
G03  /office-hours             L2   YC stress-test
G04  write plan.md             —    Karpathy writes
G05  /plan-ceo-review          L2   ambitious enough?
G06  /plan-eng-review          L2   architecture locked
G07  /sketch or /design-html   L1+L2 variants → Sir picks
G08  /choose-engine            L1   route to claude-code or codex
G09  /review + /cso + /qa      L2   code review + security + QA
G10  /ship + /create-pr        L1+L2 CHANGELOG + PR
G11  /await-merge-approval     L1   Sir YES/NO on Telegram
G12  /deploy-to-vercel         L1   live URL
G13  /canary + /health-check   L1+L2 production watch
G14  /cto-status-report        L1   notify Sir, docs updated
```

**Sir's touchpoints:** Gate 1 (7 Qs) + Gate 7 (pick design) + Gate 11 (YES/NO). That's it.

## Install

```bash
git clone https://github.com/toilalesondev/sonstack ~/.hermes/profiles/karpathy/skills/sonstack
```

## Structure

```
skills/
  l1/   ← Hermes-native project wiring (oh-my-hermes lineage)
  l2/   ← Quality gates (gstack lineage, Hermes tools)
workflows/
  idea-to-deploy.md    ← master 14-gate orchestrator
  cto-loop.md          ← autonomous CTO loop
  design-to-code.md    ← design handoff to build
bin/
  gate-check           ← verify all gate skills installed
docs/
  architecture.md
  gates.md
```

## Credits

- [gstack](https://github.com/garrytan/gstack) — Garry Tan's engineering workflow skills
- [oh-my-hermes](https://github.com/Salomondiei08/oh-my-hermes) — Hermes Agent skill pack
