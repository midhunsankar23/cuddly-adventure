# FitDesk — Database Schema

All tables in order of dependency.
Run migrations in numbered order.
Never modify an existing migration — add a new one.

---

## Migration 0001 — Core Identity + Workspaces

```sql
-- users
create table users (
  id              uuid primary key default gen_random_uuid(),
  phone           text unique,
  email           text unique,
  full_name       text not null,
  avatar_url      text,
  created_at      timestamptz default now(),
  constraint users_contact_check check (phone is not null or email is not null),
  constraint users_phone_e164    check (phone is null or phone ~ '^\+[1-9]\d{7,14}$')
  -- phone must be E.164 format: +919847001234 (country code required)
);

alter table users enable row level security;

create policy "users can view own profile"
on users for select
using (auth.uid() = id);

create policy "users can update own profile"
on users for update
using (auth.uid() = id);


-- workspaces
create table workspaces (
  id               uuid primary key default gen_random_uuid(),
  name             text not null,
  type             text not null check (type in ('gym', 'freelancer')),
  logo_url         text,
  owner_id         uuid not null references users(id),
  is_active        boolean default true,
  default_currency text not null default 'INR',
  -- ISO 4217: 'INR', 'USD', 'AED', 'GBP', 'EUR' etc.
  -- This is the currency the gym bills their members in.
  country_code     text,
  -- ISO 3166-1 alpha-2: 'IN', 'AE', 'US', 'GB' etc.
  created_at       timestamptz default now()
);

alter table workspaces enable row level security;

create policy "workspace members can view workspace"
on workspaces for select
using (
  id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and status = 'active'
  )
);

create policy "owners can update their workspace"
on workspaces for update
using (owner_id = auth.uid());


-- branches
create table branches (
  id                  uuid primary key default gen_random_uuid(),
  workspace_id        uuid not null references workspaces(id) on delete cascade,
  name                text not null,
  address             text,
  qr_code             text unique default gen_random_uuid()::text,
  is_active           boolean default true,
  latitude            numeric,
  longitude           numeric,
  timezone            text not null default 'Asia/Kolkata',
  -- IANA timezone: 'Asia/Kolkata', 'Asia/Dubai', 'America/New_York' etc.
  -- Used by cron jobs to determine "today" for this branch.
  device_serial       text,
  device_model        text,
  device_connected_at timestamptz,
  created_at          timestamptz default now()
);

alter table branches enable row level security;

create policy "workspace members can view branches"
on branches for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and status = 'active'
  )
);

create policy "owners and managers can manage branches"
on branches for all
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager')
    and status = 'active'
  )
);


-- workspace_members
-- THE central table. Every user-workspace relationship lives here.
create table workspace_members (
  id                  uuid primary key default gen_random_uuid(),
  workspace_id        uuid not null references workspaces(id) on delete cascade,
  branch_id           uuid references branches(id) on delete set null,
  user_id             uuid not null references users(id) on delete cascade,
  role                text not null check (role in ('owner','manager','receptionist','trainer','member')),
  status              text not null default 'active' check (status in ('pending','active','suspended')),
  joined_at           timestamptz default now(),
  invited_by          uuid references users(id),
  zkteco_user_id      text,
  zkteco_enrolled     boolean default false,
  zkteco_enrolled_at  timestamptz
);

alter table workspace_members enable row level security;

create unique index workspace_members_unique
  on workspace_members(workspace_id, user_id, role);

create index idx_workspace_members_user      on workspace_members(user_id);
create index idx_workspace_members_workspace on workspace_members(workspace_id);

create policy "users can view their own memberships"
on workspace_members for select
using (user_id = auth.uid());

create policy "workspace staff can view members in same workspace"
on workspace_members for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager', 'receptionist', 'trainer')
    and status = 'active'
  )
);

create policy "owner manager receptionist can create members"
on workspace_members for insert
with check (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager', 'receptionist')
    and status = 'active'
  )
);

create policy "owner and manager can update member status"
on workspace_members for update
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager')
    and status = 'active'
  )
);
```

---

## Migration 0002 — Subscriptions + Billing

