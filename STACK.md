# FitDesk — Stack Reference

## Core Stack

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Framework | Next.js 15 | Latest | React, App Router, Server Components |
| Hosting | Cloudflare Workers & Pages | — | Auto-deploy from GitHub |
| Database | Supabase Postgres | — | Full Postgres + RLS |
| Auth | Supabase Auth | — | OTP via phone/email, JWT |
| Realtime | Supabase Realtime | — | Live updates via websocket |
| File storage | Cloudflare R2 | — | Videos, photos, screenshots |
| Edge functions | Supabase Edge Functions | — | Webhooks, background logic |
| Styling | Tailwind CSS + shadcn/ui | Latest | Utility-first + components |
| State | Zustand | Latest | React store |
| Testing (unit) | Vitest | Latest | — |
| Testing (e2e) | Playwright | Latest | — |
| CI/CD | GitHub Actions | — | Runs on every push |
| Mobile | Capacitor | Latest | Added in v2 |
| Notifications | Gupshup | — | WhatsApp + SMS, added in v2 |
| Payments | Manual v1, Razorpay v2 | — | Provider-agnostic schema |

---

## Next.js Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Nothing special needed for Cloudflare Pages
  // Cloudflare auto-detects Next.js
}

export default nextConfig
```

---

## Environment Variables

```bash
# .env.local (never commit this)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_KEY=eyJ...   # server-side only, never expose to client

CLOUDFLARE_ACCOUNT_ID=xxx
CLOUDFLARE_R2_ACCESS_KEY=xxx
CLOUDFLARE_R2_SECRET_KEY=xxx
CLOUDFLARE_R2_BUCKET=fitdesk-uploads

# v2 only
RAZORPAY_KEY_ID=rzp_live_xxx
RAZORPAY_KEY_SECRET=xxx
GUPSHUP_API_KEY=xxx
```

```bash
# .env.example (commit this — no real values)
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_KEY=your_service_key
CLOUDFLARE_ACCOUNT_ID=your_account_id
CLOUDFLARE_R2_ACCESS_KEY=your_r2_access_key
CLOUDFLARE_R2_SECRET_KEY=your_r2_secret_key
CLOUDFLARE_R2_BUCKET=fitdesk-uploads
```

Note: NEXT_PUBLIC_ prefix = safe to expose to browser.
Without prefix = server only, never sent to browser.

---

## Folder Structure

```
fitdesk/
├── .github/
│   └── workflows/
│       └── ci.yml
├── app/                          # Next.js App Router root
│   ├── (auth)/                   # route group — no layout
│   │   ├── login/
│   │   │   └── page.tsx          # /login
│   │   └── verify/
│   │       └── page.tsx          # /verify
│   ├── (onboarding)/             # route group
│   │   ├── onboarding/
│   │   │   └── page.tsx          # /onboarding
│   │   ├── create-gym/
│   │   │   └── page.tsx
│   │   └── join-gym/
│   │       └── page.tsx
│   ├── (app)/                    # route group — authenticated
│   │   ├── layout.tsx            # authenticated layout with nav
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── members/
│   │   │   ├── page.tsx          # /members
│   │   │   └── [id]/
│   │   │       └── page.tsx      # /members/123
│   │   ├── workout/
│   │   │   ├── page.tsx
│   │   │   └── [planId]/
│   │   │       └── page.tsx
│   │   ├── diet/
│   │   │   └── page.tsx
│   │   ├── attendance/
│   │   │   └── page.tsx
│   │   ├── progress/
│   │   │   └── page.tsx
│   │   ├── settings/
│   │   │   ├── gym/
│   │   │   │   └── page.tsx
│   │   │   ├── billing/
│   │   │   │   └── page.tsx
│   │   │   └── hardware/
│   │   │       └── page.tsx
│   │   └── workspace/
│   │       └── switch/
│   │           └── page.tsx
│   ├── api/                      # Next.js API routes (server only)
│   │   ├── webhooks/
│   │   │   └── zkteco/
│   │   │       └── route.ts      # POST /api/webhooks/zkteco
│   │   └── r2/
│   │       └── presign/
│   │           └── route.ts      # POST /api/r2/presign
│   ├── layout.tsx                # root layout
│   ├── page.tsx                  # landing page /
│   └── globals.css
├── components/
│   ├── ui/                       # shadcn/ui components
│   ├── workspace/                # workspace switcher, cards
│   ├── member/                   # member list, card, form
│   ├── workout/                  # plan, session, exercise
│   ├── diet/                     # meal, item components
│   ├── attendance/               # QR, GPS, history
│   └── shared/                   # buttons, inputs, badges
├── hooks/                        # React hooks (was composables in Nuxt)
│   ├── use-workspace.ts          # active workspace, role checks
│   ├── use-auth.ts               # current user, sign in/out
│   ├── use-member.ts             # member CRUD
│   ├── use-workout.ts            # plan logic, session advancement
│   ├── use-attendance.ts         # check-in, GPS, QR
│   └── use-permissions.ts        # role-based permission checks
├── stores/                       # Zustand stores
│   ├── workspace-store.ts        # active workspace + role
│   └── auth-store.ts             # current user
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # browser supabase client
│   │   ├── server.ts             # server supabase client
│   │   └── middleware.ts         # auth middleware helper
│   └── utils.ts                  # cn() and other utils
├── middleware.ts                  # Next.js middleware (auth guard)
├── types/
│   └── index.ts                  # all TypeScript types
├── supabase/
│   ├── migrations/
│   │   ├── 0001_initial.sql
│   │   ├── 0002_rls.sql
│   │   └── 0003_seed.sql
│   └── functions/
│       ├── webhook-zkteco/
│       │   └── index.ts
│       └── send-notification/
│           └── index.ts
├── tests/
│   ├── unit/
│   │   ├── workout-logic.test.ts
│   │   ├── attendance.test.ts
│   │   └── permissions.test.ts
│   ├── integration/
│   │   ├── rls-owner.test.ts
│   │   ├── rls-trainer.test.ts
│   │   └── rls-member.test.ts
│   └── e2e/
│       ├── gym-setup.spec.ts
│       ├── member-flow.spec.ts
│       └── trainer-flow.spec.ts
├── public/
├── README.md
├── STACK.md
├── SCHEMA.md
├── CLAUDE.md
├── DEVLOG.md
├── .env.example
├── .env.local                    # gitignored
├── next.config.ts
├── tailwind.config.ts
├── middleware.ts                 # auth guard
└── package.json
```

---

## Key Packages

```json
{
  "dependencies": {
    "next": "^15.x",
    "react": "^19.x",
    "react-dom": "^19.x",
    "@supabase/supabase-js": "^2.x",
    "@supabase/ssr": "^0.x",
    "zustand": "^5.x",
    "zod": "^3.x",
    "date-fns": "^3.x"
  },
  "devDependencies": {
    "vitest": "^1.x",
    "@playwright/test": "^1.x",
    "typescript": "^5.x",
    "@types/node": "^20.x",
    "@types/react": "^19.x",
    "@types/react-dom": "^19.x",
    "wrangler": "^3.x"
  }
}
```

---

## Supabase Client Setup

Two separate clients. This is critical — never mix them up.

```typescript
// lib/supabase/client.ts
// Use this in Client Components ('use client')
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

