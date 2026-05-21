# FitDesk — Module Guide

This replaces BUILD_ORDER.md phases.
Each module is a self-contained unit you can build, test, and iterate on independently.
Modules don't import from each other's internals — they communicate through the database.

---

## The One Rule

**After M0 is done, every other module is independent.**
You can build M3 before M2. You can go back and improve M1 while M4 is half-done.
The database ties everything together. The code doesn't need to.

---

## Module Map

```
M0: Foundation          ← build this first, everything needs it
    Auth + Workspace

        ↓ (done once, never touched again)

M1        M2          M3        M4        M5        M6
Members   Attendance  Fees      Workout   Diet      Progress
                                  ↕
                               M4b: Injuries
                               (extension of workout)

                        ↓ (add last, feeds off all modules)

                            M7: Notifications
                            M8: Settings
```

---

## The Iteration Levels

Every module goes through 4 levels. The rule: **don't take one module to L3 while others are at L0.**
Keep them all moving together. You always have a working app, just with placeholder screens in some areas.

```
L0 — Skeleton    Page exists. Button exists. Nothing works yet. Navigation works.
L1 — Data        Real data shows. No writes yet. Queries working. Empty states.
L2 — Interactive Forms work. Creates, updates, deletes work. Basic error handling.
L3 — Complete    Loading states. All edge cases. RLS tests. Ready for real users.
```

---

## M0 — Foundation
**Must be done before anything else. Build it fully to L3 before starting other modules.**

### What it covers
- Login page (phone number input)
- OTP verify page
- Post-login routing logic:
  - 0 workspaces → /onboarding
  - 1 workspace → /dashboard
  - 2+ workspaces → /workspace/switch
- Onboarding: "Create a gym" / "Start freelancing" / "Join a gym"
- Workspace switcher (Instagram-style)
- Authenticated layout with nav
- Role-aware nav (owner sees different tabs than member)
- Auth guard in middleware.ts

### Files it owns
```
app/(auth)/login/page.tsx
app/(auth)/verify/page.tsx
app/(onboarding)/onboarding/page.tsx
app/(onboarding)/create-gym/page.tsx
app/(onboarding)/join-gym/page.tsx
app/(app)/layout.tsx
app/(app)/dashboard/page.tsx
app/(app)/workspace/switch/page.tsx
middleware.ts
stores/workspace-store.ts
stores/auth-store.ts
hooks/use-auth.ts
hooks/use-workspace.ts
hooks/use-permissions.ts
lib/supabase/client.ts
lib/supabase/server.ts
types/index.ts
```

### Done when
- [ ] Can sign up and log in with phone OTP
- [ ] Can create a gym and land on dashboard
- [ ] Can start freelancing and land on dashboard
- [ ] Can switch between workspaces
- [ ] Role badge shows correctly
- [ ] Unauthenticated user redirected to /login
- [ ] No workspace → redirected to /onboarding
- [ ] TypeScript zero errors

---

## M1 — Members
**Dependency: M0**

### What it covers
- Member list page (owner/trainer view)
- Filter by: All | No plan | Fee due | Inactive
- Add member by phone number → sends invite
- Member joins via gym code → pending → approved
- Member profile page (basic info, trainer assigned)
- Assign trainer to member
- Suspend / reactivate member

### Files it owns
```
app/(app)/members/page.tsx
app/(app)/members/[id]/page.tsx
app/(app)/join/page.tsx            ← public page, no auth needed
components/members/
hooks/use-members.ts
```

### Done when
- [ ] Owner/trainer can see member list
- [ ] Can add a member by phone
- [ ] Member joins via gym code and shows as pending
- [ ] Admin approves → member becomes active
- [ ] Trainer assigned to member
- [ ] Member cannot see this page (access denied)

---

## M2 — Attendance
**Dependency: M0, M1**

### What it covers
- QR check-in (member scans gym QR)
- GPS check-in (within 50m of branch)
- GPS + QR combined mode
- Manual mark (trainer/receptionist)
- Duplicate check-in on same day → rejected with message
- Member's weekly attendance strip (M T W T F S S)
- Trainer's live attendance feed for today

### Files it owns
```
app/(app)/attendance/page.tsx
components/attendance/
hooks/use-attendance.ts
```

### Done when
- [ ] Member can check in with QR
- [ ] GPS within 50m → allowed
- [ ] GPS outside 50m → rejected
- [ ] Duplicate same-day check-in → rejected
- [ ] Weekly strip shows correct present/absent
- [ ] Trainer sees today's check-ins live

