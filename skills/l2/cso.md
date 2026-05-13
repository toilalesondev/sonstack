---
name: cso
layer: L2
source: gstack
description: OWASP Top 10 security scan of the codebase — finds injection, auth, XSS, IDOR, and misconfigurations
version: 1.0.0
tags: [security, owasp, audit, cso, vulnerability]
---

## When to Use

- Before any public launch or beta
- After adding a new authentication flow
- After adding any user-generated content handling
- After adding payment or sensitive data flows
- Quarterly security audit on production codebases
- When a new engineer has been writing backend code unsupervised

## Prerequisites

- Source code is readable (no encrypted/compiled-only files)
- Environment config files accessible (`.env.example`, `next.config.js`, etc.)
- Git history available for `terminal("git log")` if needed
- At minimum: API routes, auth middleware, and DB layer are readable

## Procedure

1. **Map the attack surface**
   ```
   search_files("route.ts", path="src/app/api")       // Next.js API routes
   search_files("middleware.ts", path="src")           // auth middleware
   search_files("*.sql", path=".")                     // raw SQL files
   search_files("schema.prisma", path=".")             // ORM schema
   search_files(".env.example", path=".")              // env var manifest
   ```
   Read each file found. Build a mental map of: what endpoints exist, what auth guards them, what data they touch.

2. **Run OWASP Top 10 checks** — each is a named scan:

   **A01 — Broken Access Control (IDOR / missing auth)**
   Search for routes that:
   - Accept a user-controlled ID (URL param or body) and query it directly without checking ownership
   - Are missing auth middleware or session checks
   - Return more data than the requester should see (over-fetching)
   ```
   search_files("params.id|params\\[.id.|userId|req.body.id", path="src/app/api")
   ```
   For each match: does the query verify the requesting user owns this resource?

   **A02 — Cryptographic Failures**
   - Passwords stored without bcrypt/argon2 (plaintext or weak hash)
   - Sensitive data (SSN, credit card, health info) in logs or unencrypted DB columns
   - JWTs without expiry or signed with a weak secret
   - Session tokens in URLs (visible in server logs)
   ```
   search_files("password|secret|token|jwt", path="src")
   ```

   **A03 — Injection**
   - Raw SQL string interpolation (not parameterized)
   - `eval()`, `Function()`, or `exec()` with user input
   - Shell commands with user-controlled strings
   - Template literals building SQL/shell/HTML
   ```
   search_files("\\$\\{.*\\}.*FROM|exec\\(|eval\\(", path="src")
   ```

   **A04 — Insecure Design**
   - No rate limiting on auth endpoints (brute-force risk)
   - Password reset flows that are predictable or don't expire
   - Business logic that can be gamed (e.g., discount codes with no use-limit check)

   **A05 — Security Misconfiguration**
   ```
   read_file("next.config.js")        // or next.config.ts
   read_file(".env.example")
   ```
   Check for:
   - CORS set to `*` or too broad
   - Debug mode / verbose error messages in production config
   - Security headers missing: `X-Frame-Options`, `Content-Security-Policy`, `X-Content-Type-Options`
   - Default credentials or example secrets left in config

   **A06 — Vulnerable Components**
   ```
   terminal("npm audit --json")
   ```
   Flag any HIGH or CRITICAL severity. Note: don't block on LOW unless it's in a critical path.

   **A07 — Authentication Failures**
   - Session not invalidated on logout
   - Session IDs not rotated after login (session fixation)
   - No account lockout after N failed attempts
   - "Remember me" tokens stored insecurely (localStorage vs httpOnly cookie)
   ```
   search_files("signOut|logout|signIn|session", path="src")
   ```
   Read each match for proper session handling.

   **A08 — Software & Data Integrity**
   - Untrusted CDN resources without Subresource Integrity (SRI) hashes
   - Auto-update mechanisms without signature verification
   - Deserialization of user-supplied data (JSON.parse without schema validation)

   **A09 — Security Logging Failures**
   - No logging on failed auth attempts
   - Sensitive data (passwords, tokens, PII) present in log output
   - No audit trail for admin actions or data mutations
   ```
   search_files("console\\.log|logger\\.", path="src")
   ```
   Check that logs don't contain passwords, tokens, or full user records.

   **A10 — Server-Side Request Forgery (SSRF)**
   - User-controlled URLs passed to `fetch()`, `axios.get()`, or `http.request()`
   - Webhook handlers that follow user-supplied URLs without allowlisting
   ```
   search_files("fetch\\(|axios\\.get\\(|http\\.request\\(", path="src")
   ```
   For each: is the URL user-controlled? Is there an allowlist?

3. **Rate each finding** by severity:
   - **CRITICAL** — exploitable now, data breach or account takeover possible
   - **HIGH** — serious risk, should fix before launch
   - **MEDIUM** — risk with specific conditions, fix within 30 days
   - **LOW** — defense-in-depth, fix when convenient
   - **INFO** — observation, no action required

4. **Write the security report**
   ```
   write_file("docs/security-report.md", <report>)
   ```
   Format:
   ```markdown
   # Security Report — OWASP Top 10 — {DATE}

   ## Summary
   - Critical: N
   - High: N
   - Medium: N
   - Low: N
   - Overall posture: STRONG / MODERATE / WEAK / CRITICAL

   ## Findings

   ### [CRITICAL] A01 — IDOR on /api/documents/:id
   **File:** src/app/api/documents/[id]/route.ts
   **Line:** 14
   **Description:** Document is fetched by ID without checking if the requesting user owns it.
   **Exploit:** Any authenticated user can read any other user's documents by iterating IDs.
   **Fix:** Add `WHERE userId = session.user.id` to the query.

   ### [HIGH] A03 — SQL Injection in search endpoint
   ...

   ## `npm audit` Results
   ...

   ## Remediation Priority
   1. Fix all CRITICAL before any public access
   2. Fix all HIGH before launch
   3. Schedule MEDIUM for next sprint
   ```

5. **For any CRITICAL finding**: surface it immediately in the conversation, do not only write to file.

## Sir's Defaults

- Default report output: `docs/security-report.md`
- Default date: `terminal("date +%Y-%m-%d")`
- Run `npm audit` every time — even if the codebase looks clean
- Check `next.config.js` security headers on every Next.js project — they're almost always misconfigured
- Check auth on every API route, not just new ones

## Pitfalls

- **Don't skip A01 (access control).** It's the #1 OWASP finding year after year. Assume every ID endpoint is vulnerable until proven otherwise.
- **ORM doesn't mean no injection.** Prisma is safer than raw SQL, but raw queries (`$queryRaw`) still need scrutiny.
- **"Internal tool" is not a security waiver.** Internal users have credentials too. Breached internal tools = breached production data.
- **`npm audit` false positives exist** but don't dismiss them without reading. A "low" severity in a critical auth path is still a HIGH for your system.
- **Log audit is often skipped.** Don't skip it. Logging passwords is embarrassingly common and a GDPR/HIPAA landmine.

## Verification

- All 10 OWASP categories have been checked (not skipped because "doesn't apply")
- `docs/security-report.md` exists with severity counts in the summary
- Any CRITICAL finding was surfaced in the conversation, not just the file
- `npm audit` was run and results are recorded
- Every HIGH+ finding has a specific file, line number, and proposed fix
