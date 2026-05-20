# FitDesk — Build Order

This is the exact sequence to build FitDesk.
Each phase produces something working and shippable.
Do not skip phases. Do not build phase 2 before phase 1 is done.

---

## Phase 0 — Project Setup (Day 1) ✅ DONE

```
✅ Next.js 15 project created
✅ Deployed to Cloudflare Workers & Pages
✅ GitHub repo connected — auto-deploys on push
✅ Supabase project created on supabase.com
```

### Remaining Phase 0 steps

```bash
# Install all dependencies
npm install @supabase/supabase-js @supabase/ssr
npm install zustand
npm install date-fns zod

# shadcn/ui
npx shadcn@latest init
# When asked: Style → Default, Base color → Neutral, CSS variables → Yes

# Dev dependencies
npm install -D vitest @playwright/test @vitejs/plugin-react
npm install -D wrangler

# Create environment files
# .env.local (never commit)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_KEY=eyJ...

# .env.example (commit this — no real values)
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_KEY=your_service_key

# Add to .gitignore
.env.local
```

```bash
# Create the Supabase folder structure
mkdir -p supabase/migrations
mkdir -p supabase/functions/webhook-zkteco
mkdir -p supabase/functions/send-notification

# Create lib/supabase files (see STACK.md for content)
mkdir -p lib/supabase
touch lib/supabase/client.ts
touch lib/supabase/server.ts

# Create types file
mkdir -p types
touch types/index.ts

# Create hooks folder
mkdir -p hooks

# Create stores folder
mkdir -p stores
```

### Verification
- [ ] `npm run dev` starts without errors
- [ ] Cloudflare deploys on push (already working)
- [ ] `.env.local` created with real Supabase keys
- [ ] `lib/supabase/client.ts` and `server.ts` created (see STACK.md)
- [ ] TypeScript has zero errors
- [ ] `.env.local` is in `.gitignore`

---

## Phase 1 — Auth + Workspace (Week 1-2)

This is the foundation everything else sits on.
No other feature can be built before this is solid.

### What to build

```
1. Database migrations (run in Supabase SQL editor or via CLI)
   - users table
   - workspaces table
   - workspace_members table
   - branches table (basic, no ZKTeco columns yet)
   - RLS policies for all four tables
   → All SQL is in SCHEMA.md migration 0001

2. Supabase client files
   - lib/supabase/client.ts  → browser client (see STACK.md)
   - lib/supabase/server.ts  → server client (see STACK.md)

3. TypeScript types
   - types/index.ts → all types (see STACK.md)

4. Zustand store
   - stores/workspace-store.ts (see STACK.md)

5. Next.js middleware (root middleware.ts)
   - Refreshes Supabase session on every request
   - Redirects unauthenticated users away from /dashboard/*
   → See STACK.md for exact middleware code

6. Auth screens
   - app/(auth)/login/page.tsx → phone or email input
   - app/(auth)/verify/page.tsx → OTP input
   - Uses Supabase Auth OTP flow

7. Onboarding flow
   - app/(onboarding)/onboarding/page.tsx
     → "What do you want to do?"
     → Options: Create a gym | Start freelancing | Join a gym
   - app/(onboarding)/create-gym/page.tsx
     → gym name, logo upload
   - app/(onboarding)/join-gym/page.tsx
     → enter gym code

8. Workspace switcher
   - app/(app)/workspace/switch/page.tsx
     → list all workspaces with role badge
     → tap to switch → updates Zustand store
     → active workspace persisted via Zustand persist middleware

9. Basic dashboard shell
   - app/(app)/layout.tsx → authenticated layout with nav
   - app/(app)/dashboard/page.tsx → role-aware dashboard
   - Role-aware nav items (owner sees different tabs than member)
   - Empty states for everything (filled in later phases)

10. Hooks
    - hooks/use-workspace.ts → read active workspace + role
    - hooks/use-permissions.ts → can() function (see STACK.md)
    - hooks/use-auth.ts → current user, sign in, sign out
```