---

## M3 — Fees
**Dependency: M0, M1**

### What it covers
- Owner creates membership plans (Monthly/Quarterly/Annual + price)
- Enroll member in a plan (set start/end date)
- Member uploads payment screenshot
- Receptionist/trainer marks payment as paid
- Overdue members shown in red on member list
- Fee due reminders (notifications — links to M7)

### Files it owns
```
app/(app)/fees/page.tsx
app/(app)/members/[id]/subscription/page.tsx
components/fees/
hooks/use-fees.ts
```

### Done when
- [ ] Owner can create membership plans
- [ ] Member enrolled in a plan
- [ ] Member can upload screenshot
- [ ] Receptionist marks as paid
- [ ] Overdue shows in red
- [ ] Cannot mark payment without receptionist/owner/manager role

---

## M4 — Workout
**Dependency: M0, M1**
**The biggest module. Split it into sub-tasks within the module.**

### Sub-tasks (do in this order)
```
M4a — Exercise Library
  Upload exercise: name, muscle group, equipment, video → R2
  Platform exercises visible to all
  Workspace exercises visible only within workspace

M4b — Create Plan + Sessions
  Trainer creates a plan for a member
  Add sessions (Push day, Pull day, Legs day...)
  Add exercises to each session (Squat 4×10, Bench 3×8)
  Set: sequential / plan_end_behaviour / missed_session_behaviour

M4c — Member View
  Member opens app → sees today's session
  Exercise list with sets/reps/weight
  Video play button next to each exercise

M4d — Session Completion + Pointer
  Member marks session complete → current_session_index advances
  Push_forward: missed session stays
  Skip_and_continue: moves on
  Plan loops or ends based on plan_end_behaviour

M4e — Member Self-Service
  Member skips a session ("skip today")
  Member swaps an exercise ("did leg press instead of squats — knee hurt")
  No permission needed, trainer sees it in the log

M4f — Trainer Controls
  Trainer overrides one session for one member on one date
  Override linked to injury if applicable
  Plan pause (member going on vacation)
  Plan switch (deactivate old, create new)

M4g — Injuries
  Member or trainer flags a body part as painful
  Active injuries shown on member profile
  Injury linked to skips and overrides for audit trail
  Resolved when pain clears
```

### Files it owns
```
app/(app)/workout/page.tsx                  ← member's today view
app/(app)/workout/[planId]/page.tsx         ← full plan view
app/(app)/workout/library/page.tsx          ← exercise library
app/(app)/workout/new/page.tsx              ← trainer creates plan
app/(app)/workout/[planId]/log/page.tsx     ← log a session
app/(app)/members/[id]/injuries/page.tsx    ← injury tracking
components/workout/
hooks/use-workout.ts
hooks/use-injuries.ts
```

### Done when
- [ ] Trainer creates a plan with sessions and exercises
- [ ] Member sees correct session based on current_session_index
- [ ] Mark complete advances the pointer
- [ ] Skip works correctly per missed_session_behaviour
- [ ] Member can swap an exercise and trainer sees it
- [ ] Trainer can override one day
- [ ] Plan pause stops missed-session counting
- [ ] Injury flagging works and shows on profile
- [ ] Video plays from R2

---

## M5 — Diet
**Dependency: M0, M1**
**Independent of M4. Can be built before, after, or in parallel.**

### Sub-tasks
```
M5a — Trainer Creates Diet
  Trainer creates a diet plan for a member
  Add meals (Breakfast, Lunch, Dinner, Pre-workout)
  Add food items to each meal (Oats 100g — 350 kcal — P:10 C:60 F:6)
  Upload reference photos per item (multiple allowed)
  Set plan_variant: default / training_day / rest_day

M5b — Member Views Diet
  Member opens Diet tab
  Sees today's meals
  Calorie progress bar (total vs target)
  Macros per item
  Reference photos

M5c — Food Log (member uploads what they ate)
  Member taps "Log what I ate"
  Selects meal time (breakfast, lunch, etc.)
  Writes description ("Rice and dal, small portion")
  Uploads photos (multiple allowed)
  Trainer can view and leave a note
```

### Files it owns
```
app/(app)/diet/page.tsx                 ← member's diet view
app/(app)/diet/new/page.tsx             ← trainer creates diet
app/(app)/diet/log/page.tsx             ← member logs food
app/(app)/members/[id]/diet/page.tsx    ← trainer views member's food log
components/diet/
hooks/use-diet.ts
hooks/use-food-log.ts
```

