# FitDesk — Table Map

Quick reference. When building a feature, scan here first.
One line per table. Links show exactly what connects to what.

---

## Quick Lookup — "What do I query for X?"

| I need to... | Use this table |
|---|---|
| Know who the logged-in user is | `users` |
| Know the user's role in the active workspace | `workspace_members` (filter by `workspace_id` + `user_id`) |
| List all members in a gym | `workspace_members` where `workspace_id = ?` and `role = 'member'` |
| Check if member has an active membership plan | `member_subscriptions` where `status = 'active'` |
| Find who is overdue on payment | `member_subscriptions` where `end_date < now()` or `payment_status = 'unpaid'` |
| Get a member's active workout plan | `workout_plans` where `member_id = ?` and `is_active = true` |
| Know which session to show a member today | `workout_plans.current_session_index` → lookup in `workout_sessions` |
| Show today's diet to a member | `diet_plans` where `member_id = ?` and `is_active = true`, pick variant based on `attendance` |
| Check if member came in today | `attendance` where `workspace_member_id = ?` and `checked_in_at::date = today` |
| Show member's injury history | `member_injuries` where `workspace_member_id = ?` |
| See what member actually ate | `food_logs` where `workspace_member_id = ?` |
| Show weight/measurement history | `member_progress` where `user_id = ?` |
| Show body progress photos | `progress_photos` where `workspace_member_id = ?` |
| Send a notification | Insert into `notifications` |

---

## All Tables — Module by Module

### 🏛️ Foundation (always needed)

| Table | What it stores | Key columns |
|---|---|---|
| `users` | One row per person. Identity only — no role here. | `id`, `phone`, `email`, `full_name` |
| `workspaces` | A gym or freelancer account. | `id`, `name`, `type`, `owner_id`, `default_currency` |
| `branches` | Physical locations within a gym. | `id`, `workspace_id`, `name`, `qr_code`, `latitude`, `longitude`, `timezone` |
| `workspace_members` | **THE central table.** User + workspace + role. | `id`, `workspace_id`, `user_id`, `role`, `status`, `branch_id` |

> **workspace_members is the bridge for almost everything.**
> Most tables link to either `workspace_members.id` (workspace-scoped) or `users.id` (user-scoped).

---

### 💳 Subscriptions & Billing

| Table | What it stores | Key columns |
|---|---|---|
| `subscription_plans` | FitDesk's own plans (Trial/Starter/Growth/Pro). Seeded once. | `id`, `name`, `type`, `max_members` |
| `subscription_plan_prices` | Price per plan per currency. | `plan_id`, `currency`, `price_monthly` |
| `workspace_subscriptions` | Which FitDesk plan this gym is on. | `workspace_id`, `plan_id`, `status`, `current_period_ends_at` |
| `workspace_payments` | Gym paying FitDesk. | `workspace_id`, `subscription_id`, `amount`, `currency`, `status` |
| `membership_plans` | Plans a gym creates for their members (₹1500/month etc.). | `workspace_id`, `name`, `duration_days`, `price`, `currency` |
| `member_subscriptions` | Member enrolled in a gym's membership plan. | `workspace_member_id`, `plan_id`, `start_date`, `end_date`, `payment_status` |
| `member_payments` | Payment records for a member paying the gym. | `member_subscription_id`, `amount`, `screenshot_url`, `status`, `marked_paid_by` |

**Chain:** `membership_plans` ← `member_subscriptions` ← `member_payments`

---

### 🏋️ Workout

