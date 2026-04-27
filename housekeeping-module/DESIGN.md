# Housekeeping Module — Design Lock

Status: **spec'd, 8 open decisions pending operator sign-off**
Owner: Hassan Ateeq · Last updated: 2026-04-27
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4326`

---

## 1. What this module is

A separate operational module for **cleaning and deep-cleaning tasks**. The cleaning team owns this queue end-to-end. Reception sees a peek (in the command center) so they know what's pending and overdue across teams.

**In scope** — daily cleaning, deep cleaning, bathroom-only, turn-down, post-checkout cleaning, linen change as part of a cleaning visit.

**Out of scope** — items requests (towels, linen, toiletries, slippers, kettle, hairdryer, amenities top-up, laundry pickup, iron). Those continue through the existing reception request portal.

---

## 2. What's locked

### 2.1 Four task sources

| # | Source | Trigger | Generates | Default |
|---|---|---|---|---|
| 1 | **On checkout** (auto) | `bookings.checkin_status` → `CHECKED_OUT` | `post_checkout` cleaning task | Always on, per-property override possible |
| 2 | **Daily cron** (auto) | Daily at fixed time PKT | `daily_clean` task per qualifying booking | Skip rules apply (see §3) |
| 3 | **Guest portal** (user) | Guest taps "Request cleaning" in stay portal | Manual cleaning task with chosen subtype | — |
| 4 | **Reception / PMS** (user) | Staff manual entry (walk-in request, mid-stay deep clean booked at check-in) | Manual cleaning task | Permission `cleaning.create` |

### 2.2 Single source of truth for auto-daily

Booking-level boolean column: `bookings.auto_daily_cleaning`. Three entry points all write to the same column:

- **Guest portal** — toggle in `/portal/preferences`
- **WhatsApp reply** — guest replies `DAILY` / `STOP` (parsed by the WhatsApp inbound webhook)
- **Reception PMS** — checkbox on booking edit form

Default for new bookings: **off** (opt-in via WhatsApp at check-in). See Q2.

### 2.3 Skip rules

When `auto_daily_cleaning = true`, the daily cron still skips:

1. **Arrival day** — room is fresh from previous post-checkout
2. **Departure day** — post-checkout source will fire anyway
3. **DND today** — `bookings.cleaning_dnd_today = true` (one-day flag, reset at midnight)
4. **Already cleaned today** — task already exists in active or done state for this room today
5. **Date in skip list** — `today ∈ bookings.cleaning_skip_dates[]` (planned future skips)

### 2.4 Task subtypes

`daily_clean` · `deep_clean` · `bathroom_only` · `post_checkout` · `turn_down` · `linen_change` · `refresh`

### 2.5 Workflow states

`scheduled` (initial) → `assigned` → `in_progress` → `done` (terminal)
Off-path: `skipped` (terminal) · `cancelled` (terminal)

Color map: amber → blue → indigo → green | orange (skipped) | gray (cancelled).

### 2.6 Workflow actions

| Button | From | To | Flags |
|---|---|---|---|
| → Assign | `scheduled` | `assigned` | requires_assignee |
| ✓ Started | `assigned` | `in_progress` | — |
| ✓ Done | `in_progress` | `done` | notify_guest_on_action (template `cleaning_done`) |
| ⊘ Skip today | any active | `skipped` | requires_note |
| ✕ Cancel | any active | `cancelled` | destructive · requires_note |

### 2.7 WhatsApp reply parsing

| Guest message | Effect | Bot reply |
|---|---|---|
| `DAILY`, "yes daily", "yes please" | Set `auto_daily_cleaning = true` | "Got it — daily cleaning starting tomorrow." |
| `STOP`, "no daily", "cancel daily" | Set `auto_daily_cleaning = false` | "Daily cleaning paused. Reply DAILY anytime to restart." |
| `SKIP`, "skip today", "no clean today" | Set `cleaning_dnd_today = true` | "Skipping today's cleaning. Tomorrow as usual." |
| `CLEAN NOW`, "send cleaning" | Create immediate cleaning task | "Cleaning crew on the way — within 30 min." |
| `CHANGE TIME hh:mm` | Set `cleaning_pref_time = 'hh:mm'` | "Cleaning shifted to {time} starting tomorrow." |

### 2.8 Guest portal toggle

A "Cleaning preferences" section in the stay portal with three rows:

1. **Daily cleaning** — toggle for `auto_daily_cleaning`
2. **Skip today** — toggle for `cleaning_dnd_today`
3. **Preferred time** — picker for `cleaning_pref_time`

Plus a button: **Request cleaning now** (creates immediate task).

### 2.9 Reception command center peek

The reception inbox shows a small "Housekeeping (cleaning)" peek card with: count of new / in progress / done today, top 3 oldest items with overdue highlighting, "Open cleaning →" jump button.

---

## 3. What's parked (deliberately not in v1)

| Parked | Why |
|---|---|
| Per-property different daily-cleaning policies | Single global policy easier; per-property in v2 once we know what differs |
| Multi-day skip via WhatsApp ("skip Wed and Thu") | Single-day SKIP covers 90%; multi-day via portal only |
| Photo proof on cleaning done | Nice-to-have, parks until v2 |
| Cleaning crew assignment dropdown synced from staff table | Free-text staff name in v1; staff-table integration in v2 |
| Auto-deep-clean weekly for stays > 7 nights | See Q4 — not v1 unless operator says yes |
| Damage / extra-clean fee on post-checkout | See Q8 — out of v1 scope |

---

## 4. Backend schema spec

### 4.1 Booking preference columns

```sql
ALTER TABLE public.bookings
  ADD COLUMN auto_daily_cleaning   boolean DEFAULT false,
  ADD COLUMN cleaning_pref_time     time DEFAULT '10:00',
  ADD COLUMN cleaning_dnd_today     boolean DEFAULT false,
  ADD COLUMN cleaning_skip_dates    date[] DEFAULT '{}',
  ADD COLUMN cleaning_special_notes text;