```typescript
// lib/supabase/server.ts
// Use this in Server Components, API routes, middleware
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        },
      },
    }
  )
}
```

```typescript
// middleware.ts (root of project, not inside app/)
// Runs on every request — refreshes auth session
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  const { data: { user } } = await supabase.auth.getUser()

  // Redirect to login if not authenticated and trying to access app
  if (!user && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return supabaseResponse
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

---

## Zustand Store

```typescript
// stores/workspace-store.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import type { Workspace, WorkspaceMember, Role } from '@/types'

interface WorkspaceState {
  activeWorkspace: Workspace | null
  activeRole: Role | null
  workspaces: WorkspaceMember[]
  setActiveWorkspace: (wm: WorkspaceMember) => void
  setWorkspaces: (workspaces: WorkspaceMember[]) => void
  reset: () => void
}

export const useWorkspaceStore = create<WorkspaceState>()(
  persist(
    (set) => ({
      activeWorkspace: null,
      activeRole: null,
      workspaces: [],
      setActiveWorkspace: (wm) => set({
        activeWorkspace: wm.workspace,
        activeRole: wm.role,
      }),
      setWorkspaces: (workspaces) => set({ workspaces }),
      reset: () => set({ activeWorkspace: null, activeRole: null, workspaces: [] }),
    }),
    {
      name: 'fitdesk-workspace', // persists to localStorage
    }
  )
)
```

---

## Permissions Hook

```typescript
// hooks/use-permissions.ts
'use client'
import { useWorkspaceStore } from '@/stores/workspace-store'

const PERMISSIONS = {
  'member:create':     ['owner', 'manager', 'receptionist'],
  'member:view':       ['owner', 'manager', 'receptionist', 'trainer'],
  'member:edit':       ['owner', 'manager', 'receptionist'],
  'member:suspend':    ['owner', 'manager'],
  'workout:create':    ['owner', 'trainer'],
  'workout:view':      ['owner', 'manager', 'trainer'],
  'workout:edit':      ['owner', 'trainer'],
  'diet:create':       ['owner', 'trainer'],
  'diet:view':         ['owner', 'manager', 'trainer'],
  'attendance:mark':   ['owner', 'manager', 'receptionist', 'trainer'],
  'attendance:view':   ['owner', 'manager', 'receptionist', 'trainer'],
  'payment:mark':      ['owner', 'manager', 'receptionist'],
  'payment:view':      ['owner', 'manager', 'receptionist'],
  'billing:manage':    ['owner'],
  'branch:manage':     ['owner'],
  'hardware:manage':   ['owner', 'manager'],
  'video:upload':      ['owner', 'trainer'],
  'video:view':        ['owner', 'manager', 'trainer', 'member'],
} as const

