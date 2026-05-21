---
name: security-review
description: >
  Audit code changes for security vulnerabilities. Use this skill whenever the user:
  asks for a security review or security check, wants to audit a diff, PR, or file for
  vulnerabilities, says "is this secure?", "check this for security issues", "review before merge",
  mentions OWASP, CSRF, injection, XSS, hardcoded credentials, secrets, or access control,
  pastes or points to an API route / Supabase query / auth code and wants it reviewed,
  or runs /security-review as a slash command.
  Stack: Next.js 15 App Router + Supabase + Cloudflare R2 + TypeScript.
  Always invoke this skill proactively when the user merges or commits code touching
  app/api/, middleware.ts, lib/supabase/, or supabase/migrations/.
---

# Security Review

Audit every changed file for security issues. Never pass a diff silently — always report what
was checked and the verdict, even when the code is clean. A clean report with evidence is
more useful than silence.

---

## Step 1 — Get the Diff

If the user provided a diff or file contents, use them directly.

Otherwise run:
```bash
git diff HEAD                    # all uncommitted changes
git diff main..HEAD              # everything on current branch vs main
git diff HEAD~1 HEAD             # last commit only
```

If reviewing specific files: read them with the Read tool.

List each file you will review before starting. This sets expectations.

---

## Step 2 — Classify Each File

Before checking, classify each changed file. This tells you which checks matter most.

| File matches | Top-priority checks |
|---|---|
| `app/api/**/route.ts` | Auth guard · input validation · CSRF · workspace scope |
| `middleware.ts` | Route coverage gaps · auth bypass |
| `lib/supabase/client.ts` or `server.ts` | Service key exposure · client/server mixing |
| `supabase/migrations/*.sql` | RLS enabled · policies exist · no `using (true)` |
| `stores/*.ts` | Sensitive data in persisted localStorage state |
| `components/**/*.tsx` | `dangerouslySetInnerHTML` · client-side-only auth trust |
| `app/(app)/**/page.tsx` | Missing `usePermissions()` · data scoped to workspace |
| `.env*` or config files | NEXT_PUBLIC_ leaks · secrets committed |
| `app/api/r2/presign/route.ts` | Path traversal · unrestricted file types · SSRF |
| `app/api/webhooks/**/route.ts` | Signature verification · SSRF from payload URLs |

---

## Step 3 — The Checklist

Work through every applicable section for each changed file.
Mark each item ✓ (clean), ✗ (issue found), or — (not applicable, state why).

---

### 🔑 CREDENTIALS & SECRETS

- [ ] No hardcoded API keys, JWTs, or passwords — scan for: `eyJ`, `sk_`, `pk_`, `rzp_live_`, `rzp_test_`, `Bearer `, any assignment `secret =`, `password =`, `api_key =` followed by a string literal
- [ ] `SUPABASE_SERVICE_KEY` (or `SERVICE_ROLE_KEY`) appears **only** in server-side code and is never referenced in any `NEXT_PUBLIC_` variable or any file under `components/`, `stores/`, or `hooks/`
- [ ] No credentials in comments (`// TODO: remove this key`) or test fixtures committed to repo
- [ ] `.env.local` is in `.gitignore` — verify if `.gitignore` was changed

---

### A01 — BROKEN ACCESS CONTROL  *(most critical for this codebase)*

**Supabase queries — workspace scoping:**
- [ ] Every query that touches workspace data includes `.eq('workspace_id', workspaceId)` — no naked `select('*')` on multi-tenant tables
- [ ] `workspace_id` comes from a verified server-side source (Zustand active workspace confirmed by server), not from client-supplied request body

**API route auth:**
- [ ] Every handler starts with `const { data: { user } } = await supabase.auth.getUser()`
- [ ] Returns `401` immediately if `!user` — no logic runs before auth is confirmed
- [ ] **Never use `supabase.auth.getSession()` server-side** — it trusts the JWT from the browser without server revalidation; `getUser()` hits the Supabase API to confirm validity
- [ ] Role-based operations (e.g. only trainer can create plans) verified server-side, not just via `usePermissions()` in the frontend