### Route structure for Phase 1

```
app/
├── (auth)/
│   ├── login/page.tsx          → /login
│   └── verify/page.tsx         → /verify
├── (onboarding)/
│   ├── onboarding/page.tsx     → /onboarding
│   ├── create-gym/page.tsx     → /create-gym
│   └── join-gym/page.tsx       → /join-gym
├── (app)/
│   ├── layout.tsx              → authenticated shell
│   ├── dashboard/page.tsx      → /dashboard
│   └── workspace/
│       └── switch/page.tsx     → /workspace/switch
├── layout.tsx                  → root layout
├── page.tsx                    → / landing (placeholder for now)
└── globals.css
middleware.ts                   → root level, not inside app/
```

### Tests to write (Phase 1)

```
tests/unit/permissions.test.ts
- usePermissions can() returns true for correct roles
- usePermissions can() returns false for wrong roles

tests/unit/workspace-store.test.ts
- setActiveWorkspace updates store correctly
- persist saves to localStorage

tests/integration/rls-workspace.test.ts
- Owner can see their workspace
- Member cannot see another member's workspace_member row
- User with no workspaces gets empty array not error
```

### Verification
- [ ] User can sign up with phone OTP
- [ ] User can sign up with email OTP
- [ ] User can create a gym → lands in /dashboard
- [ ] User can start freelancing → lands in /dashboard
- [ ] User can switch between workspaces
- [ ] Role badge shows correctly per workspace
- [ ] Unauthenticated user redirected to /login
- [ ] User with no workspace redirected to /onboarding
- [ ] TypeScript zero errors
- [ ] Build succeeds

---

## Phase 2 — Member Module (Week 3-4)

Gym owner and trainer can add and manage members.
Members can join via invite or gym code.

### What to build

```
1. Database migrations
   - workspace_members status column already in migration 0001
   - Add unique index: workspace_members(workspace_id, user_id, role)
   → See SCHEMA.md migration 0001

2. Member list page
   app/(app)/members/page.tsx → /members
   - Shows all members in active workspace
   - Filter by: All | No plan | Fee due | Inactive
   - Search by name
   - Status badge per member
   - Role check: only owner/manager/receptionist/trainer can see this
   - Members see "Access denied"

3. Add member flow
   - Enter phone number → sends invite (WhatsApp/SMS in v2, for now email)
   - Member receives link → signs up → auto-joined to workspace
   - Member status: active immediately

4. Join via gym code flow
   - Member enters gym code → pending request created
   - Admin sees pending list → approves or rejects
   - Member status: pending → active after approval

5. Member profile page
   app/(app)/members/[id]/page.tsx → /members/123
   - Basic info: name, phone, joined date, trainer assigned
   - Assign trainer dropdown
   - Suspend / reactivate member
   - Trainer can only see their own assigned members

6. Invite link generation
   - Each branch has unique qr_code (already in branches table)
   - Invite URL: /join?code=xxx
   - app/join/page.tsx → reads code, creates pending membership
```

### Tests to write (Phase 2)

```
tests/unit/member.test.ts
- generateGymCode produces unique codes
- invite link resolves to correct workspace

tests/integration/rls-member.test.ts
- Trainer sees only their assigned members
- Receptionist can create workspace_member rows
- Receptionist cannot update workout_plans
- Member cannot see other members

tests/e2e/member-flow.spec.ts
- Owner adds member → member receives invite
- Member joins via gym code → admin approves → member sees dashboard
```

### Verification
- [ ] Owner can add a member
- [ ] Member receives invite link
- [ ] Member clicks link → signs up → lands in gym workspace
- [ ] Member self-registers with gym code → pending → approved → active
- [ ] Trainer sees only their assigned members
- [ ] Receptionist can add members but cannot edit workout plans

---

## Phase 3 — Workout Module (Week 5-6)

Trainers assign workout plans. Members see today's workout.

### What to build

