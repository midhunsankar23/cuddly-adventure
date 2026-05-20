# FitDesk — Development Log

Decisions made during development. Why things work the way they do.
Add an entry every time you make an architectural decision.

---

## May 2026 — Initial architecture

**Decision:** Next.js 15 + Cloudflare Workers & Pages + Supabase
**Why:** Switched from Nuxt due to oxc-parser native binding bug on Cloudflare's build environment. Next.js 15 has mature Cloudflare support, larger ecosystem, better AI tooling support. Supabase provides Auth + Postgres + Realtime in one. Cloudflare auto-deploys from GitHub.

**Decision:** One account, roles per workspace
**Why:** Same user can be gym owner, trainer at another gym, and member at a third — all from one login. Like Instagram account switching. Roles stored in workspace_members table, not on the user.

**Decision:** Sequential workout plans as the default
**Why:** Most Indian gym trainers use PPL / bro splits where the rotation doesn't follow the calendar. Sequential mode means skipping a day doesn't break the sequence — it picks up where it left off.

**Decision:** GPS geofencing for attendance in v1, ZKTeco in v2
**Why:** Most small gyms (80% of market) have no biometric hardware. GPS works on any phone with no hardware cost. ZKTeco integration is a premium feature for gyms that already have devices.

**Decision:** Files uploaded directly to R2 via presigned URLs
**Why:** Files never pass through your server. Cheaper, faster, and more secure. Server only generates the presigned URL — actual upload goes client → R2 directly.

**Decision:** Money stored in paise (smallest unit)
**Why:** Avoids floating point precision issues. ₹999 stored as 99900. Standard practice for payment systems.

**Decision:** member_progress is user-scoped, not workspace-scoped
**Why:** A person's weight and measurements are personal data belonging to them, not to any specific gym. If they train at two gyms, both trainers see the same progress data. Progress photos are workspace-scoped because those are more sensitive and gym-specific.

**Decision:** Payment system is provider-agnostic from day one
**Why:** Different gyms may want different providers (Razorpay for India, Stripe for UAE). Schema uses payment_provider text field + provider_response jsonb so any provider works without schema changes. v1 uses manual payments, v2 adds Razorpay, v3 adds Stripe — zero migrations needed.

**Decision:** Raw webhook logs saved before processing
**Why:** If the normaliser has a bug, the raw data is preserved. Can reprocess any failed webhook by replaying from payment_webhooks_raw. Never lose a payment event.

**Decision:** ZKTeco member enrollment done from FitDesk, not ZKTeco interface
**Why:** Gym staff should never need to touch the ZKTeco device interface. FitDesk sends ADMS commands via the device heartbeat response. Receptionist adds member in FitDesk → device gets the user registered → member scans finger → done. Zero ZKTeco UI interaction needed.

---

*Add new entries below as development progresses*