**Components and pages:**
- [ ] Role checks use `usePermissions().can('action:resource')` — no raw `role === 'owner'` comparisons
- [ ] Sensitive data not rendered in the DOM before auth/permission state resolves (no flash of unauthorized content)

---

### A03 — INJECTION

- [ ] `dangerouslySetInnerHTML` is absent, or if present, input has been sanitized with a library like `DOMPurify` — raw user strings are never passed in
- [ ] All API route bodies validated with a **Zod schema** before any data is used:
  ```typescript
  // ✓ correct
  const body = RequestSchema.parse(await req.json())
  // ✗ wrong
  const { name, workspaceId } = await req.json()
  ```
- [ ] R2 presign route: `filename` and `contentType` are validated — no path traversal (`../`, `%2F`, null bytes), content type is from an allowlist
- [ ] No `eval()`, `new Function(code)`, or dynamic `require(variable)` anywhere
- [ ] `JSON.parse()` on user input is wrapped in try/catch

---

### A07 — AUTHENTICATION FAILURES

- [ ] `middleware.ts` protects **all** authenticated routes — check that new routes added in this diff are covered by the matcher pattern, not just `/dashboard`
- [ ] Supabase SSR client (`lib/supabase/server.ts`) uses `getAll`/`setAll` cookie methods — this is required for correct session refresh; missing this breaks token rotation silently
- [ ] Auth state is never read from `localStorage` or `document.cookie` directly in server components or API routes
- [ ] No new "public" API routes that accidentally expose data that should be gated

---

### CSRF — CROSS-SITE REQUEST FORGERY

> Next.js App Router **route handlers** (`route.ts`) have **no built-in CSRF protection**.
> Server Actions are protected by the framework. Route handlers are not.

For every `POST`, `PUT`, `PATCH`, or `DELETE` route handler in the diff:

- [ ] Validates `Origin` header matches the app domain:
  ```typescript
  const origin = request.headers.get('origin')
  if (origin !== process.env.NEXT_PUBLIC_APP_URL) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }
  ```
  **OR** relies on Supabase session cookies with `SameSite=Lax` (Supabase SSR sets this — verify the Supabase client is set up correctly)
- [ ] No state-changing operation reachable via a `GET` handler
- [ ] Webhook endpoints (`/api/webhooks/*`) verify a provider signature (e.g. `X-ZKTeco-Signature`, `X-Razorpay-Signature`) — external webhooks must not be callable by arbitrary clients

---

### A05 — SECURITY MISCONFIGURATION

- [ ] No `console.log`, `console.error` printing: full user objects, JWT tokens, Supabase query results with PII, env variables — logs visible in production should be safe to expose
- [ ] `next.config.ts` includes security headers if this is a new or modified config:
  ```typescript
  headers: [{ source: '/(.*)', headers: [
    { key: 'X-Frame-Options', value: 'DENY' },
    { key: 'X-Content-Type-Options', value: 'nosniff' },
    { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  ]}]
  ```
- [ ] CORS `Access-Control-Allow-Origin: *` not set on state-changing endpoints
- [ ] Error responses sent to the client don't include stack traces, SQL error details, or internal paths

---

### A08 — SOFTWARE & DATA INTEGRITY (Input Validation)

