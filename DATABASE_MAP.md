# FitDesk — Database Map
# Open this whenever you're unsure which table to touch.

---

## Quick Reference — Every Table in One Line

### Identity & Access
| Table | What it stores |
|---|---|
| `users` | One row per person. Phone or email. That's it. |
| `workspaces` | One row per gym or freelancer practice. |
| `workspace_members` | Who belongs to which workspace, with what role. THE central table. |
| `branches` | Physical locations of a gym. One gym can have many branches. |

### FitDesk's Own Billing (you charging gyms/trainers)
| Table | What it stores |
|---|---|
| `subscription_plans` | Your SaaS tiers: Trial, Starter, Growth, Pro. You seed this once. |
| `workspace_subscriptions` | Which plan a gym/trainer is currently on. |
| `workspace_payments` | Payment records for gym/trainer paying FitDesk. |

### Gym's Billing (gym charging their members)
| Table | What it stores |
|---|---|
| `membership_plans` | Plans a gym offers: Monthly ₹1500, Quarterly ₹4000, etc. |
| `member_subscriptions` | Which plan a member is enrolled in, start/end date. |
| `member_payments` | Payment records for member paying gym. Screenshot URL lives here. |

### Workout
| Table | What it stores |
|---|---|
| `exercise_library` | Videos/exercises uploaded by trainer or provided by platform. |
| `workout_plans` | A plan assigned to one member by one trainer. The top-level container. |
| `workout_sessions` | Individual days/sessions within a plan. "Push day", "Pull day". |
| `workout_exercises` | The exercises inside a session. Squats 4×10, Bench 3×8. |
| `workout_session_overrides` | Trainer swaps a session for one member on one specific date. |
| `workout_exercise_overrides` | The replacement exercises inside an override session. |
| `workout_session_skips` | Records when a session was skipped — by whom and why. |
| `workout_logs` | Member completed a session. One row per session done. |
| `workout_set_logs` | The actual sets logged during a session. 10 reps @ 80kg. |
| `plan_pauses` | Member/trainer paused the plan (vacation, injury break). |

### Injuries
| Table | What it stores |
|---|---|
| `member_injuries` | Body part that hurts, severity, when it started, when resolved. |

### Diet (Trainer side — prescribed plan)
| Table | What it stores |
|---|---|
| `diet_plans` | A diet assigned to one member by one trainer. |
| `diet_meals` | Meals within the plan: Breakfast, Lunch, Pre-workout. |
| `diet_items` | Food items within a meal: Oats 100g, 350 kcal, P:10 C:60 F:6. |
| `diet_item_photos` | Multiple reference photos per food item (what 100g looks like). |

### Diet (Member side — what they actually ate)
| Table | What it stores |
|---|---|
| `food_logs` | Member's log of what they actually ate. Text description + meal time. |
| `food_log_photos` | Photos the member uploaded of their actual food. Multiple per log. |

### Attendance
| Table | What it stores |
|---|---|
| `attendance` | One check-in per member per branch per day. Method: QR/GPS/manual. |

### Progress
| Table | What it stores |
|---|---|
| `member_progress` | Weight, body fat, measurements. User-scoped — not tied to one gym. |
| `progress_photos` | Front/back/side photos. Workspace-scoped. |

### Notifications
| Table | What it stores |
|---|---|
| `notifications` | Every notification: in-app, WhatsApp, SMS. One row per message. |

### Payments v2 (Razorpay/Stripe — not built yet)
| Table | What it stores |
|---|---|
| `workspace_payment_configs` | Gym's own Razorpay keys. |
| `payment_webhooks_raw` | Raw webhook payloads saved before processing. Never lose a payment. |
| `payment_events` | Normalised payment events. Provider-agnostic. |

### ZKTeco Hardware v2 (not built yet)
| Table | What it stores |
|---|---|
| `zkteco_device_commands` | Commands sent to biometric device: enroll user, delete user. |
| `zkteco_raw_logs` | Raw attendance data from device before processing. |

---

## Visual Relationship Map

