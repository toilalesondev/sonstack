---
name: create-github-pr
layer: L1
source: oh-my-hermes
description: Open a pull request on GitHub after implementation is complete on a feature branch.
version: 1.0.0
tags: [github, pr, pull-request, ship]
---

## When to Use
After G08 (build) passes. Before G11 (merge approval). Implementation complete on feature branch.

## Prerequisites
- `gh` CLI authenticated as toilalesondev
- Feature branch pushed to GitHub
- Tests passing

## Procedure

1. `git branch --show-current` — confirm on feature branch, not main
2. `git log main..HEAD --oneline` — collect commits for PR body
3. `gh pr create --title "<conventional title>" --body "<summary of changes, what was built, how to test>" --base main`
4. Capture PR URL from output
5. Save PR URL to memory
6. Report PR URL to Sir

## Sir's Defaults
- Base branch: `main`
- GitHub account: toilalesondev
- PR title format: `feat: <description>` / `fix: <description>` / `chore: <description>`

## Pitfalls
- Never open PR from main → main
- Always include "How to test" in PR body
- If CI fails after PR open, fix before proceeding to G11

## Verification
Pass: `gh pr view` returns PR in OPEN state, CI checks started