type Permission = keyof typeof PERMISSIONS

export function usePermissions() {
  const activeRole = useWorkspaceStore((s) => s.activeRole)

  function can(permission: Permission): boolean {
    if (!activeRole) return false
    return (PERMISSIONS[permission] as readonly string[]).includes(activeRole)
  }

  return { can }
}
```

---

## Cloudflare R2 Upload Pattern

Files never pass through your server.

```typescript
// Client component
async function uploadFile(file: File) {
  // 1. Get presigned URL from your API route
  const { url, key } = await fetch('/api/r2/presign', {
    method: 'POST',
    body: JSON.stringify({ filename: file.name, contentType: file.type }),
    headers: { 'Content-Type': 'application/json' }
  }).then(r => r.json())

  // 2. Upload directly to R2 — never touches your server
  await fetch(url, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type }
  })

  // 3. Save the key to Supabase (not the full URL)
  return key
}
```

```typescript
// app/api/r2/presign/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function POST(req: NextRequest) {
  const { filename, contentType } = await req.json()

  // Generate presigned URL using AWS SDK (R2 is S3-compatible)
  // Return { url, key }
}
```

---

## Auth Flow

```
User lands on /
→ middleware.ts checks session
→ Not logged in + accessing /dashboard → redirect /login
→ /login → enter phone number → OTP sent via Supabase
→ /verify → enter OTP → session created
→ Check workspaces in Supabase:
   0 workspaces → /onboarding
   1 workspace  → /dashboard (auto-select)
   2+ workspaces → /workspace/switch
```

---

## Server vs Client Components

Critical in Next.js App Router. Get this wrong and nothing works.

```
Server Component (default — no directive needed)
→ Runs on server only
→ Can use lib/supabase/server.ts
→ Cannot use hooks, useState, useEffect
→ Cannot use browser APIs
→ Use for: data fetching, layouts, pages that show data

Client Component ('use client' at top of file)
→ Runs in browser
→ Must use lib/supabase/client.ts
→ Can use hooks, useState, useEffect, Zustand
→ Can use browser APIs (GPS, camera etc)
→ Use for: interactive UI, forms, real-time updates
```

Rule of thumb:
```
Page that just shows data    → Server Component
Page with buttons/forms      → Client Component
Layout with nav              → Client Component (needs Zustand)
API route                    → always server
```

---

## API Routes Pattern

```typescript
// app/api/example/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'

export async function GET(req: NextRequest) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { data } = await supabase.from('...').select('*')
  return NextResponse.json({ data })
}

export async function POST(req: NextRequest) {
  const body = await req.json()
  // ...
}
```

---

## Testing Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'node',
    globals: true,
    setupFiles: ['./tests/setup.ts']
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') }
  }
})
```

```typescript
// tests/setup.ts
import { createClient } from '@supabase/supabase-js'

export const testSupabase = createClient(
  'http://localhost:54321',
  process.env.SUPABASE_SERVICE_KEY!
)
```

---

## GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22.18.0'
          cache: 'npm'
      - run: npm ci
      - uses: supabase/setup-cli@v1
      - run: supabase start
      - run: npm run test:unit
      - run: npm run test:integration
      - name: Build check
        run: npm run build
```

---

## Naming Conventions

- Files and folders: `kebab-case`
- Components: `PascalCase.tsx`
- Hooks: `use-feature-name.ts`
- Stores: `feature-store.ts`
- Types: `PascalCase` in `types/index.ts`
- DB columns: `snake_case` (matches Supabase)
- API routes: folder based `app/api/feature/route.ts`
- 'use client' goes at the very top of the file, before imports

---

## TypeScript Types

```typescript
// types/index.ts

export type Role = 'owner' | 'manager' | 'receptionist' | 'trainer' | 'member'
export type WorkspaceType = 'gym' | 'freelancer'
export type MemberStatus = 'pending' | 'active' | 'suspended'
export type ScheduleType = 'sequential' | 'calendar'
export type MissedBehaviour = 'push_forward' | 'skip_and_continue' | 'trainer_decides'
export type AttendanceMethod = 'qr_scan' | 'manual' | 'biometric' | 'gps'
export type PaymentStatus = 'pending' | 'paid' | 'failed' | 'refunded'
export type NotificationChannel = 'in_app' | 'whatsapp' | 'sms'

export interface User {
  id: string
  phone: string | null
  email: string | null
  full_name: string
  avatar_url: string | null
  created_at: string
}

export interface Workspace {
  id: string
  name: string
  type: WorkspaceType
  logo_url: string | null
  owner_id: string
  is_active: boolean
  created_at: string
}

export interface WorkspaceMember {
  id: string
  workspace_id: string
  branch_id: string | null
  user_id: string
  role: Role
  status: MemberStatus
  joined_at: string
  workspace?: Workspace
  user?: User
}

export interface Branch {
  id: string
  workspace_id: string
  name: string
  address: string | null
  qr_code: string
  is_active: boolean
  device_serial: string | null
  device_model: string | null
  latitude: number | null
  longitude: number | null
}
```