```

### 4.2 New `cleaning` request_type

Reuses existing `portal.requests` infrastructure — no new task table. Insert into `portal.request_types`:

```sql
INSERT INTO portal.request_types (slug, display_name, icon, default_team, is_paid, default_sla_minutes, workflow)
VALUES (
  'cleaning',
  'Housekeeping (cleaning)',
  'broom',
  'housekeeping',
  false,
  60,
  '{ "initial_status": "scheduled", "statuses": [...], "actions": [...], "subtypes": ["daily_clean", "deep_clean", "bathroom_only", "post_checkout", "turn_down", "linen_change", "refresh"] }'::jsonb
);
```

Full workflow JSON: see [`index.html`](./index.html) §4 and the workflows admin format in [`../workflows-admin/DESIGN.md`](../workflows-admin/DESIGN.md).

### 4.3 New source enum values

```sql
ALTER TYPE portal.request_source ADD VALUE 'cleaning_cron';
ALTER TYPE portal.request_source ADD VALUE 'checkout_trigger';
```

### 4.4 Optional bookkeeping log

```sql
CREATE TABLE housekeeping.daily_cron_log (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  ran_at       timestamptz NOT NULL DEFAULT now(),
  booking_id   uuid NOT NULL REFERENCES public.bookings(id),
  ran_date     date NOT NULL,
  outcome      text NOT NULL CHECK (outcome IN (
    'task_created', 'skip_arrival', 'skip_departure',
    'skip_dnd', 'skip_already_cleaned', 'skip_listed_date',
    'skip_pref_off'
  )),
  task_id      uuid REFERENCES portal.requests(id),
  UNIQUE (booking_id, ran_date)
);
```

---

## 5. Triggers & cron

| # | What | When | Where |
|---|---|---|---|
| 1 | Postgres trigger on `bookings.checkin_status` → `CHECKED_OUT` inserts a `cleaning` request with `subtype='post_checkout'`, `source='checkout_trigger'`, `scheduled_for=now()` | On row update | DB trigger |
| 2 | Daily cron — loops opted-in CHECKED_IN bookings, applies skip rules, inserts `daily_clean` tasks, logs every booking checked into `daily_cron_log` | 09:00 PKT daily (Q1) | Edge function `daily-cleaning-cron` |
| 3 | WhatsApp inbound parser — extends the WhatsApp inbound webhook to recognise DAILY / STOP / SKIP / CLEAN NOW / CHANGE TIME and update the booking flags + send confirm reply | On every WhatsApp inbound message | Edge function (existing, extended) |
| 4 | Daily DND reset — `UPDATE bookings SET cleaning_dnd_today = false WHERE cleaning_dnd_today = true` | 23:59 PKT daily | SQL cron / pg_cron |

---

## 6. Build order

| Phase | Work | Effort |
|---|---|---|
| 1 | Apply schema migration (booking columns + cleaning request_type + source enum + daily_cron_log table) | 1 hr |
| 2 | Postgres trigger for post-checkout cleaning task | 1 hr |
| 3 | Edge function `daily-cleaning-cron` with full skip-rule logic | 4 hr |
| 4 | Schedule the cron (Supabase pg_cron or external scheduler) at chosen time | 30 min |
| 5 | Extend the WhatsApp inbound webhook for DAILY/STOP/SKIP/CLEAN NOW/CHANGE TIME parsing | 3 hr |
| 6 | Guest portal — Cleaning preferences section in stay portal | 4 hr |
| 7 | Guest portal — "Request cleaning" item in request flow with subtype picker | 2 hr |
| 8 | PMS — booking edit form: add `auto_daily_cleaning` checkbox + special notes field | 2 hr |
| 9 | PMS — Reception inbox housekeeping peek card (already designed in `reception-inbox/`) | 2 hr |
| 10 | PMS — Housekeeping queue page (full view for housekeeping team) | 6 hr |
| 11 | Daily DND reset job | 30 min |
| 12 | E2E test plan + dev branch verification | 6 hr |
| **Total** | | **~32 hours** |

---

## 7. Open decisions (Q1–Q8)

These need operator sign-off before phase 1 starts:

| # | Question | Default proposed |
|---|---|---|
| Q1 | What time should the daily cron run? | `09:00 PKT` |
| Q2 | Default for new bookings — auto-daily on or off? | **Off**, with WhatsApp opt-in at check-in |
| Q3 | Skip arrival day & departure day automatically? | **Yes** to both |
| Q4 | Auto weekly deep-clean for stays > 7 nights? | **No** for v1, all manual |
| Q5 | Cleaning team — separate or part of housekeeping? | Default `default_team='housekeeping'` |
| Q6 | When does the WhatsApp opt-in fire? | At check-in, as part of welcome |
| Q7 | Naming — keep new type as `cleaning` (separate from existing `housekeeping` request type for items)? | **Yes**, keep distinct in DB; UI labels both as "Housekeeping" |
| Q8 | Can post-checkout ever bill the guest (damage / extra mess fee)? | **No** in v1, always free |

Operator can answer "your defaults, proceed" to use all eight.

---

## 8. Files

```
housekeeping-module/
├── index.html              ← visual flow doc (10 sections, sticky TOC)
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4326
```

---

*Locked once Q1–Q8 are answered. Then proceed to Phase 1.*