```sql
-- subscription_plans
-- Seeded by you once. Gym owners and freelancers subscribe to these.
create table subscription_plans (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  type          text not null check (type in ('gym', 'freelancer')),
  max_members   int not null default -1,  -- -1 = unlimited
  max_branches  int not null default 1,   -- -1 = unlimited
  is_active     boolean default true,
  created_at    timestamptz default now()
);

alter table subscription_plans enable row level security;

create policy "anyone can view active plans"
on subscription_plans for select
using (is_active = true);


-- subscription_plan_prices
-- One row per (plan, currency). Supports INR, USD, AED, GBP etc.
-- When you add a new currency, insert rows here — no schema change needed.
create table subscription_plan_prices (
  id            uuid primary key default gen_random_uuid(),
  plan_id       uuid not null references subscription_plans(id) on delete cascade,
  currency      text not null,             -- 'INR', 'USD', 'AED' etc.
  price_monthly int not null,              -- in smallest unit: paise, cents, fils
  is_active     boolean default true,
  created_at    timestamptz default now(),
  unique(plan_id, currency)
);

alter table subscription_plan_prices enable row level security;

create policy "anyone can view active prices"
on subscription_plan_prices for select
using (is_active = true);


-- workspace_subscriptions
-- Which FitDesk plan a gym/trainer is currently on.
create table workspace_subscriptions (
  id                     uuid primary key default gen_random_uuid(),
  workspace_id           uuid not null references workspaces(id) on delete cascade,
  plan_id                uuid not null references subscription_plans(id),
  currency               text not null default 'INR',
  status                 text not null default 'trial'
                           check (status in ('trial','active','expired','cancelled')),
  trial_ends_at          timestamptz,
  current_period_ends_at timestamptz,
  created_at             timestamptz default now()
);

alter table workspace_subscriptions enable row level security;

create policy "owners can view their subscription"
on workspace_subscriptions for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and role = 'owner'
  )
);


-- workspace_payments
-- FitDesk billing records — gym/trainer paying FitDesk.
create table workspace_payments (
  id                  uuid primary key default gen_random_uuid(),
  workspace_id        uuid not null references workspaces(id) on delete cascade,
  subscription_id     uuid references workspace_subscriptions(id),
  amount              int not null,
  currency            text not null default 'INR',
  status              text not null default 'pending'
                        check (status in ('pending','paid','failed','refunded')),
  payment_provider    text not null default 'manual',
  provider_order_id   text,
  provider_payment_id text,
  provider_response   jsonb,
  paid_at             timestamptz,
  created_at          timestamptz default now()
);

alter table workspace_payments enable row level security;

create policy "owners can view their payments"
on workspace_payments for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and role = 'owner'
  )
);


-- membership_plans
-- Plans a gym creates for their own members: Monthly ₹1500, Quarterly ₹4000 etc.
create table membership_plans (
  id            uuid primary key default gen_random_uuid(),
  workspace_id  uuid not null references workspaces(id) on delete cascade,
  name          text not null,
  duration_days int not null,
  price         int not null,    -- in smallest currency unit (paise, cents, fils)
  currency      text not null default 'INR',
  is_active     boolean default true,
  created_at    timestamptz default now()
);

alter table membership_plans enable row level security;

create policy "workspace staff can view membership plans"
on membership_plans for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and status = 'active'
  )
);

create policy "owners and managers can manage membership plans"
on membership_plans for all
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager')
    and status = 'active'
  )
);


-- member_subscriptions
-- A member enrolled in a specific gym membership plan.
create table member_subscriptions (
  id                  uuid primary key default gen_random_uuid(),
  workspace_member_id uuid not null references workspace_members(id) on delete cascade,
  plan_id             uuid not null references membership_plans(id),
  start_date          date not null,
  end_date            date not null,
  status              text not null default 'active'
                        check (status in ('active','expired','pending')),
  payment_status      text not null default 'unpaid'
                        check (payment_status in ('paid','unpaid','partial')),
  created_at          timestamptz default now()
);

alter table member_subscriptions enable row level security;

create index idx_member_subscriptions_status on member_subscriptions(status, end_date);

create policy "staff can view member subscriptions"
on member_subscriptions for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'receptionist', 'trainer')
    and wm1.status = 'active'
  )
);

create policy "members can view own subscription"
on member_subscriptions for select
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "staff can create and update subscriptions"
on member_subscriptions for all
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'receptionist')
    and wm1.status = 'active'
  )
);


-- member_payments
-- Payment records for a member paying the gym.
create table member_payments (
  id                     uuid primary key default gen_random_uuid(),
  member_subscription_id uuid not null references member_subscriptions(id) on delete cascade,
  amount                 int not null,
  currency               text not null default 'INR',
  status                 text not null default 'pending'
                           check (status in ('pending','paid','failed','refunded')),
  payment_provider       text not null default 'manual',
  provider_order_id      text,
  provider_payment_id    text,
  provider_response      jsonb,
  screenshot_url         text,         -- member uploads screenshot of UPI/cash payment
  paid_at                timestamptz,
  marked_paid_by         uuid references users(id),
  created_at             timestamptz default now()
);

alter table member_payments enable row level security;

create policy "staff can view member payments"
on member_payments for select
using (
  member_subscription_id in (
    select ms.id from member_subscriptions ms
    join workspace_members wm on ms.workspace_member_id = wm.id
    where wm.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'receptionist')
      and status = 'active'
    )
  )
);

create policy "members can view own payments"
on member_payments for select
using (
  member_subscription_id in (
    select id from member_subscriptions
    where workspace_member_id in (
      select id from workspace_members where user_id = auth.uid()
    )
  )
);

create policy "staff can create and update payments"
on member_payments for all
using (
  member_subscription_id in (
    select ms.id from member_subscriptions ms
    join workspace_members wm on ms.workspace_member_id = wm.id
    where wm.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'receptionist')
      and status = 'active'
    )
  )
);
```

---

## Migration 0003 — Exercise Library + Workout Plans