```
1. Database migrations → SCHEMA.md migration 0003
   - exercise_library
   - workout_plans
   - workout_weeks
   - workout_sessions
   - workout_exercises
   - workout_session_overrides
   - workout_exercise_overrides
   - workout_session_skips
   - workout_logs
   - workout_set_logs

2. Exercise library
   app/(app)/workout/library/page.tsx
   - List platform + workspace exercises
   - Upload: name, muscle group, equipment, video → R2

3. Create workout plan
   app/(app)/workout/new/page.tsx
   - Select member
   - Sequential or Calendar
   - missed_session_behaviour
   - total_weeks (or null = forever)
   - Add sessions with exercises

4. Member workout view
   app/(app)/workout/page.tsx
   - Sequential: shows session at current_session_index
   - Calendar: shows session for today's day of week
   - Exercise list with sets/reps/weight
   - Video play button
   - Mark complete → advances index

5. Missed session logic
   supabase/functions/missed-sessions/index.ts
   - Supabase cron at midnight
   - skip_and_continue plans → auto-advance
   - Records skip in workout_session_skips

6. Session override
   - Trainer opens member → today's session → Override
   - Swap exercises for today only
   - Template untouched
```

### Tests to write (Phase 3)

```
tests/unit/workout-logic.test.ts
- getNextSession sequential: correct index returned
- getNextSession calendar: correct day returned
- push_forward: missed session shown next visit
- skip_and_continue: sequence advances past missed
- trainer_decides: notification created
- override takes priority over template

tests/integration/rls-workout.test.ts
- Trainer can only assign plans to their members
- Member can only view their own plans
- Owner can view all plans in workspace

tests/e2e/trainer-flow.spec.ts
- Trainer creates plan → member sees it
- Member marks complete → next session shown
- Override shown to member, template unchanged
```

### Verification
- [ ] Trainer creates sequential plan
- [ ] Trainer creates calendar plan
- [ ] Member sees correct session today
- [ ] Mark complete advances the plan
- [ ] Skip recorded when member absent
- [ ] Override works for one day only
- [ ] Video plays from R2

---

## Phase 4 — Diet Module (Week 7)

### What to build

```
1. Database migrations → SCHEMA.md migration 0004
   - diet_plans, diet_meals, diet_items

2. Create diet plan (trainer)
   app/(app)/diet/new/page.tsx
   - Plan name, total calories
   - Add meals with time labels
   - Add items: name, quantity, macros, optional photo

3. Member diet view
   app/(app)/diet/page.tsx
   - Today's meals
   - Calorie progress bar
   - Filter by meal
   - Macros per item
   - Food photo if attached
```

### Verification
- [ ] Trainer creates diet plan
- [ ] Member sees today's meals
- [ ] Calorie total correct
- [ ] Macros shown
- [ ] Food photos load from R2

---

## Phase 5 — Attendance Module (Week 8)

### What to build

```
1. Database migrations → SCHEMA.md migration 0005
   - attendance table with unique index

2. GPS check-in
   app/(app)/attendance/page.tsx (client component)
   - navigator.geolocation.getCurrentPosition()
   - Calculate distance from branch lat/lng
   - Within 50m → log attendance, method: gps
   - Outside → error message

3. QR check-in
   - Member scans gym QR → reads branch qr_code
   - Validates + logs attendance, method: qr_scan
   - With GPS combined: both must pass

4. Manual (trainer/receptionist)
   - Member list → tap → Mark attendance
   - method: manual, marked_by: staff user id

5. Attendance history (member)
   - Weekly strip M T W T F S S
   - Streak counter

6. Attendance dashboard (trainer/owner)
   - Today's check-ins live
   - Inactive members list
```

### Verification
- [ ] GPS check-in within 50m works
- [ ] GPS check-in outside 50m rejected
- [ ] QR scannable and logs attendance
- [ ] Duplicate check-in same day rejected
- [ ] Streak calculated correctly
- [ ] Trainer sees today's attendance live