- [ ] Numeric inputs (amounts, counts, indexes) validated as positive integers before use
- [ ] UUIDs validated as proper UUID format before database queries (`z.string().uuid()`)
- [ ] Date inputs validated as real dates
- [ ] File uploads: content type validated server-side (not just from the `Content-Type` header the client sends — that's spoofable), and file size capped
- [ ] Webhook payloads raw-saved before processing (already in schema as `payment_webhooks_raw` / `zkteco_raw_logs`) — new webhook handlers should follow this pattern

---

### A10 — SERVER-SIDE REQUEST FORGERY (SSRF)

- [ ] Server-side `fetch()` calls do not use user-supplied URLs without an allowlist
- [ ] `/api/r2/presign`: generated key is scoped to a prefix (e.g. `workspace_id/`) — a user cannot generate a presigned URL for another workspace's files
- [ ] Webhook handlers do not make outbound HTTP requests to URLs extracted from the incoming payload

---

### 🗄️ DATABASE / RLS  *(for migration files only)*

- [ ] Every new table has `alter table <name> enable row level security`
- [ ] Every new table has **at least one policy** — `enable rls` with no policies locks out everyone including the app (a silent bug, not a security feature)
- [ ] No policy body is `using (true)` or `with check (true)` unless it is explicitly for public/platform data and that's documented
- [ ] Policies for workspace data filter by `workspace_id` — not just by `user_id`
- [ ] `SUPABASE_SERVICE_KEY` is only used in server-side admin operations and never passed to RLS-governed queries unless intentionally bypassing RLS (flag this)

---

### 🗃️ CLIENT STATE  *(for Zustand stores)*

- [ ] Zustand `persist()` config does not store: raw auth tokens, Supabase session objects, payment card details, full phone numbers, or bulk user records
- [ ] Active workspace and role stored in persist is safe to expose (it's not sensitive), but raw API credentials are not

---

## Step 4 — Write the Report

Use this exact format. Every severity level must appear even if empty.

```
## 🔍 Security Review — [filename / "Staged Changes" / "Branch diff"]
**Stack:** Next.js 15 App Router + Supabase + Cloudflare R2
**Files reviewed:** [list files]
**Lines changed:** ~[N]
**Date:** [today]

---

### 🚨 CRITICAL
Data breach, account takeover, or credential exposure. Fix before merging.

> [finding block — see format below] OR > ✅ None found.

---

### 🔴 HIGH
Exploitable under normal use. Fix before merging.

> [finding block] OR > ✅ None found.

---

### 🟡 MEDIUM
Exploitable under specific conditions or weakens overall defence. Fix soon.

> [finding block] OR > ✅ None found.

---

### 🟢 LOW / INFO
Best-practice gaps, hardening improvements. Fix when convenient.

> [finding block] OR > ✅ None found.

---

### ✅ Checks Run — Clean
Prove the review was thorough by listing what was verified and found safe.

| Check | Verdict |
|---|---|
| A01 — Workspace scoping | ✓ All queries include workspace_id |
| A01 — API auth guard | ✓ getUser() called, 401 on missing user |
| A03 — Zod validation | ✓ Request body validated |
| Credentials | ✓ No hardcoded secrets |
| CSRF | ✓ Origin header validated |
| [etc.] | |

---

### 💡 Recommendations
Optional improvements that don't qualify as vulnerabilities.
```

---

## Finding Format

```
**[SEVERITY] [OWASP ref] — [Short title]**
File: `path/to/file.ts` line [N]
Risk: [one sentence on what an attacker can do]
Code:
\```typescript
// the problematic code
\```
Fix:
\```typescript
// the corrected code
\```
```

---

## Severity Reference

| Level | Trigger |
|---|---|
| **CRITICAL** | Service key in client code · Missing auth on API route · RLS bypass · Hardcoded production secret · Unscoped cross-workspace query |
| **HIGH** | Missing workspace_id scope · CSRF on payment/state-change endpoint · XSS via dangerouslySetInnerHTML · getSession() instead of getUser() server-side · Missing middleware coverage for new route |
| **MEDIUM** | Missing Zod validation on API body · SSRF potential in presign · Missing security headers · Sensitive data logged · Unrestricted file type upload |
| **LOW** | console.log with user object · Missing rate-limit comment · Minor CORS looseness · Non-persisted sensitive Zustand field |