```sql
-- exercise_library
-- Videos/exercises uploaded by trainer or provided by platform.
-- workspace_id = null means it's a platform exercise (available everywhere).
create table exercise_library (
  id            uuid primary key default gen_random_uuid(),
  workspace_id  uuid references workspaces(id) on delete cascade,
  created_by    uuid references users(id),
  name          text not null,
  muscle_group  text,
  equipment     text,
  video_url     text,
  thumbnail_url text,
  is_platform   boolean default false,
  created_at    timestamptz default now()
);

alter table exercise_library enable row level security;

create policy "view platform and own workspace exercises"
on exercise_library for select
using (
  is_platform = true
  or workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and status = 'active'
  )
);

create policy "trainers and owners can create exercises"
on exercise_library for insert
with check (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'trainer')
    and status = 'active'
  )
);

create policy "creators can update their exercises"
on exercise_library for update
using (created_by = auth.uid());


-- workout_plans
-- One plan per member per workspace (only one active at a time).
create table workout_plans (
  id                       uuid primary key default gen_random_uuid(),
  workspace_id             uuid not null references workspaces(id) on delete cascade,
  trainer_id               uuid not null references users(id),
  member_id                uuid not null references users(id),
  name                     text not null,
  schedule_type            text not null default 'sequential'
                             check (schedule_type in ('sequential')),
  -- calendar mode is a separate module built later.
  -- only 'sequential' exists in v1.
  total_weeks              int,
  -- null = no fixed end, plan loops forever (PPL etc.)
  -- set  = program has N weeks then ends
  plan_end_behaviour       text not null default 'loop'
                             check (plan_end_behaviour in ('loop', 'end')),
  -- loop: PPL, 5-day split — repeats forever after last session
  -- end:  12-week program — stops at last session, trainer assigns next plan
  missed_session_behaviour text not null default 'push_forward'
                             check (missed_session_behaviour in (
                               'push_forward',    -- missed = shown again next visit
                               'skip_and_continue', -- missed = dropped, move on
                               'trainer_decides'  -- missed = trainer notified, manual decision
                             )),
  current_session_index    int default 0,
  -- tracks where member is in the sequence.
  -- advances only when member marks session complete.
  is_active                boolean default true,
  deactivated_at           timestamptz,
  deactivated_by           uuid references users(id),
  switch_reason            text,
  created_at               timestamptz default now()
);

alter table workout_plans enable row level security;

-- only one active plan per member per workspace
create unique index workout_plans_one_active
  on workout_plans(workspace_id, member_id)
  where is_active = true;

create index idx_workout_plans_member on workout_plans(member_id, is_active);

create policy "trainers and owners can view plans in workspace"
on workout_plans for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager', 'trainer')
    and status = 'active'
  )
);

create policy "members can view own plans"
on workout_plans for select
using (member_id = auth.uid());

create policy "trainers and owners can create plans"
on workout_plans for insert
with check (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'trainer')
    and status = 'active'
  )
);

create policy "trainers and owners can update plans"
on workout_plans for update
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'trainer')
    and status = 'active'
  )
);


-- workout_sessions
-- Individual sessions within a plan. "Push day", "Pull day", "Legs day".
-- session_index drives the plan pointer — it is always a flat sequence.
-- week_number and day_in_week are display helpers only.
create table workout_sessions (
  id            uuid primary key default gen_random_uuid(),
  plan_id       uuid not null references workout_plans(id) on delete cascade,
  session_index int not null,
  -- 0-based global index within the plan.
  -- for a 12-week plan: Week 1 Day 1 = index 0, Week 1 Day 2 = index 1, etc.
  week_number   int,
  -- display only: show "Week 3" to member. null = no week grouping.
  day_in_week   int,
  -- display only: show "Day 2". null = no day grouping.
  label         text not null,     -- "Push Day", "Back + Biceps", "Full Body"
  rest_day      boolean default false,
  notes         text
);

alter table workout_sessions enable row level security;

-- session_index must be unique within a plan
create unique index workout_sessions_index_unique
  on workout_sessions(plan_id, session_index);

create policy "view sessions if can view plan"
on workout_sessions for select
using (
  plan_id in (select id from workout_plans)
);

create policy "trainers and owners can manage sessions"
on workout_sessions for all
using (
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- workout_exercises (the template)
-- Exercises inside a session: Squat 4×10 @ 80kg, Bench 3×8 etc.
create table workout_exercises (
  id           uuid primary key default gen_random_uuid(),
  session_id   uuid not null references workout_sessions(id) on delete cascade,
  exercise_id  uuid not null references exercise_library(id),
  order_index  int not null,
  sets         int,
  reps         int,
  weight_kg    numeric,
  rest_seconds int,
  notes        text
);

alter table workout_exercises enable row level security;

create policy "view exercises if can view session"
on workout_exercises for select
using (
  session_id in (select id from workout_sessions)
);

create policy "trainers and owners can manage exercises"
on workout_exercises for all
using (
  session_id in (
    select ws.id from workout_sessions ws
    join workout_plans wp on ws.plan_id = wp.id
    where wp.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- member_injuries
-- Tracks body part pain/injuries for a member.
-- Placed here because workout_session_overrides and workout_set_logs reference it.
create table member_injuries (
  id                  uuid primary key default gen_random_uuid(),
  workspace_member_id uuid not null references workspace_members(id) on delete cascade,
  body_part           text not null,
  -- examples: 'left_knee', 'right_shoulder', 'lower_back', 'neck',
  --           'left_elbow', 'right_wrist', 'right_knee', 'other'
  severity            text not null default 'mild'
                        check (severity in ('mild', 'moderate', 'severe')),
  reported_by         text not null check (reported_by in ('member', 'trainer')),
  reported_by_user_id uuid not null references users(id),
  started_at          date not null default current_date,
  resolved_at         date,      -- null = still active
  resolved_by         uuid references users(id),
  notes               text,
  created_at          timestamptz default now()
);

alter table member_injuries enable row level security;

-- fast lookup: show active injuries on member profile
create index idx_member_injuries_active
  on member_injuries(workspace_member_id)
  where resolved_at is null;

create policy "members can view own injuries"
on member_injuries for select
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "trainers can view member injuries in workspace"
on member_injuries for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'trainer')
    and wm1.status = 'active'
  )
);

create policy "members and trainers can create injuries"
on member_injuries for insert
with check (
  -- member flagging their own pain
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
  or
  -- trainer logging an injury for a member in their workspace
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'trainer')
    and wm1.status = 'active'
  )
);

create policy "members and trainers can resolve injuries"
on member_injuries for update
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
  or
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'trainer')
    and wm1.status = 'active'
  )
);


-- workout_session_overrides
-- Trainer swaps a session for one member on one specific date.
-- The plan template is untouched. Only that date is affected.
create table workout_session_overrides (
  id            uuid primary key default gen_random_uuid(),
  plan_id       uuid not null references workout_plans(id) on delete cascade,
  member_id     uuid not null references users(id),
  session_index int not null,
  override_date date not null,
  reason        text,
  injury_id     uuid references member_injuries(id),
  -- if this override was triggered by an injury, link it here for the audit trail
  created_by    uuid not null references users(id),
  created_at    timestamptz default now()
);

alter table workout_session_overrides enable row level security;

-- only one override per member per date
create unique index workout_overrides_unique
  on workout_session_overrides(plan_id, member_id, override_date);

create policy "members can view own overrides"
on workout_session_overrides for select
using (member_id = auth.uid());

create policy "trainers can view overrides in workspace"
on workout_session_overrides for select
using (
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'trainer')
      and status = 'active'
    )
  )
);

create policy "trainers can manage overrides"
on workout_session_overrides for all
using (
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- workout_exercise_overrides
-- The replacement exercises for a session override.
create table workout_exercise_overrides (
  id           uuid primary key default gen_random_uuid(),
  override_id  uuid not null references workout_session_overrides(id) on delete cascade,
  exercise_id  uuid not null references exercise_library(id),
  order_index  int not null,
  sets         int,
  reps         int,
  weight_kg    numeric,
  rest_seconds int,
  notes        text
);

alter table workout_exercise_overrides enable row level security;

create policy "view exercise overrides if can view override"
on workout_exercise_overrides for select
using (
  override_id in (select id from workout_session_overrides)
);

create policy "trainers can manage exercise overrides"
on workout_exercise_overrides for all
using (
  override_id in (
    select wo.id from workout_session_overrides wo
    join workout_plans wp on wo.plan_id = wp.id
    where wp.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- workout_session_skips
-- Records when a session was missed or intentionally skipped.
create table workout_session_skips (
  id             uuid primary key default gen_random_uuid(),
  plan_id        uuid not null references workout_plans(id) on delete cascade,
  session_id     uuid not null references workout_sessions(id),
  member_id      uuid not null references users(id),
  scheduled_date date not null,
  skip_reason    text check (skip_reason in (
                   'absent',            -- cron: no check-in detected that day
                   'trainer_override',  -- trainer changed the session
                   'rest_day_swap',     -- trainer marked as rest
                   'member_skip'        -- member explicitly said "skip today"
                 )),
  created_at     timestamptz default now()
);

alter table workout_session_skips enable row level security;

create policy "members can view own skips"
on workout_session_skips for select
using (member_id = auth.uid());

create policy "trainers can view skips in workspace"
on workout_session_skips for select
using (
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'trainer')
      and status = 'active'
    )
  )
);

create policy "members and trainers can create skips"
on workout_session_skips for insert
with check (
  member_id = auth.uid()
  or
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- plan_pauses
-- Tracks when a plan was paused (vacation, injury break).
-- While paused, the missed session cron ignores this member entirely.
-- On resume, member picks up at the exact same session_index.
create table plan_pauses (
  id          uuid primary key default gen_random_uuid(),
  plan_id     uuid not null references workout_plans(id) on delete cascade,
  paused_by   uuid not null references users(id),
  paused_at   date not null,
  resumed_at  date,   -- null = still paused
  reason      text,   -- 'vacation', 'injury', 'personal'
  created_at  timestamptz default now()
);

alter table plan_pauses enable row level security;

create policy "members can view own plan pauses"
on plan_pauses for select
using (
  plan_id in (
    select id from workout_plans where member_id = auth.uid()
  )
);

create policy "trainers can view plan pauses in workspace"
on plan_pauses for select
using (
  plan_id in (
    select id from workout_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'trainer')
      and status = 'active'
    )
  )
);

create policy "members and trainers can manage plan pauses"
on plan_pauses for all
using (
  plan_id in (
    select id from workout_plans
    where member_id = auth.uid()
    or workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- workout_logs
-- One row per session completed by a member.
create table workout_logs (
  id                  uuid primary key default gen_random_uuid(),
  plan_id             uuid not null references workout_plans(id),
  session_id          uuid references workout_sessions(id),
  -- null = off-plan workout (member did something not in their plan)
  member_id           uuid not null references users(id),
  workspace_member_id uuid not null references workspace_members(id),
  logged_date         date not null default current_date,
  started_at          timestamptz,
  completed_at        timestamptz,
  was_override        boolean default false,
  pain_notes          text,
  -- member can note how their body felt: "left knee felt unstable during lunges"
  notes               text,
  created_at          timestamptz default now()
);

alter table workout_logs enable row level security;

create index idx_workout_logs_member on workout_logs(member_id, logged_date desc);

create policy "members can manage own workout logs"
on workout_logs for all
using (member_id = auth.uid());

create policy "trainers can view workout logs in workspace"
on workout_logs for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'trainer')
    and wm1.status = 'active'
  )
);


-- workout_set_logs
-- The actual sets logged within a session: 10 reps @ 80kg.
create table workout_set_logs (
  id                  uuid primary key default gen_random_uuid(),
  log_id              uuid not null references workout_logs(id) on delete cascade,
  exercise_id         uuid not null references exercise_library(id),
  -- what they actually did
  planned_exercise_id uuid references exercise_library(id),
  -- null  = they followed the plan (did the assigned exercise)
  -- set   = they swapped. this is what WAS planned.
  swap_note           text,
  -- member's reason for swapping: "knee hurt, did leg press instead"
  -- no permission needed — member just logs what they did
  set_number          int not null,
  reps_done           int,
  weight_done_kg      numeric,
  rpe                 int check (rpe between 1 and 10),
  skipped             boolean default false,
  skip_reason         text,    -- 'pain', 'equipment_unavailable', 'trainer_instruction'
  injury_id           uuid references member_injuries(id),
  -- if skipped or swapped due to an injury, link to it
  notes               text
);

alter table workout_set_logs enable row level security;

create policy "members can manage own set logs"
on workout_set_logs for all
using (
  log_id in (
    select id from workout_logs where member_id = auth.uid()
  )
);

create policy "trainers can view set logs in workspace"
on workout_set_logs for select
using (
  log_id in (
    select wl.id from workout_logs wl
    join workout_plans wp on wl.plan_id = wp.id
    where wp.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'manager', 'trainer')
      and status = 'active'
    )
  )
);
```