| Table | What it stores | Key columns |
|---|---|---|
| `exercise_library` | Exercise videos/details. `workspace_id = null` = platform exercise. | `workspace_id`, `name`, `muscle_group`, `video_url`, `is_platform` |
| `workout_plans` | A plan assigned to one member. | `workspace_id`, `trainer_id`, `member_id`, `current_session_index`, `plan_end_behaviour`, `missed_session_behaviour`, `is_active` |
| `workout_sessions` | Sessions in the plan (Push day, Legs day…). | `plan_id`, `session_index`, `week_number`, `day_in_week`, `label`, `rest_day` |
| `workout_exercises` | Exercises in a session. The template. | `session_id`, `exercise_id`, `sets`, `reps`, `weight_kg` |
| `workout_session_overrides` | Trainer replaces a session for a member on one date. | `plan_id`, `member_id`, `session_index`, `override_date`, `injury_id` |
| `workout_exercise_overrides` | Replacement exercises for an override. | `override_id`, `exercise_id`, `sets`, `reps` |
| `workout_session_skips` | A session that was missed or skipped. | `plan_id`, `session_id`, `member_id`, `scheduled_date`, `skip_reason` |
| `plan_pauses` | Plan paused for vacation or injury. Cron ignores paused plans. | `plan_id`, `paused_by`, `paused_at`, `resumed_at`, `reason` |
| `workout_logs` | Member completed a session (or did an off-plan workout). | `plan_id`, `session_id`, `member_id`, `workspace_member_id`, `logged_date`, `pain_notes` |
| `workout_set_logs` | Actual sets within a completed session. | `log_id`, `exercise_id`, `planned_exercise_id`, `swap_note`, `reps_done`, `weight_done_kg`, `injury_id` |

**Chain:** `workout_plans` → `workout_sessions` → `workout_exercises`
**Log chain:** `workout_logs` → `workout_set_logs`
**Override chain:** `workout_session_overrides` → `workout_exercise_overrides`

**Pointer logic:**
- `workout_plans.current_session_index` tells you which session the member is on
- It advances only when the member marks a session complete
- Check `plan_pauses` — if an active pause exists, skip missed-session logic
- Check `workout_session_overrides` — if one exists for today's date, use that instead

---

### 🥗 Diet

| Table | What it stores | Key columns |
|---|---|---|
| `diet_plans` | Diet plan assigned to one member. Multiple can exist (training vs rest day). | `workspace_id`, `trainer_id`, `member_id`, `plan_variant`, `is_active` |
| `diet_meals` | Meals within a plan (Breakfast, Lunch, Dinner…). | `plan_id`, `name`, `time_label`, `image_url`, `order_index` |
| `diet_items` | Food items within a meal (Oats 100g, 350 kcal…). | `meal_id`, `name`, `quantity`, `calories`, `protein_g`, `carbs_g`, `fat_g` |
| `diet_item_photos` | Multiple reference photos per food item (trainer uploads). | `item_id`, `photo_url`, `order_index` |
| `food_logs` | What the member actually ate. Separate from the plan. | `workspace_member_id`, `logged_date`, `meal_time`, `description`, `trainer_note` |
| `food_log_photos` | Photos of actual food uploaded by the member. | `log_id`, `photo_url`, `order_index` |

**Trainer's prescription chain:** `diet_plans` → `diet_meals` → `diet_items` → `diet_item_photos`
**Member's actual food:** `food_logs` → `food_log_photos`

**Which diet to show today?**
1. Check `attendance` — did member check in today?
2. If yes → find `diet_plans` with `plan_variant = 'training_day'`
3. If no → find `diet_plans` with `plan_variant = 'rest_day'`
4. If neither exists → fallback to `plan_variant = 'default'`

---

### 📅 Attendance

| Table | What it stores | Key columns |
|---|---|---|
| `attendance` | One check-in per member per branch per day. Duplicate blocked by unique index. | `workspace_member_id`, `branch_id`, `checked_in_at`, `method`, `marked_by` |

**Check-in methods:** `qr_scan` | `manual` | `biometric` | `gps`

---

### 🩹 Injuries

| Table | What it stores | Key columns |
|---|---|---|
| `member_injuries` | Body part pain/injury log. | `workspace_member_id`, `body_part`, `severity`, `reported_by`, `started_at`, `resolved_at` |

**Where it connects:**
- `workout_set_logs.injury_id` — links a set skip/swap to the injury
- `workout_session_overrides.injury_id` — links a trainer override to the injury

---

### 📊 Progress

| Table | What it stores | Key columns |
|---|---|---|
| `member_progress` | Weight + measurements. User-scoped (same across all gyms). | `user_id`, `logged_at`, `weight_kg`, `body_fat_pct`, `chest_cm`, `waist_cm` |
| `progress_photos` | Front/back/side body photos. Workspace-scoped. | `workspace_member_id`, `photo_url`, `type`, `logged_at` |

> `member_progress` uses `user_id` (not `workspace_member_id`) because a person's weight doesn't change per gym.
> `progress_photos` uses `workspace_member_id` because Gym A trainer shouldn't see Gym B photos.

