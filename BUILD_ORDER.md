# FitDesk — Build Order

This is the exact sequence to build FitDesk.
Each phase produces something working and shippable.
Do not skip phases. Do not build phase 2 before phase 1 is done.

---

## Phase 0 — Project Setup (Day 1)

Do this once. Get the skeleton running before writing any feature code.

### Steps

```bash
# 1. Create Nuxt project
npx nuxi@latest init fitdesk
cd fitdesk

# 2. Install all dependencies
npm install @nuxtjs/supabase @supabase/supabase-js
npm install @pinia/nuxt pinia
npm install @nuxtjs/tailwindcss
npm install shadcn-nuxt
npm install @vite-pwa/nuxt
npm install @nuxtjs/color-mode
npm install zod date-fns

npm install -D vitest @playwright/test
npm install -D wrangler typescript @types/node

# 3. Configure nuxt.config.ts (see STACK.md)

# 4. Set up Supabase local
npx supabase init
npx supabase start

# 5. Set up Cloudflare Pages
# Connect GitHub repo in Cloudflare dashboard
# Set build command: npm run build
# Set output dir: .output/public

# 6. Create .env.local with real values
# Create .env.example with placeholder values
# Add .env.local to .gitignore

# 7. Create GitHub repo
# Push initial commit
# Verify Cloudflare auto-deploys
```

### Verification
- [ ] `npm run dev` starts without errors
- [ ] Cloudflare Pages deploys on push
- [ ] Supabase local runs on localhost:54321
- [ ] TypeScript has zero errors

---

## Phase 1 — Auth + Workspace (Week 1-2)

This is the foundation everything else sits on.
No other feature can be built before this is solid.

### What to build

```
1. Database migrations
   - users table
   - workspaces table
   - workspace_members table
   - branches table (basic, no ZKTeco columns yet)
   - RLS policies for all four tables

2. Auth screens
   - /auth/login → phone or email input
   - /auth/verify → OTP input
   - Supabase Auth OTP flow

3. Onboarding flow
   - /onboarding → "What do you want to do?"
     Options: Create a gym | Start freelancing | Join a gym
   - /onboarding/create-gym → gym name, logo upload
   - /onboarding/join-gym → enter gym code
   - After any option → land in dashboard

4. Workspace switcher
   - /workspace/switch → list all workspaces with role badge
   - Switch updates Pinia store → entire UI reflects new context
   - Active workspace persisted in localStorage

5. Middleware
   - auth.ts → redirect to /auth/login if no session
   - workspace.ts → redirect to /workspace/switch if no active workspace

6. Basic dashboard shell
   - Layout with sidebar/bottom nav
   - Role-aware nav items (owner sees different tabs than member)
   - "Good morning, Name" header
   - Empty states for everything (will be filled in later phases)
```

### Tests to write (Phase 1)
```
Unit:
- useWorkspace returns correct role
- switchWorkspace updates store correctly
- middleware redirects work

Integration (RLS):
- Owner can see their workspace
- Member cannot see another member's workspace_member row
- User with no workspaces gets empty array not error
```

### Verification
- [ ] User can sign up with phone OTP
- [ ] User can create a gym → lands in gym dashboard
- [ ] User can create freelancer profile → lands in freelancer dashboard
- [ ] User can switch between workspaces
- [ ] Role badge shows correctly per workspace
- [ ] Logged out user redirected to /auth/login
- [ ] User with no workspace redirected to /onboarding

---

## Phase 2 — Member Module (Week 3-4)

Gym owner and trainer can add and manage members.
Members can join via invite or gym code.

### What to build

```
1. Database migrations
   - workspace_members extended (status column)
   - Add unique index: workspace_members(workspace_id, user_id, role)

2. Member list page (/members)
   - Shows all members in active workspace
   - Filter by: All | No plan | Fee due | Inactive
   - Search by name
   - Status badge per member (on track / needs plan / fee due / inactive)
   - Role check: only owner/manager/receptionist/trainer can see this

3. Add member flow
   - Enter phone number → sends WhatsApp/SMS invite
   - Member receives link → signs up → auto-joined to workspace
   - Member status: active immediately

4. Join via gym code flow
   - Member enters gym code → pending request created
   - Admin sees pending list → approves or rejects
   - Member status: pending → active after approval

5. Member profile page (/members/[id])
   - Basic info tab: name, phone, joined date, trainer assigned
   - Assign trainer (dropdown of gym's trainers)
   - Suspend / reactivate member
   - Role check: trainer can only see their own assigned members

6. Invite link generation
   - Each gym has a unique invite link
   - Clicking link → /join?code=xxx → auto-populates gym code
```

### Tests to write (Phase 2)
```
Unit:
- generateGymCode produces unique codes
- Member invite link resolves to correct gym

Integration (RLS):
- Trainer can only see members assigned to them
- Receptionist can create workspace_member rows
- Receptionist cannot update workout_plans
- Member cannot see other members

E2E:
- Owner adds member via phone → member receives invite
- Member joins via gym code → admin approves → member sees dashboard
```