---

## Migration 0004 — Diet Plans

```sql
-- diet_plans
-- A structured diet assigned to one member by one trainer.
-- plan_variant allows a member to have a training-day and rest-day diet simultaneously.
create table diet_plans (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  trainer_id      uuid not null references users(id),
  member_id       uuid not null references users(id),
  name            text not null,
  total_calories  int,
  plan_variant    text not null default 'default'
                    check (plan_variant in ('default', 'training_day', 'rest_day', 'custom')),
  -- default:      standard plan, shown every day
  -- training_day: shown on days the member checks in
  -- rest_day:     shown on days the member does not check in
  -- custom:       manually activated by trainer or member
  is_active       boolean default true,
  deactivated_at  timestamptz,
  deactivated_by  uuid references users(id),
  switch_reason   text,
  created_at      timestamptz default now()
);

alter table diet_plans enable row level security;

-- one active plan per member per variant per workspace
create unique index diet_plans_one_active_per_variant
  on diet_plans(workspace_id, member_id, plan_variant)
  where is_active = true;

create policy "trainers and owners can view diet plans in workspace"
on diet_plans for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'manager', 'trainer')
    and status = 'active'
  )
);

create policy "members can view own diet plans"
on diet_plans for select
using (member_id = auth.uid());

create policy "trainers and owners can create diet plans"
on diet_plans for insert
with check (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'trainer')
    and status = 'active'
  )
);

create policy "trainers and owners can update diet plans"
on diet_plans for update
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid()
    and role in ('owner', 'trainer')
    and status = 'active'
  )
);


-- diet_meals
-- Meals within the plan: Breakfast, Lunch, Dinner, Pre-workout snack.
create table diet_meals (
  id          uuid primary key default gen_random_uuid(),
  plan_id     uuid not null references diet_plans(id) on delete cascade,
  name        text not null,
  time_label  text,        -- "7:00 AM", "Before training"
  image_url   text,        -- optional cover photo of the full meal plate
  order_index int not null
);

alter table diet_meals enable row level security;

create policy "view meals if can view plan"
on diet_meals for select
using (
  plan_id in (select id from diet_plans)
);

create policy "trainers and owners can manage meals"
on diet_meals for all
using (
  plan_id in (
    select id from diet_plans
    where workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- diet_items
-- Food items within a meal: Oats 100g — 350 kcal — P:10 C:60 F:6.
create table diet_items (
  id          uuid primary key default gen_random_uuid(),
  meal_id     uuid not null references diet_meals(id) on delete cascade,
  name        text not null,
  quantity    text,       -- "100g", "1 cup", "2 pieces"
  calories    int,
  protein_g   numeric,
  carbs_g     numeric,
  fat_g       numeric,
  image_url   text,       -- single reference photo (backward compat)
  notes       text,       -- "eat this 30 min before training"
  order_index int not null
);

alter table diet_items enable row level security;

create policy "view items if can view meal"
on diet_items for select
using (
  meal_id in (select id from diet_meals)
);

create policy "trainers and owners can manage items"
on diet_items for all
using (
  meal_id in (
    select m.id from diet_meals m
    join diet_plans p on m.plan_id = p.id
    where p.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- diet_item_photos
-- Multiple reference photos per food item (trainer uploads).
-- Use when you need more than one photo for the same food item.
create table diet_item_photos (
  id          uuid primary key default gen_random_uuid(),
  item_id     uuid not null references diet_items(id) on delete cascade,
  photo_url   text not null,
  order_index int not null default 0,
  created_at  timestamptz default now()
);

alter table diet_item_photos enable row level security;

create policy "view item photos if can view item"
on diet_item_photos for select
using (
  item_id in (select id from diet_items)
);

create policy "trainers and owners can manage item photos"
on diet_item_photos for all
using (
  item_id in (
    select di.id from diet_items di
    join diet_meals dm on di.meal_id = dm.id
    join diet_plans dp on dm.plan_id = dp.id
    where dp.workspace_id in (
      select workspace_id from workspace_members
      where user_id = auth.uid()
      and role in ('owner', 'trainer')
      and status = 'active'
    )
  )
);


-- food_logs
-- What the member actually ate throughout the day.
-- Completely separate from diet_plans (which is what the trainer prescribed).
-- No permission needed to create — member just logs it.
-- Trainer can view and leave a note. That's it.
create table food_logs (
  id                  uuid primary key default gen_random_uuid(),
  workspace_member_id uuid not null references workspace_members(id) on delete cascade,
  logged_date         date not null default current_date,
  meal_time           text check (meal_time in (
                        'breakfast', 'lunch', 'dinner',
                        'snack', 'pre_workout', 'post_workout', 'other'
                      )),
  description         text,
  -- member writes: "Rice and dal, smaller portion. No ghee today."
  trainer_note        text,
  -- trainer writes back: "Good. Cut the evening snack tomorrow."
  trainer_noted_by    uuid references users(id),
  trainer_noted_at    timestamptz,
  created_at          timestamptz default now()
);

alter table food_logs enable row level security;

create index idx_food_logs_member on food_logs(workspace_member_id, logged_date desc);

create policy "members can manage own food logs"
on food_logs for all
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "trainers can view food logs in workspace"
on food_logs for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'trainer')
    and wm1.status = 'active'
  )
);

create policy "trainers can add notes to food logs"
on food_logs for update
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'trainer')
    and wm1.status = 'active'
  )
);


-- food_log_photos
-- Photos the member uploads of their actual food. Multiple per log entry.
create table food_log_photos (
  id          uuid primary key default gen_random_uuid(),
  log_id      uuid not null references food_logs(id) on delete cascade,
  photo_url   text not null,
  order_index int not null default 0,
  created_at  timestamptz default now()
);

alter table food_log_photos enable row level security;

create policy "members can manage own food log photos"
on food_log_photos for all
using (
  log_id in (
    select id from food_logs
    where workspace_member_id in (
      select id from workspace_members where user_id = auth.uid()
    )
  )
);

create policy "trainers can view food log photos"
on food_log_photos for select
using (
  log_id in (select id from food_logs)
);
```