---

### 🔔 Notifications

| Table | What it stores | Key columns |
|---|---|---|
| `notifications` | In-app, WhatsApp, SMS, email notifications. | `user_id`, `workspace_id`, `type`, `title`, `body`, `channel`, `read` |

**When to insert here:**
- Fee due → `fee_due`
- Workout plan assigned → `plan_assigned`
- Session missed → `missed_session`
- Injury flagged → `injury_flagged`
- Payment confirmed → `payment_confirmed`

---

### ⚙️ Settings + Hardware (v2 — build last)

| Table | What it stores |
|---|---|
| `workspace_payment_configs` | Gym's Razorpay/Stripe keys |
| `payment_webhooks_raw` | Raw webhook payloads (save before processing) |
| `payment_events` | Normalised payment events |
| `zkteco_device_commands` | Commands to biometric devices |
| `zkteco_raw_logs` | Raw data from biometric devices |

---

## The Ownership Chain

```
users
└── workspace_members          (role: owner / manager / receptionist / trainer / member)
      │
      │  ← workspace-scoped data (linked by workspace_member_id)
      ├── attendance
      ├── member_subscriptions → member_payments
      ├── member_injuries
      ├── food_logs → food_log_photos
      ├── progress_photos
      │
      │  ← plan data (linked by member_id and workspace_id)
      ├── workout_plans → workout_sessions → workout_exercises
      │     └── workout_logs → workout_set_logs
      │     └── workout_session_overrides → workout_exercise_overrides
      │     └── workout_session_skips
      │     └── plan_pauses
      ├── diet_plans → diet_meals → diet_items → diet_item_photos
      │
      │  ← user-scoped data (linked by user_id, no workspace)
      └── member_progress

workspaces
├── branches
├── workspace_subscriptions → workspace_payments
├── membership_plans
├── exercise_library
└── notifications
```

---

## RLS in Plain English

Every query is automatically filtered. You never need to manually enforce these in code — Supabase does it. But know the rules:

| Table | Who can read | Who can write |
|---|---|---|
| `workspace_members` | Own rows + same-workspace staff | owner/manager/receptionist |
| `workout_plans` | trainer/owner (workspace) + member (own) | trainer/owner |
| `workout_logs` | trainer/owner (workspace) + member (own) | member (own) |
| `workout_set_logs` | trainer/owner (workspace) + member (own) | member (own) |
| `diet_plans` | trainer/owner (workspace) + member (own) | trainer/owner |
| `food_logs` | trainer/owner (workspace) + member (own) | member (create) + trainer (add notes) |
| `attendance` | staff (workspace) + member (own) | member (own) + staff (manual mark) |
| `member_progress` | own + trainers in shared workspace | own only |
| `progress_photos` | own + trainers in same workspace | own only |
| `member_injuries` | own + trainers/owner in same workspace | member (own) + trainer/owner |
| `notifications` | own only | — (server-side inserts) |

---

## Common Mistakes to Avoid

**❌ Querying without workspace_id scope**
```typescript
// Wrong — returns data from ALL workspaces
const { data } = await supabase.from('workout_plans').select('*').eq('member_id', userId)

// Correct — always scope to active workspace
const { data } = await supabase.from('workout_plans').select('*')
  .eq('workspace_id', workspaceId)
  .eq('member_id', userId)
```

**❌ Using users.id when you need workspace_members.id**
```
users.id       — the person's global identity (never changes)
workspace_members.id — the person's identity within ONE workspace
```
Most things (attendance, food_logs, subscriptions, progress_photos, injuries) use `workspace_member_id`.
Only `member_progress` uses `user_id` directly (because weight is global, not per-gym).

**❌ Querying diet_plans without checking plan_variant**
A member can have up to 3 active diet plans (default + training_day + rest_day).
Always pick the right one based on whether they checked in today.

**❌ Forgetting plan_pauses before running missed-session logic**
Before marking a session as `absent` in `workout_session_skips`, check if the plan has an active pause in `plan_pauses`. If `resumed_at` is null, the plan is still paused — skip the cron logic entirely.

**❌ Using `supabase.auth.getSession()` in server-side code**
Always use `supabase.auth.getUser()` in API routes and server components.
`getSession()` trusts the browser's JWT without verifying it with Supabase.
