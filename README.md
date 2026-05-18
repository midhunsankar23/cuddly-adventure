# FitDesk — Gym Management SaaS
### Project Document v1.0

---

## What Is This

FitDesk is an India-first gym management SaaS built for small and mid-size gyms, independent gym trainers, and their members. It replaces the chaos of WhatsApp voice notes, paper attendance registers, and verbal diet instructions with a clean, structured app that works for everyone involved — gym owners, trainers, and members.

The core idea is simple: a trainer assigns a workout plan and diet plan once. The member opens the app and knows exactly what to do that day, what to eat, and whether their fee is due. No asking. No confusion. No WhatsApp floods.

---

## The Problem Being Solved

Indian gym trainers currently manage everything through WhatsApp. They send voice notes for workout instructions, text diet plans, take attendance verbally, and chase members for fees over the phone. Members forget what was assigned, skip workouts because they didn't know what to do, and trainers waste hours repeating themselves every day.

Existing solutions like Trainerize or TrueCoach are priced for Western markets at $50–150/month. Indian gyms won't pay that. Most Indian gym software is either non-existent or terrible. This is the gap.

---

## Who This Is For

There are five types of users. Every user starts with zero role — roles come from what they join or create after signup.

**Gym Owner**
Owns a gym. Pays a monthly SaaS subscription to use FitDesk. Creates the gym profile, adds branches, sets up membership plans, invites trainers, and enrolls members. Sees everything across all branches from one dashboard.

**Branch Manager**
Assigned by the gym owner to manage a specific branch. Sees only their branch — members, attendance, fee status. Cannot see other branches.

**Receptionist**
Office staff at a gym. Enrolls new members, marks payments, manually marks attendance if a member forgot to scan. Cannot create or edit workout or diet plans.

**Trainer (Gym Employed)**
Works at a gym under a gym owner. Assigned to members within that gym. Creates workout plans, diet plans, uploads exercise videos. Can work at multiple gyms simultaneously and also run a personal freelance practice — all from one account.

**Freelance Trainer**
Independent trainer with no gym affiliation. Pays their own separate SaaS subscription. Has their own client list, same features as a gym trainer. Can also be simultaneously employed at a gym — same account, separate workspace.

**Member**
End user who comes to the gym or trains with a freelancer. Never pays FitDesk directly — the gym or freelancer covers their access. Sees their own workout plan, diet plan, attendance history, fee status, and progress log.

---

## The Account Model

One phone number or email = one account. No role is chosen at signup. The user signs up and lands on a blank dashboard.

From there they can:
- Create a gym (becomes gym owner of that workspace)
- Start a freelance trainer profile (becomes owner of that freelance workspace)
- Join a gym as a trainer via invite link
- Join a gym as a member via invite link or gym code
- Do all of the above simultaneously

This is the Instagram model. One account, multiple workspaces, switch between them instantly. A user can be:
- Owner of FitZone Gym
- Trainer at Gold's Gym
- Member at Kerala MMA Academy
- Owner of their own freelance practice

All from one login. Each workspace is completely isolated — FitZone doesn't see your MMA membership data.

---

## Workspace Model

Every context a user participates in is a workspace. Workspaces are of two types:

**Gym workspace** — has branches, membership plans, staff hierarchy (owner → manager → receptionist → trainer → member), and pays a gym SaaS plan.

**Freelancer workspace** — no branches, flat structure (owner + clients), pays a freelancer SaaS plan.

The workspace switcher UI works like switching Instagram accounts. Active workspace shown at top with a green dot. Others listed below with their role badge. One tap to switch.

---

## Subscription and Billing

FitDesk has two separate billing tracks. Members never pay FitDesk.

**Gym Plan (gym owner pays)**

| Plan | Members | Branches | Price |
|---|---|---|---|
| Trial | 20 | 1 | Free, 30 days |
| Starter | 100 | 1 | ₹999/month |
| Growth | 300 | 3 | ₹2,499/month |
| Pro | Unlimited | Unlimited | ₹4,999/month |

**Freelancer Plan (freelancer pays)**

| Plan | Clients | Price |
|---|---|---|
| Trial | 5 | Free, 30 days |
| Solo | 20 | ₹299/month |
| Pro | Unlimited | ₹599/month |

When a trainer is employed at a gym, they are covered under the gym's subscription. When that same trainer runs a freelance practice, they need their own freelancer subscription for that workspace. The two are independent.

Payment collection from gyms and freelancers uses a provider-agnostic system. v1 supports manual payment marking. Razorpay is integrated in v2 with no schema changes needed.

---