---

## Migration 0005 — Attendance + Progress

```sql
-- attendance
-- One check-in per member per branch per day.
create table attendance (
  id                  uuid primary key default gen_random_uuid(),
  workspace_member_id uuid not null references workspace_members(id) on delete cascade,
  branch_id           uuid references branches(id),
  checked_in_at       timestamptz default now(),
  method              text not null check (method in ('qr_scan', 'manual', 'biometric', 'gps')),
  marked_by           uuid references users(id)
  -- null = member checked in themselves
  -- set  = staff marked it manually
);

alter table attendance enable row level security;

-- prevents duplicate check-ins on the same day
create unique index attendance_once_per_day
  on attendance(workspace_member_id, branch_id, (checked_in_at::date));

create index idx_attendance_member
  on attendance(workspace_member_id, checked_in_at desc);

create policy "members can view own attendance"
on attendance for select
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "staff can view attendance in workspace"
on attendance for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'receptionist', 'trainer')
    and wm1.status = 'active'
  )
);

create policy "members can insert own attendance"
on attendance for insert
with check (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "staff can insert attendance for members"
on attendance for insert
with check (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'receptionist', 'trainer')
    and wm1.status = 'active'
  )
);


-- member_progress
-- Personal measurements: weight, body fat, body measurements.
-- User-scoped, NOT workspace-scoped.
-- A person's weight is the same whether they train at Gym A or Gym B.
create table member_progress (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  logged_at    date not null default current_date,
  weight_kg    numeric,
  body_fat_pct numeric,
  chest_cm     numeric,
  waist_cm     numeric,
  hips_cm      numeric,
  bicep_cm     numeric,
  thigh_cm     numeric,
  shoulder_cm  numeric,
  notes        text,
  created_at   timestamptz default now(),
  unique(user_id, logged_at)
);

alter table member_progress enable row level security;

create index idx_member_progress on member_progress(user_id, logged_at desc);

create policy "users can manage own progress"
on member_progress for all
using (user_id = auth.uid());

create policy "trainers can view their members progress"
on member_progress for select
using (
  user_id in (
    select wm_member.user_id
    from workspace_members wm_trainer
    join workspace_members wm_member on wm_trainer.workspace_id = wm_member.workspace_id
    where wm_trainer.user_id = auth.uid()
    and wm_trainer.role in ('owner', 'manager', 'trainer')
    and wm_trainer.status = 'active'
    and wm_member.role = 'member'
  )
);


-- progress_photos
-- Front/back/side body photos. Workspace-scoped.
-- Gym A trainer cannot see photos uploaded in context of Gym B.
create table progress_photos (
  id                  uuid primary key default gen_random_uuid(),
  workspace_member_id uuid not null references workspace_members(id) on delete cascade,
  photo_url           text not null,
  type                text check (type in ('front', 'back', 'side')),
  logged_at           date not null default current_date,
  created_at          timestamptz default now()
);

alter table progress_photos enable row level security;

create policy "members can manage own progress photos"
on progress_photos for all
using (
  workspace_member_id in (
    select id from workspace_members where user_id = auth.uid()
  )
);

create policy "trainers can view progress photos in workspace"
on progress_photos for select
using (
  workspace_member_id in (
    select wm2.id from workspace_members wm1
    join workspace_members wm2 on wm1.workspace_id = wm2.workspace_id
    where wm1.user_id = auth.uid()
    and wm1.role in ('owner', 'manager', 'trainer')
    and wm1.status = 'active'
  )
);
```