```
USER
 └─── workspace_members ──────────────────────────────────────────────┐
        │  role: owner/manager/receptionist/trainer/member             │
        │  status: pending/active/suspended                            │
        │                                                              │
        ├── WORKSPACE ──────────────────────────────────────────────── │
        │     └── branches                                             │
        │     └── workspace_subscriptions → subscription_plans         │
        │     └── workspace_payments                                   │
        │     └── membership_plans                                     │
        │     └── exercise_library                                     │
        │     └── workout_plans                                        │
        │     └── diet_plans                                           │
        │                                                              │
        ├── ATTENDANCE                                                 │
        │     attendance ←──────────────────────────────────────────── ┘
        │       └── branch_id (which location)                        (workspace_member_id)
        │
        ├── SUBSCRIPTIONS (member paying gym)
        │     member_subscriptions → membership_plans
        │       └── member_payments
        │
        ├── PROGRESS (personal — not workspace scoped)
        │     member_progress  (weight, measurements — same across all gyms)
        │     progress_photos  (workspace scoped — gym A can't see gym B)
        │
        ├── INJURIES
        │     member_injuries
        │
        └── FOOD LOG (what they actually ate)
              food_logs → food_log_photos


WORKOUT_PLANS (workspace + trainer + member)
  └── workout_sessions (Push day, Pull day, Legs day...)
        └── workout_exercises (Squat 4×10, Bench 3×8...)
  └── workout_session_overrides (trainer swaps one session on one date)
        └── workout_exercise_overrides (the replacement exercises)
  └── workout_session_skips (missed/skipped, why, by whom)
  └── workout_logs (member completed a session)
        └── workout_set_logs (actual sets: 10 reps @ 80kg)
              ↑ planned_exercise_id + swap_note (member swapped exercise themselves)
  └── plan_pauses (vacation, injury break — cron ignores paused plans)


DIET_PLANS (workspace + trainer + member)
  └── diet_meals (Breakfast, Lunch, Pre-workout...)
        ├── image_url (cover photo of the meal)
        └── diet_items (Oats 100g — 350 kcal — P:10 C:60 F:6)
              └── diet_item_photos (multiple reference photos per item)


MEMBER_INJURIES
  └── linked from workout_session_overrides (why was this override created)
  └── linked from workout_set_logs (why was this exercise skipped/swapped)
```

---

## "When I need to do X, touch these tables"

### Auth & Login
```
User signs up          → users
User joins workspace   → workspace_members (insert row)
User switches workspace→ just read workspace_members (Zustand, no DB write)
User is blocked        → workspace_members.status = 'suspended'
```

### Creating a Gym
```
Owner creates gym      → workspaces (insert, type='gym')
First branch auto-made → branches (insert)
Owner becomes member   → workspace_members (insert, role='owner')
Trial starts           → workspace_subscriptions (insert, status='trial')
```

### Adding a Member
```
Staff adds member         → users (upsert by phone)
                          → workspace_members (insert, role='member')
Member joins via code     → workspace_members (insert, status='pending')
Admin approves            → workspace_members (update, status='active')
Assign to membership plan → member_subscriptions (insert)
```

### Workout Plan
```
Trainer creates plan      → workout_plans (insert)
Add sessions to plan      → workout_sessions (insert, session_index 0,1,2...)
Add exercises to session  → workout_exercises (insert)
Assign to member          → workout_plans.member_id (already there at create)
Member sees today's work  → read workout_plans.current_session_index
                          → read workout_sessions where session_index matches
Member completes session  → workout_logs (insert)
                          → workout_set_logs (insert, one row per set)
                          → workout_plans.current_session_index += 1
Member swaps an exercise  → workout_set_logs.planned_exercise_id = original
                          → workout_set_logs.exercise_id = what they did
                          → workout_set_logs.swap_note = "knee hurt"
Trainer overrides today   → workout_session_overrides (insert)
                          → workout_exercise_overrides (insert replacement exercises)
Session missed            → workout_session_skips (insert)
Plan paused (vacation)    → plan_pauses (insert, resumed_at = null)
Plan resumed              → plan_pauses (update, resumed_at = today)
Plan ends / new phase     → workout_plans (update, is_active=false, switch_reason)
                          → workout_plans (insert new plan)
```

### Diet Plan
```
Trainer creates diet      → diet_plans (insert)
Add meals                 → diet_meals (insert: Breakfast, Lunch...)
Add food items            → diet_items (insert: Oats 100g, macros...)
Add photos to item        → diet_item_photos (insert, upload to R2 first)
Member sees their diet    → read diet_plans where member_id = user, is_active = true
Member logs what ate      → food_logs (insert: meal_time, description)
Member uploads food photo → food_log_photos (insert, upload to R2 first)
Trainer leaves feedback   → food_logs.trainer_note (update)
```

### Attendance
```
Member checks in (QR/GPS) → attendance (insert)
                            unique: one per member per branch per day
Staff marks manually      → attendance (insert, method='manual', marked_by=staff_id)
Member sees weekly strip  → read attendance where workspace_member_id, last 7 days
Trainer sees today live   → read attendance where branch_id, date=today
```