---

## Phase 6 — Progress + Payment (Week 9-10)

### What to build

```
1. Database migrations → SCHEMA.md migration 0005
   - member_progress (user_id scoped)
   - progress_photos (workspace_member_id scoped)
   - membership_plans
   - member_subscriptions
   - member_payments

2. Progress logging (member)
   app/(app)/progress/page.tsx
   - Weight, measurements form
   - Photo upload (front/back/side) → R2
   - Notes field
   - One entry per day

3. Progress history (trainer)
   - Weight chart (use recharts or chart.js)
   - Measurement table
   - Photo timeline

4. Membership plans (owner)
   app/(app)/settings/plans/page.tsx
   - Create Monthly/Quarterly/Annual plans

5. Fee tracking
   app/(app)/members/[id]/subscription/page.tsx
   - Assign plan → auto calc end date
   - Mark paid manually
   - Member uploads screenshot → R2
   - Overdue shown in red on member list
```

### Verification
- [ ] Member logs weight → chart updates
- [ ] Photos upload to R2
- [ ] Owner creates plans
- [ ] Receptionist assigns and marks paid
- [ ] Fee overdue shows in red

---

## Phase 7 — Notifications (Week 11)

### What to build

```
1. Database migrations → SCHEMA.md migration 0006
   - notifications table

2. In-app notifications
   - Bell icon with unread badge in nav
   - app/(app)/notifications/page.tsx
   - Mark as read

3. Create notifications on these events:
   - Member enrolled
   - Plan assigned
   - Fee due in 3 days (cron edge function)
   - Fee overdue (cron edge function)
   - Payment confirmed
   - Member inactive 7 days

4. WhatsApp via Gupshup → v2
   - Rows already created in notifications table
   - Just wire up external_sent when Gupshup added
```

### Verification
- [ ] Notification appears when plan assigned
- [ ] Fee reminder at 3 days before due
- [ ] Trainer notified when member inactive 7 days

---

## Phase 8 — Polish + Launch Prep (Week 12)

### What to build

```
1. Empty states — every page
   "Add your first member" / "No plan assigned yet" etc

2. Error handling — every fetch
   Network error → retry button
   Permission error → friendly message
   404 → clean not found page

3. Loading states — skeletons on every page
   No flash of empty content

4. PWA
   next.config.ts → PWA config
   public/manifest.json
   Service worker caches today's workout + diet

5. Landing page
   app/page.tsx → real landing page
   - What is FitDesk, who it's for, pricing, CTA

6. Run full test suite
   npm run test:unit
   npm run test:integration
   npm run test:e2e
```

### Verification
- [ ] Every page has empty state
- [ ] Every error handled gracefully
- [ ] PWA installable from mobile
- [ ] Offline shows cached workout
- [ ] Landing page live
- [ ] All tests green

---

## After Launch — v2 Queue

- [ ] ZKTeco hardware integration (Phase 9)
- [ ] Razorpay subscription billing (Phase 10)
- [ ] WhatsApp via Gupshup (Phase 11)
- [ ] Capacitor mobile app (Phase 12)
- [ ] Dealer support map (Phase 13)
- [ ] Gym public landing pages (Phase 14)

---

## Daily Development Workflow

```bash
# Start
npm run dev

# Before committing
npx tsc --noEmit        # zero TypeScript errors
npm run test:unit       # unit tests pass

# Before merging to main
npm run test:integration
npm run build
```

---

## When Stuck

1. Check SCHEMA.md — which table does this touch?
2. Check STACK.md — how do I call Supabase? Server or client component?
3. Check CLAUDE.md — am I breaking any rules?
4. Write the test first — it defines what "working" means

---

## Definition of Done

A feature is done when:
- [ ] Works in browser
- [ ] Unit test written and passing
- [ ] RLS test written and passing
- [ ] Empty state handled
- [ ] Error state handled
- [ ] TypeScript zero errors
- [ ] Build succeeds