---

## Migration 0006 — Notifications + ZKTeco + Payment Events

```sql
-- notifications
create table notifications (
  id               uuid primary key default gen_random_uuid(),
  user_id          uuid not null references users(id) on delete cascade,
  workspace_id     uuid references workspaces(id) on delete cascade,
  type             text not null,
  -- 'fee_due' | 'workout_reminder' | 'plan_assigned' | 'plan_changed'
  -- 'attendance_marked' | 'payment_confirmed' | 'member_inactive'
  -- 'missed_session' | 'injury_flagged'
  title            text not null,
  body             text,
  channel          text default 'in_app'
                     check (channel in ('in_app', 'whatsapp', 'sms', 'email')),
  read             boolean default false,
  external_sent    boolean default false,
  external_sent_at timestamptz,
  external_error   text,
  sent_at          timestamptz,
  created_at       timestamptz default now()
);

alter table notifications enable row level security;

create index idx_notifications_user
  on notifications(user_id, read, created_at desc);

create policy "users can view own notifications"
on notifications for select
using (user_id = auth.uid());

create policy "users can mark own notifications read"
on notifications for update
using (user_id = auth.uid());


-- zkteco_device_commands
-- Commands sent from FitDesk to biometric device (v2).
create table zkteco_device_commands (
  id            uuid primary key default gen_random_uuid(),
  device_serial text not null,
  branch_id     uuid references branches(id),
  command_id    int not null,
  command_type  text check (command_type in ('enroll_user','delete_user','enroll_finger','sync_time')),
  zkteco_user_id text,
  member_name   text,
  status        text default 'pending'
                  check (status in ('pending','sent','confirmed','failed')),
  created_at    timestamptz default now(),
  sent_at       timestamptz,
  confirmed_at  timestamptz
);

alter table zkteco_device_commands enable row level security;

create policy "owners and managers can manage device commands"
on zkteco_device_commands for all
using (
  branch_id in (
    select b.id from branches b
    join workspace_members wm on b.workspace_id = wm.workspace_id
    where wm.user_id = auth.uid()
    and wm.role in ('owner', 'manager')
    and wm.status = 'active'
  )
);


-- zkteco_raw_logs
-- Raw attendance data from biometric device before processing.
-- Safety net — raw data is always preserved even if the processor has a bug.
create table zkteco_raw_logs (
  id           uuid primary key default gen_random_uuid(),
  branch_id    uuid references branches(id),
  device_serial text,
  payload      text,
  processed    boolean default false,
  processed_at timestamptz,
  error        text,
  received_at  timestamptz default now()
);

alter table zkteco_raw_logs enable row level security;

create policy "owners and managers can view raw logs"
on zkteco_raw_logs for select
using (
  branch_id in (
    select b.id from branches b
    join workspace_members wm on b.workspace_id = wm.workspace_id
    where wm.user_id = auth.uid()
    and wm.role in ('owner', 'manager')
    and wm.status = 'active'
  )
);


-- workspace_payment_configs
-- Gym's own Razorpay/Stripe keys (v2).
create table workspace_payment_configs (
  id           uuid primary key default gen_random_uuid(),
  workspace_id uuid not null references workspaces(id) on delete cascade,
  provider     text not null,
  is_active    boolean default true,
  country      text,
  config       jsonb,   -- encrypted provider keys
  created_at   timestamptz default now()
);

alter table workspace_payment_configs enable row level security;

create policy "only owners can manage payment configs"
on workspace_payment_configs for all
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and role = 'owner'
  )
);


-- payment_webhooks_raw
-- Raw webhook payloads saved before processing (v2).
-- Never lose a payment event even if the normaliser has a bug.
create table payment_webhooks_raw (
  id           uuid primary key default gen_random_uuid(),
  workspace_id uuid references workspaces(id),
  provider     text,
  event_type   text,
  payload      jsonb,
  received_at  timestamptz default now(),
  processed    boolean default false,
  processed_at timestamptz,
  error        text
);

alter table payment_webhooks_raw enable row level security;

create policy "owners can view their payment webhooks"
on payment_webhooks_raw for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and role = 'owner'
  )
);


-- payment_events
-- Normalised, provider-agnostic payment events (v2).
create table payment_events (
  id                  uuid primary key default gen_random_uuid(),
  workspace_id        uuid references workspaces(id),
  webhook_raw_id      uuid references payment_webhooks_raw(id),
  event_type          text check (event_type in (
                        'payment.success', 'payment.failed', 'payment.refunded',
                        'subscription.created', 'subscription.cancelled', 'subscription.renewed'
                      )),
  provider            text,
  provider_payment_id text,
  provider_order_id   text,
  amount              int,
  currency            text,
  entity_type         text check (entity_type in ('workspace_subscription', 'member_subscription')),
  entity_id           uuid,
  status              text check (status in ('success', 'failed', 'refunded', 'pending')),
  created_at          timestamptz default now()
);

alter table payment_events enable row level security;

create policy "owners can view their payment events"
on payment_events for select
using (
  workspace_id in (
    select workspace_id from workspace_members
    where user_id = auth.uid() and role = 'owner'
  )
);
```

