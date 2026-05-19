# FitDesk — Stack Reference

## Core Stack

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Framework | Nuxt 3 | Latest | Vue 3, Nitro, file-based routing |
| Hosting | Cloudflare Pages | — | Auto-deploy from GitHub |
| Database | Supabase Postgres | — | Full Postgres + RLS |
| Auth | Supabase Auth | — | OTP via phone/email, JWT |
| Realtime | Supabase Realtime | — | Live updates via websocket |
| File storage | Cloudflare R2 | — | Videos, photos, screenshots |
| Edge functions | Supabase Edge Functions | — | Webhooks, background logic |
| Styling | Tailwind CSS + shadcn-vue | Latest | Utility-first + components |
| State | Pinia | Latest | Vue store |
| Testing (unit) | Vitest | Latest | — |
| Testing (e2e) | Playwright | Latest | — |
| CI/CD | GitHub Actions | — | Runs on every push |
| Mobile | Capacitor | Latest | Added in v2 |
| Notifications | Gupshup | — | WhatsApp + SMS, added in v2 |
| Payments | Manual v1, Razorpay v2 | — | Provider-agnostic schema |

---

## Nuxt Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages'
  },
  modules: [
    '@nuxtjs/supabase',
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    'shadcn-nuxt',
    '@vite-pwa/nuxt',
    '@nuxtjs/color-mode'
  ],
  supabase: {
    redirect: false   // we handle auth redirects manually
  },
  colorMode: {
    classSuffix: ''
  }
})
```

---

## Environment Variables

```bash
# .env.local (never commit this)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
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
SUPABASE_URL=your_supabase_url
SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_KEY=your_service_key
CLOUDFLARE_ACCOUNT_ID=your_account_id
CLOUDFLARE_R2_ACCESS_KEY=your_r2_access_key
CLOUDFLARE_R2_SECRET_KEY=your_r2_secret_key
CLOUDFLARE_R2_BUCKET=fitdesk-uploads
```

---

## Folder Structure

```
fitdesk/
├── .github/
│   └── workflows/
│       └── ci.yml              # runs tests on every push
├── app/
│   ├── components/
│   │   ├── ui/                 # shadcn-vue components
│   │   ├── workspace/          # workspace switcher, cards
│   │   ├── member/             # member list, card, form
│   │   ├── workout/            # plan, session, exercise components
│   │   ├── diet/               # meal, item components
│   │   ├── attendance/         # QR, GPS, history
│   │   └── shared/             # buttons, inputs, badges
│   ├── composables/
│   │   ├── useWorkspace.ts     # active workspace, role checks
│   │   ├── useAuth.ts          # current user, sign in/out
│   │   ├── useMember.ts        # member CRUD
│   │   ├── useWorkout.ts       # plan logic, session advancement
│   │   ├── useAttendance.ts    # check-in, GPS, QR
│   │   └── usePermissions.ts   # role-based permission checks
│   ├── layouts/
│   │   ├── default.vue         # authenticated layout with nav
│   │   ├── auth.vue            # login/signup layout
│   │   └── public.vue          # landing page layout
│   ├── middleware/
│   │   ├── auth.ts             # redirect if not logged in
│   │   └── workspace.ts        # redirect if no workspace selected
│   └── pages/
│       ├── index.vue           # landing page
│       ├── auth/
│       │   ├── login.vue
│       │   └── verify.vue
│       ├── onboarding/
│       │   ├── index.vue       # what do you want to do?
│       │   ├── create-gym.vue
│       │   └── join-gym.vue
│       ├── dashboard/
│       │   └── index.vue       # role-aware dashboard
│       ├── members/
│       │   ├── index.vue
│       │   └── [id].vue
│       ├── workout/
│       │   ├── index.vue
│       │   └── [planId].vue
│       ├── diet/
│       │   └── index.vue
│       ├── attendance/
│       │   └── index.vue
│       ├── settings/
│       │   ├── gym.vue
│       │   ├── billing.vue
│       │   └── hardware.vue
│       └── workspace/
│           └── switch.vue
├── server/
│   └── api/                    # Nuxt server routes (thin layer)
│       ├── webhooks/
│       │   └── zkteco.post.ts
│       └── r2/
│           └── presign.post.ts
├── supabase/
│   ├── migrations/             # all schema migrations in order
│   │   ├── 0001_initial.sql
│   │   ├── 0002_rls.sql
│   │   └── 0003_seed.sql
│   ├── functions/              # edge functions
│   │   ├── webhook-zkteco/
│   │   │   └── index.ts
│   │   └── send-notification/
│   │       └── index.ts
│   └── seed.sql                # subscription plans seed data
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
├── STACK.md                    # this file
├── SCHEMA.md                   # database schema reference
├── DEVLOG.md                   # decisions made during development
├── .env.example
├── .env.local                  # gitignored
├── nuxt.config.ts
├── tailwind.config.ts
├── wrangler.toml               # Cloudflare config
└── package.json
```

---

## Key Packages

```json
{
  "dependencies": {
    "nuxt": "^3.x",
    "@nuxtjs/supabase": "^1.x",
    "@supabase/supabase-js": "^2.x",
    "@pinia/nuxt": "^0.x",
    "pinia": "^2.x",
    "@nuxtjs/tailwindcss": "^6.x",
    "shadcn-nuxt": "^0.x",
    "@vite-pwa/nuxt": "^0.x",
    "@nuxtjs/color-mode": "^3.x",
    "zod": "^3.x",
    "date-fns": "^3.x"
  },
  "devDependencies": {
    "vitest": "^1.x",
    "@playwright/test": "^1.x",
    "typescript": "^5.x",
    "@types/node": "^20.x",
    "wrangler": "^3.x"
  }
}
```

---

## Supabase Client Setup

```typescript
// composables/useSupabase.ts
// Provided automatically by @nuxtjs/supabase
// Use via:
const supabase = useSupabaseClient()
const user = useSupabaseUser()
```

---

## Cloudflare R2 Upload Pattern

All file uploads go directly to R2 via presigned URLs.
Files never pass through your server.

```typescript
// 1. Client requests presigned URL from your server
const { url, key } = await $fetch('/api/r2/presign', {
  method: 'POST',
  body: { filename: file.name, contentType: file.type }
})