## Staff Roles Inside a Gym

Within a gym workspace, roles form a clear hierarchy:

```
Gym Owner
└── Branch Manager (per branch)
    ├── Receptionist
    │   Can: enroll members, mark payments, mark attendance manually
    │   Cannot: create or edit workout/diet plans
    └── Trainer
        Can: create workout plans, diet plans, view member progress, mark attendance
        Cannot: see other trainers' members, access billing, manage branches
└── Member
    Can: view own data only
```

Owner sees all branches. Manager sees their branch only. A trainer assigned to two branches sees members from both.

---

## Member Enrollment

Two paths, both supported:

**Path A — Staff adds member directly**
Receptionist or trainer enters the member's phone number. Member receives a WhatsApp or SMS invite link. They click, sign up or log in, and are already connected to the gym. Status: active immediately.

**Path B — Member self-registers with gym code**
Each branch has a unique QR code and a short gym code. Member opens the app, taps "Join a gym", enters the code. A pending request is sent to the gym admin. Admin approves. Status: pending until approved.

---

## Core Features

### Workout Plans

Trainers assign workout plans to members. Two modes:

**Sequential (default)** — The plan runs in order regardless of calendar day. This is the PPL / bro split model. Member does Session 1, then Session 2, then Session 3, and it loops. Skipped days do not reset the sequence. The plan only advances when the member marks a session complete.

Example:
```
Session 1 — Back + Biceps
Session 2 — Chest + Triceps
Session 3 — Shoulder + Legs
(repeats forever or for N weeks)
```

If the member skips Tuesday, Wednesday still shows Session 2 (not Session 3). The sequence picks up where it left off.

**Calendar** — Sessions are tied to specific days of the week. Monday is always Back day, Wednesday is always Chest day, etc.

**Missed session behaviour** — Set by the trainer at plan creation. Can be changed anytime.

- `push_forward` (default): Missed session is shown at the next visit. Member always completes every session in order, just shifted.
- `skip_and_continue`: Missed session is dropped. Sequence moves on. Good for trainers who don't want makeup sessions.
- `trainer_decides`: Trainer gets a notification when a session is missed and manually decides what to show next.

**Multi-week programs** — Plans can span multiple weeks (e.g. 12-week programs). Each week can have different sessions, allowing progressive overload to be built into the plan structure. Week 1 lighter, Week 2 heavier, Week 4 deload, etc.

**One-off overrides** — Trainer can change today's workout for a specific member without touching the plan template. Member has knee pain today? Trainer swaps leg exercises just for today. The template is untouched for all future sessions.

**Plan switching** — When a trainer switches a member to a new plan, the old plan is deactivated with a reason recorded. Full history preserved. Member's workout logs from the old plan stay attached to it permanently.

**Exercise library** — Trainer uploads an exercise video once (name, muscle group, equipment, video). That exercise lives in their library and can be attached to any session for any member. Upload once, reuse everywhere. No re-uploading the same squat video fifty times.

Platform library — FitDesk also provides a built-in exercise library. Trainer can use platform exercises or their own or a mix.

Videos stored on Cloudflare R2 (zero egress fees). Thumbnails auto-generated.

### Diet Plans

Trainer assigns a structured diet plan to each member. One active plan at a time per member.

Structure:
```
Diet Plan
└── Meals (Breakfast, Lunch, Dinner, Pre-workout snack...)
    └── Items (Oats with banana — 100g — 300 kcal — P:8g C:54g F:5g)
```

Each item has name, quantity, calories, protein, carbs, fat. Optional photo of the food so member knows exactly what it looks like. Optional notes from trainer ("eat this 30 minutes before training").

Total daily calories shown at plan level. Member sees a progress bar for calories consumed vs target.

### Attendance

Two methods:

**QR scan** — Each branch has a QR code posted at the entrance. Member opens the app, taps Check In, scans the gym QR. Attendance logged instantly with branch, timestamp, and method = qr_scan.

**Manual** — Trainer or receptionist marks attendance from the member list. Used when member forgot their phone, scanner issue, or trainer is marking at end of session. Method = manual, marked_by recorded.

**GPS + QR Combined** — QR alone is gameable. GPS alone can be spoofed. Both together is very hard to fake simultaneously.

- Member scans QR at gym entrance AND GPS confirms they're within 50m
- Both must pass → attendance marked
- Either fails → rejected

To fake this you'd need to:
1. Have the QR photo
2. Be physically near the gym anyway

If you're physically near the gym you might as well just go in. This kills 99% of cheating.

Duplicate prevention — only one check-in per member per branch per day is recorded. Second scan on the same day is ignored.