### Done when
- [ ] Trainer creates diet plan with meals and items
- [ ] Multiple photos upload per item
- [ ] Member sees today's meals
- [ ] Calorie total and macros shown
- [ ] Member can log what they actually ate with photos
- [ ] Trainer can see food log and leave a note
- [ ] Training day / rest day variants work

---

## M6 — Progress
**Dependency: M0, M1**
**Independent of everything else.**

### What it covers
- Member logs weight + measurements (one entry per day)
- Member uploads progress photos (front/back/side)
- Trainer sees weight chart over time
- Trainer sees photo timeline

### Files it owns
```
app/(app)/progress/page.tsx
components/progress/
hooks/use-progress.ts
```

### Done when
- [ ] Member logs weight and measurements
- [ ] Progress photos upload to R2
- [ ] Trainer sees chart with weight history
- [ ] Photos are workspace-scoped (correct isolation)

---

## M7 — Notifications
**Dependency: all other modules**
**Build this last. Other modules create notification rows — this module just displays them.**

### What it covers
- Bell icon in nav with unread count
- Notification list page
- Mark as read
- Each module creates notifications on events:
  - Fee due (M3)
  - Plan assigned (M4)
  - Session missed (M4)
  - Injury flagged (M4g)
  - Payment confirmed (M3)
  - Member inactive 7 days (background cron)

### Files it owns
```
app/(app)/notifications/page.tsx
components/notifications/
hooks/use-notifications.ts
```

### Done when
- [ ] Bell shows unread count
- [ ] Can view all notifications
- [ ] Mark as read works
- [ ] Correct notifications fire from other modules

---

## M8 — Settings
**Dependency: M0**
**Can be built any time after M0. Low priority.**

### What it covers
- Gym profile (name, logo, address)
- Branch management (add branch, set GPS coordinates)
- Subscription / billing view (which FitDesk plan, renew)
- Hardware settings (ZKTeco — v2)

### Files it owns
```
app/(app)/settings/gym/page.tsx
app/(app)/settings/billing/page.tsx
app/(app)/settings/branches/page.tsx
components/settings/
```

---

## What to Build and In What Order

```
Step 1 — M0 to L3 (full)
  Get auth, workspace, onboarding working completely.
  Don't start anything else until login → dashboard works.
  Time: 1-2 weeks.

Step 2 — All modules to L0 (skeletons)
  Build every page as a skeleton.
  Static text. Placeholder buttons. No data. No forms.
  Navigation between all pages works.
  You can click through the entire app even if nothing does anything.
  Time: 2-3 days.

Step 3 — Pick ONE module, take it to L2
  Recommended: M1 (Members) — everything else needs real members to test.
  Then M2 (Attendance) — quick win, self-contained.
  Then M3 (Fees) — quick win, self-contained.
  Time: 1 week each.

Step 4 — M4 and M5 in parallel
  Biggest modules. Split M4 into sub-tasks (M4a through M4g).
  Do one sub-task at a time.
  M5 can happen between M4 sub-tasks when you need a break.
  Time: 2-3 weeks.

Step 5 — M6, M7, M8
  Progress tracking, notifications, settings.
  Time: 1 week.

Step 6 — L3 pass across all modules
  Go through every module and handle all edge cases.
  Empty states, loading states, error states.
  Write tests.
  Time: 1 week.
```

---

## The Golden Rule for Solo Dev

**Always be shippable.**

After Step 2, you have a working app with skeleton screens.
After Step 3, you have a working app where members can be managed and attendance tracked.
After Step 4, you have a working app that does everything a gym needs.

At any point you can show it to a gym owner and say "this part works, this part coming soon."
That's how you stay sane as a solo developer.

Never go deeper than 2 levels on one module while others are still at L0.

---

## How Modules Talk to Each Other

They don't — directly. The database connects them.

```
Member list (M1) shows "no workout plan" badge
→ M1 doesn't import from M4
→ M1 reads workout_plans table: if no active plan for this member, show badge

Attendance (M2) affects which diet to show (M5)
→ M2 doesn't import from M5
→ M5 reads attendance table: if checked_in today → show training_day diet

Missed session cron (M4) creates a notification (M7)
→ M4 doesn't import from M7
→ M4 inserts a row in notifications table — M7 just reads it
```

When you need data from another module, query the table directly.
Never import a hook from another module's folder.
