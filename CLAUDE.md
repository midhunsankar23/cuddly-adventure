# CLAUDE.md — Instructions for Claude Code

Read this before touching any file in this project.

---

## What This Project Is

FitDesk is a gym management SaaS for Indian gyms and freelance trainers.
Next.js 15 + Supabase + Cloudflare Workers & Pages.

Full context: README.md
Technical stack: STACK.md
Build sequence: BUILD_ORDER.md
Database schema: SCHEMA.md

---

## The Single Most Important Concept

Every user has ONE account. Roles come from workspaces, not the account.

```
user (just an identity)
└── workspace_members rows (one per context)
    ├── FitZone Gym | role: owner
    ├── Gold's Gym  | role: trainer
    └── MMA Academy | role: member
```

Active workspace + role determines what the user sees and can do.
Stored in Zustand workspace-store.ts, persisted to localStorage.
Every feature must check active role before rendering or allowing actions.

---

## Server vs Client Components — Most Important Next.js Rule

```
No directive at top = Server Component
'use client' at top = Client Component
```

Server Component:
- Use lib/supabase/server.ts
- Cannot use hooks, useState, browser APIs
- Good for: pages that fetch and display data

Client Component:
- Use lib/supabase/client.ts
- Can use hooks, useState, Zustand, browser APIs
- Good for: forms, interactive UI, real-time, GPS, camera

When in doubt: start with Server Component. Add 'use client' only when you need interactivity.

---

## How to Check Permissions

Always use the usePermissions hook. Never hardcode role checks.

```typescript
// Correct
'use client'
import { usePermissions } from '@/hooks/use-permissions'

const { can } = usePermissions()
if (can('workout:create')) { ... }

// Wrong
if (activeRole === 'trainer' || activeRole === 'owner') { ... }
```

Permission matrix is in STACK.md.

---

## How to Query Supabase

Always scope queries to active workspace. Never query without workspace_id.

```typescript
// In a Server Component or API route
import { createClient } from '@/lib/supabase/server'

const supabase = await createClient()
const { data } = await supabase
  .from('workout_plans')
  .select('*')
  .eq('workspace_id', workspaceId)  // always scope
  .eq('member_id', memberId)

// In a Client Component
'use client'
import { createClient } from '@/lib/supabase/client'

const supabase = createClient()
const { data } = await supabase
  .from('workout_plans')
  .select('*')
  .eq('workspace_id', workspaceId)  // always scope
```

---

## File Upload Pattern

Files never go through your server. Always use presigned URLs.

```typescript
// 1. Get presigned URL from API route
const res = await fetch('/api/r2/presign', {
  method: 'POST',
  body: JSON.stringify({ filename: file.name, contentType: file.type }),
  headers: { 'Content-Type': 'application/json' }
})
const { url, key } = await res.json()

// 2. Upload directly to R2
await fetch(url, { method: 'PUT', body: file })

// 3. Save key to Supabase (not the full URL)
await supabase.from('table').update({ file_url: key })
```

---

## Error Handling Pattern

Every async operation must handle errors. Never empty catch blocks.

```typescript
const { data, error } = await supabase.from('...').select('*')

if (error) {
  console.error('Operation failed:', error.message)
  // show user-friendly error in UI
  return
}
```

---

## API Route Pattern

```typescript
// app/api/feature/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'

export async function POST(req: NextRequest) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // do work
  return NextResponse.json({ data })
}
```

---

## TypeScript Rules

- No any types. Use proper types from types/index.ts
- All Supabase query results must be typed
- All component props must have an interface defined
- Run tsc --noEmit to check before committing

---

## Database Rules

- Never write raw SQL in Next.js code. Use Supabase client only.
- All schema changes go in supabase/migrations/ as numbered SQL files
- Never modify existing migration files — add new ones
- Every new table needs RLS enabled and policies written

```sql
alter table your_table enable row level security;

create policy "policy name"
on your_table for select
using ( ... );
```

---

## Testing Rules

- Write tests alongside the feature, not after
- Every new hook needs unit tests
- Every new table needs RLS integration tests
- Tests live in tests/ matching the source structure

---

## Current Phase

Check BUILD_ORDER.md. Find the current phase.
Only build what is in the current phase.
Do not jump ahead to the next phase.

---

## Commit Message Format

```
feat: add member invite via phone
fix: correct GPS distance calculation
test: add RLS tests for workout plans
chore: update supabase types
```

---

## Things to Never Do

- Never put SUPABASE_SERVICE_KEY in client-side code or NEXT_PUBLIC_ variables
- Never query Supabase without workspace_id scope
- Never skip RLS policies on new tables
- Never use any type
- Never commit .env.local
- Never modify existing migration files
- Never use 'use client' unless you actually need browser features
- Never mix server and client Supabase clients