Weekly attendance strip shown to member: M T W T F S S with present/absent/missed visual.

### Member Progress Tracking

Members log their own data. Trainers can view history and chart progress over time.

**Measurements logged per entry:**
- Weight (kg)
- Body fat %
- Chest, waist, hips, bicep, thigh, shoulder (cm)
- Notes — context like "post-competition", "started creatine", "feeling bloated"

One entry per day per user. Measurements are personal data, not workspace-scoped. If a member belongs to two gyms, their weight is the same person's weight.

**Progress photos** — Separate from measurements. Front, back, side photos. Workspace-scoped — Gym A trainer cannot see photos uploaded in context of Gym B. Stored on Cloudflare R2.

### Fee Management (v1 — Manual)

Gym creates membership plans:
- Monthly — ₹1,500 — 30 days
- Quarterly — ₹4,000 — 90 days
- Annual — ₹15,000 — 365 days

Member is enrolled in a plan with a start and end date. Payment tracked manually.

Payment flow:
1. Member pays gym via UPI/cash outside the app
2. Member uploads screenshot of payment in app (optional)
3. Receptionist or trainer marks payment as paid with confirmation
4. `marked_paid_by` and `marked_paid_at` recorded for audit trail
5. Payment status visible on member list — overdue shown in red

Automated reminders — 3 days before fee due, member gets WhatsApp/SMS reminder. On due date, another reminder. Post-due, shown as overdue on dashboard.

Razorpay integration in v2 — schema is ready, no changes needed.

### Video Library

Video libraries are **workspace-scoped**. A trainer working at a gym AND running a freelance practice has two separate video libraries — one per workspace.

**Gym workspace library:**
- Videos uploaded by trainer(s) at that gym
- Only usable within that gym workspace
- Examples: squat form at that gym's equipment, chest press using that machine
- Cannot be accessed from freelancer workspace
- Gym owns the usage rights for these videos

**Freelance workspace library:**
- Videos uploaded by freelancer from their own practice
- Only usable within freelancer workspace
- Cannot be accessed from gym workspace
- Freelancer owns the usage rights

Each video has:
- Name (e.g. "Barbell Squat")
- Muscle group
- Equipment type
- Video file (uploaded to Cloudflare R2)
- Thumbnail

When creating a workout plan, trainer searches their current workspace's library and attaches a video to any exercise. Or leaves it blank — video is optional.

Platform library provides additional videos for common exercises. These are available in all workspaces and take lower priority in search results than workspace-specific videos.

**Why separate?** Prevents accidental leakage of gym-specific content to freelance clients, and keeps IP concerns clear — gym content stays in gym workspace, freelancer content stays in freelancer workspace.

### Notifications

In-app notifications for all users. External WhatsApp/SMS fired via Gupshup or Twilio from the same notification records.

Notification types:
- `fee_due` — member fee reminder
- `workout_reminder` — daily workout reminder
- `plan_assigned` — trainer assigned a new plan
- `plan_changed` — trainer modified today's workout
- `attendance_marked` — confirmation of check-in
- `payment_confirmed` — payment was confirmed by staff
- `member_inactive` — trainer notified when member hasn't checked in for N days
- `missed_session` — when `trainer_decides` mode is on

---

## User Flows

### Gym Owner — First Time Setup

```
1. Download app / open web
2. Sign up with phone (OTP)
3. Land on blank dashboard
4. Tap "Create a gym"
5. Enter gym name, upload logo, enter address
6. First branch created automatically (can rename)
7. Branch QR code generated automatically
8. Create membership plans (Monthly/Quarterly/Annual + price)
9. Choose SaaS plan (or start trial)
10. Invite trainers via phone number or link
11. Trainer accepts → appears in gym roster
12. Assign trainer to branch
13. Enroll first member (enter phone → member gets invite SMS)
14. Assign member to trainer
15. Done — gym is live
```

### Trainer — Daily Flow

```
Morning:
1. Open app → land on trainer workspace
2. See member list with status badges
   - Green: on track
   - Orange: no plan assigned
   - Red: fee due / overdue
   - Gray: inactive (not checked in for 7+ days)
3. Tap any member → see their profile
4. Assign or update workout plan if needed
5. Assign or update diet plan if needed

During gym hours:
6. See who checked in today (live attendance feed)
7. Override today's workout for a member if needed (injury, fatigue)
8. Manually mark attendance for members who forgot to scan

End of day:
9. Check who missed their session today
10. Decide action for missed sessions (if trainer_decides mode)
```

### Member — Daily Flow

