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
  constraint users_contact_check check (phone is not null or email is not null)
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
  id              uuid primary key default gen_random_uuid(),
  name            text not null,
  type            text not null check (type in ('gym', 'freelancer')),
  logo_url        text,
  owner_id        uuid not null references users(id),
  is_active       boolean default true,
  created_at      timestamptz default now()
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
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  name            text not null,
  address         text,
  qr_code         text unique default gen_random_uuid()::text,
  is_active       boolean default true,
  latitude        numeric,
  longitude       numeric,
  device_serial   text,
  device_model    text,
  device_connected_at timestamptz,
  created_at      timestamptz default now()
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
create table workspace_members (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  branch_id       uuid references branches(id) on delete set null,
  user_id         uuid not null references users(id) on delete cascade,
  role            text not null check (role in ('owner','manager','receptionist','trainer','member')),
  status          text not null default 'active' check (status in ('pending','active','suspended')),
  joined_at       timestamptz default now(),
  invited_by      uuid references users(id),
  zkteco_user_id  text,
  zkteco_enrolled boolean default false,
  zkteco_enrolled_at timestamptz
);

alter table workspace_members enable row level security;

create unique index workspace_members_unique
  on workspace_members(workspace_id, user_id, role);

create index idx_workspace_members_user on workspace_members(user_id);
create index idx_workspace_members_workspace on workspace_members(workspace_id);

create policy "users can view their own memberships"
on workspace_members for select
using (user_id = auth.uid());

create policy "workspace members can view other members in same workspace"
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
-- subscription_plans (seeded by you, not created by users)
create table subscription_plans (
  id              uuid primary key default gen_random_uuid(),
  name            text not null,
  type            text not null check (type in ('gym', 'freelancer')),
  max_members     int not null default -1,
  max_branches    int not null default 1,
  price_monthly   int not null,
  is_active       boolean default true,
  created_at      timestamptz default now()
);

alter table subscription_plans enable row level security;

create policy "anyone can view active plans"
on subscription_plans for select
using (is_active = true);