### Verification
- [ ] Owner can add a member
- [ ] Member receives invite (WhatsApp or SMS link)
- [ ] Member clicks link → signs up → lands in gym workspace
- [ ] Member self-registers with gym code → pending → approved → active
- [ ] Trainer sees only their assigned members
- [ ] Receptionist can add members but not edit workout plans

---

## Phase 3 — Workout Module (Week 5-6)

Trainers assign workout plans. Members see today's workout.

### What to build

```
1. Database migrations
   - exercise_library table
   - workout_plans table
   - workout_weeks table
   - workout_sessions table
   - workout_exercises table
   - workout_session_overrides table
   - workout_exercise_overrides table
   - workout_session_skips table
   - workout_logs table
   - workout_set_logs table
   - RLS for all workout tables

2. Exercise library (/workout/library)
   - List all exercises (platform + trainer's own)
   - Upload new exercise (name, muscle group, equipment, video)
   - Video upload → R2 presigned URL
   - Search and filter by muscle group

3. Create workout plan flow
   - Trainer selects a member
   - Chooses: Sequential or Calendar
   - Sets missed_session_behaviour
   - Sets total_weeks (or null for forever)
   - Adds sessions (label, exercises, sets/reps/weight)
   - Each exercise can attach a video from library

4. Member workout view (/workout)
   - Shows today's session based on schedule_type
   - Sequential: reads current_session_index → shows that session
   - Calendar: reads day of week → shows that session
   - Exercise list with sets/reps/weight
   - Play button if video attached
   - "Mark complete" button → advances index

5. Missed session logic
   - Cron job (Supabase Edge Function) runs at midnight
   - Checks members with skip_and_continue plans
   - Advances index for absent members
   - Records skip in workout_session_skips

6. Session override flow (trainer)
   - Trainer opens member profile → Today's session
   - Tap "Override today" → swap exercises for this date only
   - Template untouched
```

### Tests to write (Phase 3)
```
Unit:
- getNextSession returns correct index for sequential plans
- getNextSession returns correct day for calendar plans
- missed_session_behaviour logic (all three modes)
- Session override takes priority over template

Integration (RLS):
- Trainer can only assign plans to their members
- Member can only view their own plans
- Owner can view all plans in their workspace

E2E:
- Trainer creates plan → member sees it immediately
- Member marks session complete → next session shown
- Trainer overrides today → member sees override, not template
```

### Verification
- [ ] Trainer can create a sequential workout plan
- [ ] Trainer can create a calendar-based plan
- [ ] Member sees correct session today
- [ ] Mark complete advances the plan
- [ ] Skip recorded when member doesn't come
- [ ] Override works for one day without affecting template
- [ ] Video plays from R2

---

## Phase 4 — Diet Module (Week 7)

Trainers assign structured diet plans. Members see today's meals.

### What to build

```
1. Database migrations
   - diet_plans table
   - diet_meals table
   - diet_items table
   - RLS for diet tables

2. Create diet plan flow (trainer)
   - Plan name, total calories
   - Add meals (Breakfast, Lunch, Dinner, Snack etc)
   - Each meal: name, time label, order
   - Each item: name, quantity, calories, protein, carbs, fat, optional image

3. Member diet view (/diet)
   - Today's meals listed
   - Calories progress bar (eaten / target)
   - Filter by meal (All / Breakfast / Lunch etc)
   - Each item shows macros
   - Food photo if trainer attached one
```

### Tests to write (Phase 4)
```
Unit:
- calculateTotalCalories sums items correctly
- getMealProgress returns correct percentage

Integration (RLS):
- Member sees only their active diet plan
- Trainer can only see diet plans they created
```

### Verification
- [ ] Trainer creates a diet plan with meals and items
- [ ] Member sees today's diet plan
- [ ] Calorie total correct
- [ ] Macros shown per item
- [ ] Food photos load from R2

---

## Phase 5 — Attendance Module (Week 8)

Members check in. Trainers see who came.

### What to build

```
1. Database migrations
   - attendance table
   - Unique index: one check-in per member per branch per day

2. GPS check-in flow
   - Member taps Check In tab
   - App requests location permission
   - Calculates distance from branch GPS coordinates
   - Within 50m → attendance logged
   - Outside 50m → "You are not at the gym"

3. QR check-in flow
   - Branch has a unique QR code (already in branches table)
   - Member scans QR → attendance logged
   - Duplicate scan on same day → "Already checked in"
   - Print-ready QR page for gym to post at entrance

4. Manual attendance (trainer/receptionist)
   - Open member list → tap member → Mark attendance
   - Records method: manual, marked_by: staff user

5. Attendance history (member view)
   - Weekly strip: M T W T F S S
   - Present / absent / missed dots
   - Streak count
   - Monthly calendar view

6. Attendance dashboard (trainer/owner view)
   - Today's check-ins: live list
   - Who hasn't come in 7+ days (inactive filter)
```

### Tests to write (Phase 5)
```
Unit:
- calculateDistance returns correct metres
- isWithinGeofence returns true/false correctly
- Duplicate check-in prevention logic

Integration:
- Second check-in on same day is ignored
- Manual attendance records marked_by correctly
- Member can only see their own attendance

E2E:
- Member checks in via GPS → logged
- Trainer manually marks attendance → logged
- Duplicate scan shows "Already checked in"
```