```
Arrive at gym:
1. Open app
2. Home screen shows today's workout card
   - Session label (Back + Biceps)
   - Exercise list with sets/reps/weight
   - Video play button next to each exercise
3. Tap Check In tab
4. Scan gym QR code → attendance logged
5. Start workout — follow exercise list
6. Optional: tap play to watch trainer's video for each exercise

After workout:
7. Mark session complete → sequence advances
8. Optional: log today's measurements (weight, measurements)
9. Optional: upload progress photos

Check diet:
10. Tap Diet tab
11. See today's meal plan
12. Structured meals with items, calories, macros
13. Follow trainer's plan
```

### Member — Self Registration Flow

```
1. Gets gym code from gym reception or friend
2. Downloads app, signs up
3. Taps "Join a gym"
4. Enters gym code
5. Request sent to gym admin — status: pending
6. Gym admin sees pending request in dashboard
7. Admin approves
8. Member status becomes active
9. Member gets notification: "You've been approved at FitZone"
10. Member can now access their workspace
```

### Freelance Trainer — Setup

```
1. Signs up (same as any user)
2. Taps "Start freelancing"
3. Creates freelancer profile (name, bio, photo)
4. Chooses freelancer SaaS plan (or trial)
5. Gets a personal invite link
6. Shares link with private clients
7. Clients sign up and land in freelancer workspace
8. Trainer assigns plans exactly like gym trainer
```

### Workspace Switching

```
User taps avatar or workspace icon (top corner)
→ Workspace switcher opens (like Instagram account switcher)
→ Shows all workspaces with role badge
→ Active workspace has green dot
→ Tap any workspace to switch
→ Entire UI context changes to that workspace
→ Owner UI, trainer UI, or member UI depending on role
```

---

## Edge Cases

**Trainer works at two branches of same gym**
Trainer has two `workspace_member` rows — same workspace, different branches. In their member list, members from both branches appear. Trainer can filter by branch.

**Trainer is employed at gym AND has freelance clients**
Two separate workspaces. Gym workspace: role = trainer, covered by gym's subscription. Freelancer workspace: role = owner, needs their own subscription. Both appear in workspace switcher. Completely isolated data.

**Gym owner also works out at their own gym**
Owner creates gym → owner workspace. Owner also enrolls themselves as a member → member row added to same gym. Owner can switch to member view to see their own workout plan assigned by a trainer. Trainer assigns plan to owner just like any other member.

**Member belongs to two gyms simultaneously**
Two separate `workspace_member` rows — different workspaces. Two separate workout plans (one per workspace). Measurements (weight, body fat) are personal and global — same data across both workspaces. Progress photos are workspace-scoped — Gym A trainer cannot see Gym B photos.

**Member misses a session**
Depends on `missed_session_behaviour` setting on the plan:
- `push_forward`: Next visit shows the missed session. Member eventually does every session, just shifted in calendar.
- `skip_and_continue`: Missed session is recorded as a skip. Sequence moves forward. Trainer can see the skip history.
- `trainer_decides`: Trainer gets notified. Trainer manually overrides what the member sees next visit.

**Trainer makes a one-off change to today's workout**
Trainer creates a session override for that member, that date. The override replaces today's session. The template plan is untouched. Future sessions proceed normally from template. The override is recorded permanently — trainer can see "I changed Arjun's session on May 13 because of knee pain."

**Trainer switches a member to a completely new plan**
Old plan deactivated with reason and timestamp recorded. New plan created and set active. Member's workout logs from the old plan are preserved and still viewable. Member sees the new plan immediately.

**Member registers via gym code but gym owner hasn't set up plans yet**
Member status: active. They land on their dashboard which shows "Your trainer hasn't assigned a workout plan yet." Trainer gets a notification: "New member — no plan assigned."

**Receptionist marks payment, then member also uploads screenshot**
Both actions recorded in `member_payments`. The screenshot URL is stored. `marked_paid_by` records the receptionist. No conflict — both events are additive.

**Trial expires and gym owner hasn't paid**
Workspace subscription status → expired. Gym owner dashboard shows a full-screen paywall. Trainers can still log in but see a notice: "Gym subscription expired. Contact your gym owner." Members can still see their plan and check in — access is not cut for members when gym owner hasn't paid. This is a deliberate choice to not punish members for gym owner inaction.

**Gym owner wants to give a member a branded app**
v2 feature — Capacitor wraps the Nuxt web app with the gym's logo, name, and colours. Submitted to Play Store under gym's developer account. Same backend, same data. Gym pays a one-time fee for this service. All future updates are OTA — no re-submission needed unless native features change.