-- workspace_subscriptions
create table workspace_subscriptions (
  id                      uuid primary key default gen_random_uuid(),
  workspace_id            uuid not null references workspaces(id) on delete cascade,
  plan_id                 uuid not null references subscription_plans(id),
  status                  text not null default 'trial' check (status in ('trial','active','expired','cancelled')),
  trial_ends_at           timestamptz,
  current_period_ends_at  timestamptz,
  created_at              timestamptz default now()
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

-- workspace_payments (your SaaS billing — provider agnostic)
create table workspace_payments (
  id                    uuid primary key default gen_random_uuid(),
  workspace_id          uuid not null references workspaces(id) on delete cascade,
  subscription_id       uuid references workspace_subscriptions(id),
  amount                int not null,
  currency              text default 'INR',
  status                text not null default 'pending' check (status in ('pending','paid','failed','refunded')),
  payment_provider      text not null default 'manual',
  provider_order_id     text,
  provider_payment_id   text,
  provider_response     jsonb,
  paid_at               timestamptz,
  created_at            timestamptz default now()
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

-- membership_plans (gym's own plans for their members)
create table membership_plans (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  name            text not null,
  duration_days   int not null,
  price           int not null,
  is_active       boolean default true,
  created_at      timestamptz default now()
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
create table member_subscriptions (
  id                        uuid primary key default gen_random_uuid(),
  workspace_member_id       uuid not null references workspace_members(id) on delete cascade,
  plan_id                   uuid not null references membership_plans(id),
  start_date                date not null,
  end_date                  date not null,
  status                    text not null default 'active' check (status in ('active','expired','pending')),
  payment_status            text not null default 'unpaid' check (payment_status in ('paid','unpaid','partial')),
  created_at                timestamptz default now()
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

-- member_payments (provider agnostic)
create table member_payments (
  id                      uuid primary key default gen_random_uuid(),
  member_subscription_id  uuid not null references member_subscriptions(id) on delete cascade,
  amount                  int not null,
  currency                text default 'INR',
  status                  text not null default 'pending' check (status in ('pending','paid','failed','refunded')),
  payment_provider        text not null default 'manual',
  provider_order_id       text,
  provider_payment_id     text,
  provider_response       jsonb,
  screenshot_url          text,
  paid_at                 timestamptz,
  marked_paid_by          uuid references users(id),
  created_at              timestamptz default now()
);

alter table member_payments enable row level security;
```

---

## Migration 0003 — Exercise Library + Workout Plans

```sql
-- exercise_library
create table exercise_library (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid references workspaces(id) on delete cascade,
  created_by      uuid references users(id),
  name            text not null,
  muscle_group    text,
  equipment       text,
  video_url       text,
  thumbnail_url   text,
  is_platform     boolean default false,
  created_at      timestamptz default now()
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

-- workout_plans
create table workout_plans (
  id                        uuid primary key default gen_random_uuid(),
  workspace_id              uuid not null references workspaces(id) on delete cascade,
  trainer_id                uuid not null references users(id),
  member_id                 uuid not null references users(id),
  name                      text not null,
  schedule_type             text not null default 'sequential' check (schedule_type in ('sequential','calendar')),
  total_weeks               int,
  missed_session_behaviour  text not null default 'push_forward' check (
                              missed_session_behaviour in ('push_forward','skip_and_continue','trainer_decides')
                            ),
  current_session_index     int default 0,
  is_active                 boolean default true,
  deactivated_at            timestamptz,
  deactivated_by            uuid references users(id),
  switch_reason             text,
  created_at                timestamptz default now()
);

alter table workout_plans enable row level security;

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

-- workout_weeks
create table workout_weeks (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references workout_plans(id) on delete cascade,
  week_number     int not null,
  label           text
);

alter table workout_weeks enable row level security;

create unique index workout_weeks_number_unique
  on workout_weeks(plan_id, week_number);

-- inherit plan RLS via join — simplified policy
create policy "view weeks if can view plan"
on workout_weeks for select
using (
  plan_id in (select id from workout_plans)
);

-- workout_sessions
create table workout_sessions (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references workout_plans(id) on delete cascade,
  week_id         uuid references workout_weeks(id) on delete cascade,
  session_index   int not null,
  label           text not null,
  rest_day        boolean default false,
  notes           text
);

alter table workout_sessions enable row level security;

create unique index workout_sessions_index_unique
  on workout_sessions(plan_id, session_index);

create policy "view sessions if can view plan"
on workout_sessions for select
using (
  plan_id in (select id from workout_plans)
);

-- workout_exercises (template)
create table workout_exercises (
  id              uuid primary key default gen_random_uuid(),
  session_id      uuid not null references workout_sessions(id) on delete cascade,
  exercise_id     uuid not null references exercise_library(id),
  order_index     int not null,
  sets            int,
  reps            int,
  weight_kg       numeric,
  rest_seconds    int,
  notes           text
);

alter table workout_exercises enable row level security;

create policy "view exercises if can view session"
on workout_exercises for select
using (
  session_id in (select id from workout_sessions)
);

-- workout_session_overrides
create table workout_session_overrides (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references workout_plans(id) on delete cascade,
  member_id       uuid not null references users(id),
  session_index   int not null,
  override_date   date not null,
  reason          text,
  created_by      uuid not null references users(id),
  created_at      timestamptz default now()
);

alter table workout_session_overrides enable row level security;

create unique index workout_overrides_unique
  on workout_session_overrides(plan_id, member_id, override_date);

create table workout_exercise_overrides (
  id              uuid primary key default gen_random_uuid(),
  override_id     uuid not null references workout_session_overrides(id) on delete cascade,
  exercise_id     uuid not null references exercise_library(id),
  order_index     int not null,
  sets            int,
  reps            int,
  weight_kg       numeric,
  rest_seconds    int,
  notes           text
);

alter table workout_exercise_overrides enable row level security;

-- workout_session_skips
create table workout_session_skips (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references workout_plans(id) on delete cascade,
  session_id      uuid not null references workout_sessions(id),
  member_id       uuid not null references users(id),
  scheduled_date  date not null,
  skip_reason     text check (skip_reason in ('absent','trainer_override','rest_day_swap')),
  created_at      timestamptz default now()
);

alter table workout_session_skips enable row level security;

-- workout_logs
create table workout_logs (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references workout_plans(id),
  session_id      uuid references workout_sessions(id),
  member_id       uuid not null references users(id),
  workspace_member_id uuid not null references workspace_members(id),
  logged_date     date not null default current_date,
  started_at      timestamptz,
  completed_at    timestamptz,
  was_override    boolean default false,
  notes           text,
  created_at      timestamptz default now()
);

alter table workout_logs enable row level security;

create index idx_workout_logs_member on workout_logs(member_id, logged_date desc);

create table workout_set_logs (
  id              uuid primary key default gen_random_uuid(),
  log_id          uuid not null references workout_logs(id) on delete cascade,
  exercise_id     uuid not null references exercise_library(id),
  set_number      int not null,
  reps_done       int,
  weight_done_kg  numeric,
  rpe             int check (rpe between 1 and 10),
  notes           text
);

alter table workout_set_logs enable row level security;
```

---

## Migration 0004 — Diet Plans

```sql
create table diet_plans (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  trainer_id      uuid not null references users(id),
  member_id       uuid not null references users(id),
  name            text not null,
  total_calories  int,
  is_active       boolean default true,
  deactivated_at  timestamptz,
  deactivated_by  uuid references users(id),
  switch_reason   text,
  created_at      timestamptz default now()
);

alter table diet_plans enable row level security;

create unique index diet_plans_one_active
  on diet_plans(workspace_id, member_id)
  where is_active = true;

create table diet_meals (
  id              uuid primary key default gen_random_uuid(),
  plan_id         uuid not null references diet_plans(id) on delete cascade,
  name            text not null,
  time_label      text,
  order_index     int not null
);

alter table diet_meals enable row level security;

create table diet_items (
  id              uuid primary key default gen_random_uuid(),
  meal_id         uuid not null references diet_meals(id) on delete cascade,
  name            text not null,
  quantity        text,
  calories        int,
  protein_g       numeric,
  carbs_g         numeric,
  fat_g           numeric,
  image_url       text,
  notes           text,
  order_index     int not null
);

alter table diet_items enable row level security;
```

---

## Migration 0005 — Attendance + Progress

```sql
-- attendance
create table attendance (
  id                    uuid primary key default gen_random_uuid(),
  workspace_member_id   uuid not null references workspace_members(id) on delete cascade,
  branch_id             uuid references branches(id),
  checked_in_at         timestamptz default now(),
  method                text not null check (method in ('qr_scan','manual','biometric','gps')),
  marked_by             uuid references users(id)
);

alter table attendance enable row level security;

-- one check-in per member per branch per day
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
    and wm1.role in ('owner','manager','receptionist','trainer')
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
    and wm1.role in ('owner','manager','receptionist','trainer')
    and wm1.status = 'active'
  )
);

-- member_progress (user-scoped, not workspace-scoped)
create table member_progress (
  id                    uuid primary key default gen_random_uuid(),
  user_id               uuid not null references users(id) on delete cascade,
  logged_at             date not null default current_date,
  weight_kg             numeric,
  body_fat_pct          numeric,
  chest_cm              numeric,
  waist_cm              numeric,
  hips_cm               numeric,
  bicep_cm              numeric,
  thigh_cm              numeric,
  shoulder_cm           numeric,
  notes                 text,
  created_at            timestamptz default now(),
  unique(user_id, logged_at)
);

alter table member_progress enable row level security;

create index idx_member_progress
  on member_progress(user_id, logged_at desc);

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
    and wm_trainer.role in ('owner','manager','trainer')
    and wm_trainer.status = 'active'
    and wm_member.role = 'member'
  )
);

-- progress_photos (workspace-scoped)
create table progress_photos (
  id                    uuid primary key default gen_random_uuid(),
  workspace_member_id   uuid not null references workspace_members(id) on delete cascade,
  photo_url             text not null,
  type                  text check (type in ('front','back','side')),
  logged_at             date not null default current_date,
  created_at            timestamptz default now()
);

alter table progress_photos enable row level security;
```

---

## Migration 0006 — Notifications + ZKTeco

```sql
-- notifications
create table notifications (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null references users(id) on delete cascade,
  workspace_id    uuid references workspaces(id) on delete cascade,
  type            text not null,
  title           text not null,
  body            text,
  channel         text default 'in_app' check (channel in ('in_app','whatsapp','sms','email')),
  read            boolean default false,
  external_sent   boolean default false,
  external_sent_at timestamptz,
  external_error  text,
  sent_at         timestamptz,
  created_at      timestamptz default now()
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
create table zkteco_device_commands (
  id                uuid primary key default gen_random_uuid(),
  device_serial     text not null,
  branch_id         uuid references branches(id),
  command_id        int not null,
  command_type      text check (command_type in ('enroll_user','delete_user','enroll_finger','sync_time')),
  zkteco_user_id    text,
  member_name       text,
  status            text default 'pending' check (status in ('pending','sent','confirmed','failed')),
  created_at        timestamptz default now(),
  sent_at           timestamptz,
  confirmed_at      timestamptz
);

alter table zkteco_device_commands enable row level security;

-- zkteco_raw_logs (safety net — never lose attendance data)
create table zkteco_raw_logs (
  id              uuid primary key default gen_random_uuid(),
  branch_id       uuid references branches(id),
  device_serial   text,
  payload         text,
  processed       boolean default false,
  processed_at    timestamptz,
  error           text,
  received_at     timestamptz default now()
);

alter table zkteco_raw_logs enable row level security;

-- workspace_payment_configs (for bringing own Razorpay/Stripe)
create table workspace_payment_configs (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid not null references workspaces(id) on delete cascade,
  provider        text not null,
  is_active       boolean default true,
  country         text,
  config          jsonb,  -- encrypted keys stored here
  created_at      timestamptz default now()
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
create table payment_webhooks_raw (
  id              uuid primary key default gen_random_uuid(),
  workspace_id    uuid references workspaces(id),
  provider        text,
  event_type      text,
  payload         jsonb,
  received_at     timestamptz default now(),
  processed       boolean default false,
  processed_at    timestamptz,
  error           text
);

alter table payment_webhooks_raw enable row level security;

-- payment_events (normalised, provider-agnostic)
create table payment_events (
  id                    uuid primary key default gen_random_uuid(),
  workspace_id          uuid references workspaces(id),
  webhook_raw_id        uuid references payment_webhooks_raw(id),
  event_type            text check (event_type in (
                          'payment.success','payment.failed','payment.refunded',
                          'subscription.created','subscription.cancelled','subscription.renewed'
                        )),
  provider              text,
  provider_payment_id   text,
  provider_order_id     text,
  amount                int,
  currency              text,
  entity_type           text check (entity_type in ('workspace_subscription','member_subscription')),
  entity_id             uuid,
  status                text check (status in ('success','failed','refunded','pending')),
  created_at            timestamptz default now()
);

alter table payment_events enable row level security;
```

---

## Migration 0007 — Seed Data

```sql
-- Your SaaS subscription plans
-- Run this once after migrations

insert into subscription_plans (name, type, max_members, max_branches, price_monthly) values
  ('Trial',   'gym',        20,  1, 0),
  ('Starter', 'gym',        100, 1, 99900),   -- ₹999
  ('Growth',  'gym',        300, 3, 249900),  -- ₹2,499
  ('Pro',     'gym',        -1,  -1, 499900), -- ₹4,999

  ('Trial',   'freelancer', 5,   1, 0),
  ('Solo',    'freelancer', 20,  1, 29900),   -- ₹299
  ('Pro',     'freelancer', -1,  1, 59900);   -- ₹599
```

---

## Key Relationships

```
users
└── workspace_members → workspaces → branches
    ├── attendance
    ├── member_subscriptions → membership_plans
    │   └── member_payments
    ├── progress_photos
    └── workout_logs → workout_set_logs

users (personal)
└── member_progress

workspaces
├── workspace_subscriptions → subscription_plans
│   └── workspace_payments
├── exercise_library
├── workout_plans → workout_weeks → workout_sessions → workout_exercises
│   ├── workout_session_overrides → workout_exercise_overrides
│   └── workout_session_skips
└── diet_plans → diet_meals → diet_items

branches
├── zkteco_device_commands
└── zkteco_raw_logs

notifications
payment_webhooks_raw → payment_events
workspace_payment_configs
```