### Verification
- [ ] GPS check-in works within 50m of branch
- [ ] GPS check-in rejected outside 50m
- [ ] QR code printable and scannable
- [ ] Duplicate check-in rejected
- [ ] Streak calculated correctly
- [ ] Trainer sees live attendance today

---

## Phase 6 — Progress + Payment (Week 9-10)

Members log their progress. Gym tracks fee payments.

### What to build

```
1. Database migrations
   - member_progress table (user-scoped not workspace-scoped)
   - progress_photos table (workspace-scoped)
   - membership_plans table
   - member_subscriptions table
   - member_payments table

2. Progress logging (member)
   - Log today's weight, measurements
   - Upload progress photos (front/back/side)
   - Notes field
   - One entry per day enforced

3. Progress history (trainer view)
   - Weight chart over time
   - Measurement history table
   - Progress photos timeline

4. Membership plans setup (owner)
   - Create plans: Monthly ₹1500, Quarterly ₹4000 etc
   - Duration in days, price

5. Fee tracking (receptionist/owner)
   - Assign member to a plan → start and end date calculated
   - Payment status: unpaid / paid / partial
   - Mark as paid manually
   - Member uploads UPI screenshot
   - Overdue list in dashboard
   - Payment history per member
```

### Verification
- [ ] Member logs weight → shows in chart
- [ ] Progress photos upload to R2
- [ ] Owner creates membership plans
- [ ] Receptionist assigns member to plan
- [ ] Receptionist marks payment as paid
- [ ] Member uploads screenshot
- [ ] Fee overdue shows in red on dashboard

---

## Phase 7 — Notifications (Week 11)

In-app and WhatsApp notifications.

### What to build

```
1. Database migrations
   - notifications table

2. In-app notifications
   - Bell icon in nav with unread count
   - Notification list page
   - Mark as read

3. Trigger points (create notification on these events)
   - Member enrolled → "Welcome to [gym]"
   - Plan assigned → "Your trainer has assigned a workout plan"
   - Fee due in 3 days → "Your fee is due on [date]"
   - Fee overdue → "Your fee was due on [date]"
   - Payment confirmed → "Payment received ✓"
   - Member inactive 7 days → trainer notified

4. WhatsApp via Gupshup (v2 — wire up later)
   - Notification row created first
   - Edge Function reads and sends via Gupshup API
   - external_sent flag updated
```

### Verification
- [ ] Member sees notification when plan assigned
- [ ] Fee due reminder appears 3 days before
- [ ] Trainer notified when member inactive

---

## Phase 8 — Polish + Launch Prep (Week 12)

Before showing to any gym owner.

### What to build

```
1. Empty states — every page has a friendly empty state
   No members yet → "Add your first member"
   No plan → "Your trainer hasn't assigned a plan yet"
   No attendance → "Check in at the gym to start your streak"

2. Error handling — every API call has proper error UI
   Network error → retry button
   Permission error → "You don't have access to this"
   Not found → clean 404

3. Loading states — every data fetch has a skeleton
   No flash of empty content

4. PWA setup
   - App manifest with FitDesk name and icon
   - Service worker caches today's workout and diet plan
   - Offline shows cached data not blank screen

5. Basic landing page (fitdesk.in)
   - What is FitDesk
   - Who it's for
   - Pricing table
   - "Start free trial" → /onboarding

6. Run full test suite
   - All unit tests pass
   - All integration (RLS) tests pass
   - E2E happy paths pass
```

### Verification
- [ ] Every page has an empty state
- [ ] Every error is handled gracefully
- [ ] PWA installable from mobile browser
- [ ] Offline shows cached workout plan
- [ ] Landing page live at fitdesk.in
- [ ] All tests green

---

## After Launch — v2 Queue

Build these after first paying customers:

- [ ] ZKTeco hardware integration (Phase 9)
- [ ] Razorpay subscription billing (Phase 10)
- [ ] WhatsApp notifications via Gupshup (Phase 11)
- [ ] Capacitor mobile app build (Phase 12)
- [ ] Dealer map in help section (Phase 13)
- [ ] Gym public landing pages (Phase 14)

---

## Daily Development Workflow

```bash
# Start local environment
npx supabase start
npm run dev

# Before committing
npm run test:unit
npm run typecheck

# Before merging to main
npm run test:integration
npm run build  # verify build doesn't break
```

---

## When You're Stuck

1. Check SCHEMA.md for table structure
2. Check STACK.md for how to call Supabase
3. Check the relevant test file — tests describe expected behaviour
4. If a feature isn't working, write the test first, then fix the code

---

## Definition of Done (for each feature)

A feature is done when:
- [ ] It works in the browser
- [ ] Unit test written and passing
- [ ] RLS test written and passing (if touches DB)
- [ ] Empty state handled
- [ ] Error state handled
- [ ] TypeScript shows zero errors
- [ ] Build succeeds