### Injuries
```
Member flags pain         → member_injuries (insert, reported_by='member')
Trainer logs injury       → member_injuries (insert, reported_by='trainer')
See active injuries       → member_injuries where resolved_at is null
Injury resolved           → member_injuries (update, resolved_at=today)
Override caused by injury → workout_session_overrides.injury_id = injury id
Exercise skipped for pain → workout_set_logs.injury_id = injury id
```

### Fees
```
Owner creates plan        → membership_plans (insert: Monthly ₹1500, 30 days)
Enrol member in plan      → member_subscriptions (insert, start/end date)
Member uploads screenshot → member_payments.screenshot_url (update, upload R2 first)
Staff marks paid          → member_payments (update, status='paid', marked_paid_by)
Check who is overdue      → member_subscriptions where end_date < today, status='active'
```

### Progress
```
Member logs measurements  → member_progress (insert/upsert by user_id + date)
Member uploads body photo → progress_photos (insert, workspace_member_id scoped)
Trainer views chart       → read member_progress where user_id = member's user_id
```

### Notifications
```
Any event fires           → notifications (insert, channel='in_app')
WhatsApp also needed      → notifications (insert, channel='whatsapp') — v2
Member reads notification → notifications (update, read=true)
```

---

## The Two Rules That Prevent 90% of Mistakes

**Rule 1: Always scope queries to workspace_id**
```
Wrong:  select * from workout_plans where member_id = $1
Right:  select * from workout_plans where member_id = $1 and workspace_id = $2
```
Without workspace_id, a member's data from Gym A could leak into Gym B.

**Rule 2: workspace_members is the bridge — always go through it**
```
To check if a user can access a workspace  → workspace_members
To find what role a user has               → workspace_members.role
To find which branch a trainer belongs to  → workspace_members.branch_id
To get all members of a gym               → workspace_members where workspace_id
```
Almost every query touches workspace_members. When lost, start here.

---

## v1 vs v2 — What to Build Now

### Build in v1
- users, workspaces, workspace_members, branches
- subscription_plans, workspace_subscriptions, workspace_payments
- membership_plans, member_subscriptions, member_payments
- exercise_library, workout_plans, workout_sessions, workout_exercises
- workout_session_overrides, workout_exercise_overrides
- workout_session_skips, workout_logs, workout_set_logs
- plan_pauses
- member_injuries
- diet_plans, diet_meals, diet_items, diet_item_photos
- food_logs, food_log_photos
- attendance
- member_progress, progress_photos
- notifications (in-app only)

### Build in v2
- workspace_payment_configs (Razorpay/Stripe keys)
- payment_webhooks_raw, payment_events
- zkteco_device_commands, zkteco_raw_logs
- notifications (WhatsApp + SMS via Gupshup)
- exercise_alternatives (pre-defined injury substitutions)
- Calendar mode sessions (day_of_week based plans)

---

## Columns You Will Forget Exist (but matter a lot)

| Column | Table | What it does |
|---|---|---|
| `current_session_index` | `workout_plans` | Where the member is in their plan sequence |
| `plan_end_behaviour` | `workout_plans` | `loop` = PPL forever. `end` = 12-week program stops |
| `missed_session_behaviour` | `workout_plans` | `push_forward` / `skip_and_continue` / `trainer_decides` |
| `is_active` | `workout_plans`, `diet_plans` | Only one active plan per member per workspace |
| `switch_reason` | `workout_plans`, `diet_plans` | Why the old plan was replaced — audit trail |
| `was_override` | `workout_logs` | This log was from an overridden session, not the template |
| `planned_exercise_id` | `workout_set_logs` | Member swapped this exercise. NULL = followed the plan |
| `swap_note` | `workout_set_logs` | What the member wrote: "knee hurt, did leg press instead" |
| `injury_id` | `workout_set_logs`, `workout_session_overrides` | Links a skip/override to a specific injury |
| `marked_paid_by` | `member_payments` | Who confirmed this payment — audit trail |
| `reported_by` | `member_injuries` | Was this flagged by member or trainer |
| `resolved_at` | `member_injuries` | NULL = injury is still active |
| `method` | `attendance` | qr_scan / gps / manual / biometric |
| `is_platform` | `exercise_library` | FitDesk's built-in exercises vs trainer's own |
| `plan_variant` | `diet_plans` | default / training_day / rest_day |

---

*This file is the source of truth for the database. Update it whenever a new table or important column is added.*