---

## Migration 0007 — Seed Data

```sql
-- FitDesk SaaS subscription plans (run once, you manage these)
insert into subscription_plans (id, name, type, max_members, max_branches) values
  ('11111111-0000-0000-0000-000000000001', 'Trial',   'gym',        20,  1),
  ('11111111-0000-0000-0000-000000000002', 'Starter', 'gym',        100, 1),
  ('11111111-0000-0000-0000-000000000003', 'Growth',  'gym',        300, 3),
  ('11111111-0000-0000-0000-000000000004', 'Pro',     'gym',        -1,  -1),
  ('11111111-0000-0000-0000-000000000005', 'Trial',   'freelancer', 5,   1),
  ('11111111-0000-0000-0000-000000000006', 'Solo',    'freelancer', 20,  1),
  ('11111111-0000-0000-0000-000000000007', 'Pro',     'freelancer', -1,  1);

-- Prices per currency (add new currencies here without schema changes)
insert into subscription_plan_prices (plan_id, currency, price_monthly) values
  -- INR pricing (paise)
  ('11111111-0000-0000-0000-000000000001', 'INR', 0),       -- Trial free
  ('11111111-0000-0000-0000-000000000002', 'INR', 99900),   -- Starter ₹999
  ('11111111-0000-0000-0000-000000000003', 'INR', 249900),  -- Growth ₹2,499
  ('11111111-0000-0000-0000-000000000004', 'INR', 499900),  -- Pro ₹4,999
  ('11111111-0000-0000-0000-000000000005', 'INR', 0),       -- Freelancer Trial free
  ('11111111-0000-0000-0000-000000000006', 'INR', 29900),   -- Solo ₹299
  ('11111111-0000-0000-0000-000000000007', 'INR', 59900),   -- Pro ₹599

  -- USD pricing (cents) — add when launching internationally
  ('11111111-0000-0000-0000-000000000001', 'USD', 0),       -- Trial free
  ('11111111-0000-0000-0000-000000000002', 'USD', 1200),    -- Starter $12
  ('11111111-0000-0000-0000-000000000003', 'USD', 2900),    -- Growth $29
  ('11111111-0000-0000-0000-000000000004', 'USD', 5900),    -- Pro $59
  ('11111111-0000-0000-0000-000000000005', 'USD', 0),       -- Freelancer Trial free
  ('11111111-0000-0000-0000-000000000006', 'USD', 400),     -- Solo $4
  ('11111111-0000-0000-0000-000000000007', 'USD', 800);     -- Pro $8
```