**Two trainers at same gym assigned to same member**
Prevented by app logic. One trainer per member per workspace. If gym wants to reassign a member to a different trainer, old trainer is unassigned and new trainer is assigned. The member's plans created by the old trainer remain readable by the new trainer.

**Duplicate QR scan on same day**
Ignored. Only one attendance record per member per branch per day. The member sees "Already checked in today" message.

**Trainer uploads same video twice**
Not prevented at schema level. Trainer sees it in their library as two entries. Future v2 feature: duplicate detection via file hash before upload.

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | Nuxt 3 | Vue-based, excellent Cloudflare Pages support |
| Mobile | Capacitor | Wraps Nuxt as Android/iOS app, OTA updates via Cloudflare |
| Hosting | Cloudflare Pages | Cheap, fast, global CDN, zero egress |
| Backend / Auth / DB | Supabase | Auth + Postgres + Realtime + Storage in one |
| Video / File Storage | Cloudflare R2 | Zero egress fees, S3-compatible |
| Payments (v1) | Manual + screenshot | No integration needed, ships fast |
| Payments (v2) | Razorpay | India-first, UPI + cards + netbanking |
| Notifications | Gupshup or Twilio | WhatsApp + SMS, India-first pricing |
| Styling | Tailwind + shadcn/vue | Fast, consistent UI |
| Testing | Vitest + Playwright | Unit + integration + E2E from day one |
| CI/CD | GitHub Actions | Free, runs tests on every push |

**Mobile update strategy:**
- UI/logic changes → push to Cloudflare → app picks up on next open (OTA, no store submission)
- Native changes (new plugin, new permission) → build + submit to Play Store / App Store (rare, ~3-4 times a year)

---

## What Is NOT in v1

These are explicitly out of scope for the first version. They are designed into the architecture so they can be added later with minimal work.

- Razorpay payment collection (schema ready, integration not built)
- AI diet suggestions
- Branded white-label app per gym (Capacitor wrap on demand)
- Body transformation timeline / before-after comparison UI
- Multi-language support (Malayalam, Hindi)
- Gym equipment inventory management
- Class / group session scheduling
- Leaderboards or social features
- Apple Watch / WearOS integration

---

## Revenue Model

The app makes money from gym owners and freelance trainers. Members are always free.

**v1 — Subscription only**
Monthly recurring from gym plan + freelancer plan as above.

**v2 — Additional revenue streams**
- Branded app per gym: ₹15,000–25,000 one-time setup fee
- Razorpay payment collection: small transaction cut or higher tier plan
- Platform exercise library premium content
- Multi-location enterprise deals for chains

**Minimum viable revenue target:**
10 gyms at ₹1,500/month average = ₹15,000 MRR. That's the first milestone. All 10 gyms are reachable in Thrissur alone.

---

## Go-To-Market

**Phase 1 — Thrissur ground zero**
Build web app. Walk into 3 local gyms with a live demo. Offer 30-day free trial. Get feedback. Charge after trial.

**Phase 2 — Instagram content**
Record Reels showing:
- "Trainer sending WhatsApp voice notes" vs "Member opening app and seeing plan"
- Dashboard showing fee overdue tracker — gym owners relate immediately
- Workout plan assignment flow — 30 seconds, live demo

No ads needed initially. Organic reach in gym/fitness creator community.

**Phase 3 — Branded app upsell**
Once a gym trusts the product, offer them their own branded app. High-margin, low-effort, strong retention anchor.

---

## Testing Strategy

Tests written from day one. No manual testing for anything that can be automated.

**Unit tests (Vitest)**
- Workout sequence logic (sequential plan advancement)
- Missed session behaviour decision tree
- Fee due date calculation
- Calorie totals from diet items

**Integration tests (Vitest + Supabase local)**
- RLS: owner can see all branches, member cannot see other members' data
- Trainer can only edit their assigned members
- Receptionist can mark payment but cannot create workout plans
- One active plan constraint enforced

**E2E tests (Playwright)**
- Gym owner signs up, creates gym, invites trainer
- Trainer assigns workout plan to member
- Member logs in, sees plan, scans QR for attendance
- Receptionist marks payment
- Workspace switching works across roles

**CI/CD (GitHub Actions)**
All tests run automatically on every push. No broken code merges to main.

---

## Project Name

**FitDesk** — working title. Short, clear, professional enough for a SaaS product, memorable enough for a gym owner to search it.

Domain: fitdesk.in (check availability before locking in)

---

*Document version 1.0 — May 2026*
*To be updated as decisions are made during development*