// 2. Client uploads directly to R2
await fetch(url, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type }
})

// 3. Save the key to Supabase (not the full URL)
await supabase.from('exercise_library').update({ video_url: key })
```

---

## Role Permission Matrix

```typescript
// composables/usePermissions.ts

const PERMISSIONS = {
  // Member management
  'member:create':     ['owner', 'manager', 'receptionist'],
  'member:view':       ['owner', 'manager', 'receptionist', 'trainer'],
  'member:edit':       ['owner', 'manager', 'receptionist'],
  'member:suspend':    ['owner', 'manager'],

  // Workout plans
  'workout:create':    ['owner', 'trainer'],
  'workout:view':      ['owner', 'manager', 'trainer'],
  'workout:edit':      ['owner', 'trainer'],

  // Diet plans
  'diet:create':       ['owner', 'trainer'],
  'diet:view':         ['owner', 'manager', 'trainer'],

  // Attendance
  'attendance:mark':   ['owner', 'manager', 'receptionist', 'trainer'],
  'attendance:view':   ['owner', 'manager', 'receptionist', 'trainer'],

  // Payments
  'payment:mark':      ['owner', 'manager', 'receptionist'],
  'payment:view':      ['owner', 'manager', 'receptionist'],

  // Billing (your SaaS subscription)
  'billing:manage':    ['owner'],

  // Branch management
  'branch:manage':     ['owner'],

  // Hardware
  'hardware:manage':   ['owner', 'manager'],

  // Videos
  'video:upload':      ['owner', 'trainer'],
  'video:view':        ['owner', 'manager', 'trainer', 'member'],
} as const

export function usePermissions() {
  const { activeRole } = useWorkspace()

  function can(permission: keyof typeof PERMISSIONS): boolean {
    return PERMISSIONS[permission].includes(activeRole.value as any)
  }

  return { can }
}
```

---

## Workspace Store

```typescript
// stores/workspace.ts
export const useWorkspaceStore = defineStore('workspace', () => {
  const activeWorkspace = ref<Workspace | null>(null)
  const activeRole = ref<Role | null>(null)
  const workspaces = ref<WorkspaceMember[]>([])

  async function switchWorkspace(workspaceId: string) {
    const wm = workspaces.value.find(w => w.workspace_id === workspaceId)
    if (!wm) return
    activeWorkspace.value = wm.workspace
    activeRole.value = wm.role
    // persist to localStorage so it survives refresh
    localStorage.setItem('activeWorkspaceId', workspaceId)
  }

  async function loadWorkspaces(userId: string) {
    const supabase = useSupabaseClient()
    const { data } = await supabase
      .from('workspace_members')
      .select('*, workspace:workspaces(*)')
      .eq('user_id', userId)
      .eq('status', 'active')
    workspaces.value = data ?? []
  }

  return { activeWorkspace, activeRole, workspaces, switchWorkspace, loadWorkspaces }
})
```

---

## Auth Flow

```
User lands on /
→ Not logged in → redirect to /auth/login
→ Enter phone number → OTP sent
→ /auth/verify → enter OTP
→ Supabase creates session
→ Check workspaces:
   0 workspaces → /onboarding
   1 workspace  → /dashboard (auto-select)
   2+ workspaces → /workspace/switch (or last used)
```

---

## Testing Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
    setupFiles: ['./tests/setup.ts']
  }
})
```

```typescript
// tests/setup.ts
import { createClient } from '@supabase/supabase-js'

// Uses local Supabase instance for tests
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
    
    services:
      supabase:
        image: supabase/postgres:15
        
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx supabase start
      - run: npm run test:unit
      - run: npm run test:integration
      - name: Build check
        run: npm run build
```

---

## Wrangler Config

```toml
# wrangler.toml
name = "fitdesk"
compatibility_date = "2024-01-01"
pages_build_output_dir = ".output/public"

[[r2_buckets]]
binding = "FITDESK_UPLOADS"
bucket_name = "fitdesk-uploads"
```

---

## Naming Conventions

- Files: `kebab-case.ts` / `kebab-case.vue`
- Components: `PascalCase`
- Composables: `useFeatureName.ts`
- Stores: `useFeatureStore` in `stores/feature.ts`
- Types: `PascalCase` in `types/index.ts`
- DB columns: `snake_case` (matches Supabase)
- API routes: `kebab-case` matching Nuxt file-based routing

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
}
```