---

## Key Relationships

```
users
└── workspace_members ──────────────────────────── (role + status per workspace)
      │
      ├── WORKSPACE
      │     └── branches                           (physical locations)
      │     └── workspace_subscriptions            (FitDesk plan)
      │         └── workspace_payments
      │     └── membership_plans                   (gym's own plans for members)
      │     └── exercise_library                   (trainer's videos)
      │     └── workout_plans ──────────────────── (assigned to one member)
      │         └── workout_sessions               (Push/Pull/Legs etc.)
      │             └── workout_exercises          (Squat 4×10 @ 80kg)
      │         └── workout_session_overrides      (trainer swaps one day)
      │             └── workout_exercise_overrides
      │         └── workout_session_skips          (missed or skipped)
      │         └── plan_pauses                    (vacation / injury break)
      │         └── workout_logs ──────────────────(member completed session)
      │             └── workout_set_logs           (actual sets: 10 reps @ 80kg)
      │     └── diet_plans ─────────────────────── (assigned to one member)
      │         └── diet_meals                     (Breakfast, Lunch...)
      │             └── diet_items                 (Oats 100g, macros)
      │                 └── diet_item_photos       (multiple reference photos)
      │
      ├── attendance                               (check-ins, one per day per branch)
      │
      ├── member_subscriptions                     (enrolled in membership plan)
      │   └── member_payments                      (payment records + screenshot)
      │
      ├── member_injuries                          (active body part pain)
      │   (linked from workout_set_logs.injury_id)
      │   (linked from workout_session_overrides.injury_id)
      │
      ├── food_logs                                (what member actually ate)
      │   └── food_log_photos                     (photos of actual food)
      │
      └── progress_photos                          (body photos, workspace-scoped)

users (personal — not workspace-scoped)
└── member_progress                               (weight, measurements, body fat)

branches
└── zkteco_device_commands                        (v2 biometric hardware)
└── zkteco_raw_logs                               (v2 raw attendance data)

notifications                                     (all channels: in-app, WhatsApp, SMS)
payment_webhooks_raw → payment_events             (v2 Razorpay/Stripe)
workspace_payment_configs                         (v2 provider keys)
subscription_plans → subscription_plan_prices     (FitDesk SaaS tiers, multi-currency)
```

---

## Changes from Original Schema

For reference — what was changed and why.

| Change | Why |
|---|---|
| Removed `workout_weeks` table | Unnecessary join table. `week_number` + `day_in_week` added directly to `workout_sessions` as display helpers. The flat `session_index` still drives the pointer. |
| Added `plan_end_behaviour` to `workout_plans` | Sequential plans now explicitly declare whether they loop forever (PPL) or end after N sessions (12-week program). |
| Added `member_skip` to `workout_session_skips.skip_reason` | Members can explicitly skip a session without trainer permission. |
| Added `pain_notes` to `workout_logs` | Member can note how their body felt during a session. |
| Added `planned_exercise_id` + `swap_note` + `injury_id` to `workout_set_logs` | Member can log "I did Leg Press instead of Squats — knee hurt" without any permission or approval. |
| Added `member_injuries` table | Structured tracking of body part pain. Active injuries visible on member profile. Linked to skips and overrides for audit trail. |
| Added `injury_id` to `workout_session_overrides` | Links a trainer override to the injury that caused it. |
| Added `plan_pauses` table | Members/trainers can pause a plan (vacation, injury). Cron ignores paused plans. Resumes at exact same session index. |
| Added `plan_variant` to `diet_plans` | Supports training-day vs rest-day diets simultaneously. One active plan per variant per member. |
| Added `image_url` to `diet_meals` | Cover photo of the full meal plate (not just per-item). |
| Added `diet_item_photos` table | Multiple reference photos per food item. |
| Added `food_logs` + `food_log_photos` tables | Member logs what they actually ate (vs prescribed plan). Text + multiple photos. Trainer can leave a note. No permission needed to create. |
| Added `default_currency` + `country_code` to `workspaces` | International support. Gyms outside India can set AED, USD, GBP etc. |
| Added `currency` to `membership_plans` | Gym bills members in their workspace currency. |
| Added `subscription_plan_prices` table | FitDesk SaaS tiers now have per-currency pricing. Add a new currency without schema changes. |
| Added `timezone` to `branches` | Cron jobs (missed sessions, fee reminders) now calculate "today" correctly for each branch's local time. |
| Added E.164 phone constraint to `users` | International phone numbers stored consistently: +919847001234 not 9847001234. |
| Added all missing RLS policies | `diet_plans`, `diet_meals`, `diet_items`, `member_payments`, `progress_photos`, `workout_logs`, `workout_set_logs`, `workout_session_skips`, `workout_exercise_overrides`, `plan_pauses`, `member_injuries`, `zkteco_*`, `payment_webhooks_raw`, `payment_events` — all had RLS enabled but no policies. Now complete. |
```
