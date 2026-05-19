# CLAUDE.md — Instructions for Claude Code

This file tells Claude Code everything it needs to know about this project.
Read this before touching any file.

---

## What This Project Is

FitDesk is a gym management SaaS for Indian gyms and freelance trainers.
Built with Nuxt 3 + Supabase + Cloudflare.

Full context in README.md.
Technical stack in STACK.md.
Exact build sequence in BUILD_ORDER.md.
Database schema in SCHEMA.md.

---

## The Single Most Important Concept

Every user has ONE account. Roles come from workspaces, not from the account.

```
user (just an identity)
└── workspace_members rows (one per context)
    ├── FitZone Gym | role: owner
    ├── Gold's Gym  | role: trainer
    └── MMA Academy | role: member
```

The active workspace + role determines what the user sees and can do.
This is stored in the Pinia workspace store.
Every feature must check the active role before rendering or allowing actions.

---

## How to Check Permissions

Always use the `usePermissions` composable. Never hardcode role checks inline.

```typescript
// Correct
const { can } = usePermissions()
if (can('workout:create')) { ... }

// Wrong — don't do this
if (activeRole.value === 'trainer' || activeRole.value === 'owner') { ... }
```

Permission matrix is in STACK.md.

---

## How to Query Supabase

Always scope queries to the active workspace. Never query without workspace_id.

```typescript
// Correct
const { data } = await supabase
  .from('workout_plans')
  .select('*')
  .eq('workspace_id', activeWorkspace.value.id)
  .eq('member_id', memberId)

// Wrong — missing workspace scope
const { data } = await supabase
  .from('workout_plans')
  .select('*')
  .eq('member_id', memberId)
```

RLS will also enforce this at database level, but always scope in the query too.

---

## File Upload Pattern

Files never go through your server. Always use presigned URLs.

```typescript
// 1. Get presigned URL
const { url, key } = await $fetch('/api/r2/presign', {
  method: 'POST',
  body: { filename: file.name, contentType: file.type }
})

// 2. Upload directly to R2
await fetch(url, { method: 'PUT', body: file })

// 3. Save key to Supabase (not the full URL)
await supabase.from('table').update({ file_url: key })
```

---

## Error Handling Pattern

Every async operation must handle errors. Never leave empty catch blocks.

```typescript
// Composable pattern
const { data, error } = await supabase.from('...').select('*')

if (error) {
  // Log for debugging
  console.error('Failed to load members:', error.message)
  // Show user-friendly message
  toast.error('Could not load members. Please try again.')
  return
}
```

---

## TypeScript Rules

- No `any` types. Use proper types from types/index.ts.
- All Supabase query results must be typed.
- All component props must be typed.
- Run `npm run typecheck` before committing.

---

## Database Rules

- Never write raw SQL in Nuxt code. Use Supabase client only.
- All schema changes go in supabase/migrations/ as numbered SQL files.
- Never modify existing migration files. Add new ones.
- Every new table needs RLS enabled and policies written.

```sql
-- Every new table needs this
alter table your_table enable row level security;

-- Then add policies
create policy "policy name"
on your_table for select
using ( ... );
```

---

## Testing Rules

- Write the test before or alongside the feature. Never after.
- Every new composable needs unit tests.
- Every new table needs RLS integration tests.
- Tests live in tests/ matching the source structure.

---

## What Phase We Are On

Check BUILD_ORDER.md and look for the current phase.
Only build what is in the current phase.
Do not jump ahead.

---

## Commit Message Format

```
feat: add member invite via phone number
fix: correct GPS distance calculation
test: add RLS tests for workout plans
chore: update supabase types
```

---

## When Adding a New Feature

1. Check BUILD_ORDER.md — is this in the current phase?
2. Check SCHEMA.md — which tables does this touch?
3. Check STACK.md — how do you call that service?
4. Write migration if new tables needed
5. Write RLS policies
6. Write the composable
7. Write tests
8. Build the UI
9. Verify TypeScript zero errors
10. Verify tests pass

---

## Things to Never Do

- Never put SUPABASE_SERVICE_KEY in client-side code
- Never query without workspace_id scope
- Never skip RLS policies on new tables
- Never use `any` type
- Never commit .env.local
- Never modify existing migration files
- Never build phase N+1 before phase N is verified
