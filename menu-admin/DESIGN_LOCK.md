# Menu Admin — Design Lock & Backend Spec

Locked: 2026-04-26 · Hassan + Claude

This document closes the design loop on the multi-menu admin and hands off to backend implementation. The prototype lives at `http://localhost:4323/` and matches every behaviour described below.

---

## 1. What's Locked

### 1.1 Multi-menu (services)

- The admin manages **N service menus**, not just breakfast.
- Pre-seeded examples: **Breakfast · Lunch · Dinner · Late-night**.
- New ones can be added at any time via "+ Add menu" — example use cases the user named: **Snacks · All-day Tea · High Tea**.
- Each menu carries: name, icon, enabled flag, pricing mode, time slots, mains, sides, starting drinks, ending drinks.
- Switching between menus replaces the entire admin context.

### 1.2 Pricing models

Three modes per menu:

| Mode | Description | Typical use |
|---|---|---|
| `allowance` | N comp covers per booking + flat-rate extra cover charge | Breakfast |
| `a_la_carte` | Every item priced individually; folio gets the sum | Lunch · Dinner · Snacks · All-day Tea |
| `hybrid` | Comp covers basics, some items always carry a surcharge | Mixed concepts |

Switching modes hides irrelevant settings (allowance fields disappear when à la carte).

### 1.3 Per-item & per-side pricing

- Every main has a `price` field (PKR). `0` = within allowance / free.
- Every side has a `price` field. `0` = free; `>0` = surcharge added to folio when selected.
- Demo seeded with two real examples: `Avocado Toast (Premium) — PKR 850`, `Avocado side — +PKR 200`, `Smoked Salmon — +PKR 500`.

### 1.4 Choice groups (unified modifier model)

A main course attaches one or more **choice groups**. Each group:
- has a `label` and `description`
- pulls options from one of two sources:
  - `custom` — inline list (e.g., "Cooking oil: Desi ghee / Butter / Olive oil")
  - `master_sides` — references the master sides list, with per-dish allowed subset
- has a `pickMode` preset: exactly N · up to N · range · all included
- has `required` / optional flag

Replaces the old separate "modifiers" + "side rules" concepts.

### 1.5 Section rules per cover

Per-section pick min/max (mains, sides, starting, ending). Shown on the Section Rules panel; per-main side rules in the dish modal override these.

### 1.6 Time slots

Per menu — chip-based add/remove with left-right reorder. Lock window (minutes before slot) is configurable per menu.

### 1.7 Reorder controls everywhere

- Mains list: ▲▼
- Sides list: ▲▼
- Choice groups within a dish: ▲▼ + "Group N of M" badge
- Time slots: ◀▶
- Drink chips: ◀▶ + ✕ remove

### 1.8 Seasonal / dynamic items

- The Starting Drinks panel exposes a "Today's juice flavor" input. When updated:
  - Syncs to "Fresh Juice (X)" if such a chip exists
  - Updates Fruit Bowl description ("Today: Mango, Pineapple…")
- Designed to be edited daily by housekeeping or kitchen lead.

### 1.9 Kitchen + Slack notifications

Documented (not interactive) panel describing:

- Slack KOT fires per slot to per-property channel
- 06:30 AM daily summary aggregated by slot
- Status updates (Preparing / Ready / Delivered) with WhatsApp pings to the room group, each toggleable
- Late-alert escalation to reception (off by default)

### 1.10 Plating & decoration

Per-dish prep notes + plating notes (free text, surface on KOT). Plus global defaults for in-room tray setup and café table setup.

### 1.11 Allowance + folio semantics

Submission produces a charge breakdown per the active pricing mode. Example for allowance mode:

```
Breakfast — Room 301 — Sat 27 Apr
─────────────────────────────────
Aloo Paratha            (within allowance) free
Avocado Toast (Premium)              +850
Sides: Avocado                       +200
Extra cover (1 × 2,000)            +2,000
─────────────────────────────────
TOTAL                          PKR 3,050
```

All charges flow through the existing `post-charge` edge function with `category='room_service'`.

---

## 2. What's Parked (deliberately not in v1)

| Parked | Why |
|---|---|
| Per-option pricing for **custom** choice-group options (e.g. "Sourdough +100" inline within a Bread group) | Master-side surcharges cover 90% of the use case. Custom-option pricing is a 30-min add later. |
| Drag-and-drop reorder | Up/down + left/right buttons are reliable, touch-friendly, no library overhead. |
| Per-day or per-time-slot item availability (e.g. Light Bowl only after 8:30) | Adds schedule complexity. The on/off toggle handles "out of stock today" cases. |
| Delete-menu UX | User can prune via SQL during ops; not exposed in admin yet. |
| Kitchen Display System (KDS) on a wall tablet | Slack KOT covers MVP. KDS is a future module. |
| Real photography upload pipeline | URL paste works for now; uploader is a separate Storage-bucket task. |
| Per-property menu overrides (FSL might run something CLF doesn't) | Property selector in the topbar implies it; backend supports it (every menu row has `property_id`); admin UI for per-property differences not yet built. |

---

## 3. Backend Schema Spec

### 3.1 Schema rename

Rename the existing Postgres schema from `breakfast` to `service_menus` (or just leave it; either works).

### 3.2 Top-level tables

```sql
CREATE TABLE service_menus.menus (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id     uuid NOT NULL REFERENCES public.properties(id),
  code            text NOT NULL,                                  -- 'breakfast','lunch','dinner','snacks',...
  name            text NOT NULL,
  icon            text NOT NULL,                                  -- emoji or icon key
  is_enabled      boolean NOT NULL DEFAULT false,
  pricing_mode    text NOT NULL CHECK (pricing_mode IN ('allowance','a_la_carte','hybrid')),
  complimentary_count   int NOT NULL DEFAULT 0,
  extra_cover_price     numeric NOT NULL DEFAULT 0,
  lock_minutes_before   int NOT NULL DEFAULT 40,
  form_send_time        time,
  reminder_time         time,
  daily_summary_time    time,
  slack_channel         text,
  in_room_tray_setup    text,                                     -- default plating note
  cafe_table_setup      text,
  status_notify_preparing  boolean NOT NULL DEFAULT true,
  status_notify_ready      boolean NOT NULL DEFAULT true,
  status_notify_delivered  boolean NOT NULL DEFAULT true,
  status_notify_late       boolean NOT NULL DEFAULT false,
  sort_order      int NOT NULL DEFAULT 0,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  UNIQUE(property_id, code)
);

CREATE TABLE service_menus.time_slots (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_id         uuid NOT NULL REFERENCES service_menus.menus(id) ON DELETE CASCADE,
  label           text NOT NULL,
  start_time      time NOT NULL,
  end_time        time NOT NULL,
  sort_order      int NOT NULL DEFAULT 0,
  is_active       boolean NOT NULL DEFAULT true
);

-- Master-list sides per menu
CREATE TABLE service_menus.sides (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_id         uuid NOT NULL REFERENCES service_menus.menus(id) ON DELETE CASCADE,
  code            text NOT NULL,
  name            text NOT NULL,
  description     text,
  price           numeric NOT NULL DEFAULT 0,
  is_active       boolean NOT NULL DEFAULT true,
  is_seasonal     boolean NOT NULL DEFAULT false,
  sort_order      int NOT NULL DEFAULT 0,
  UNIQUE(menu_id, code)
);

-- Drinks lists per menu (start + end)
CREATE TABLE service_menus.drinks (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_id         uuid NOT NULL REFERENCES service_menus.menus(id) ON DELETE CASCADE,
  bucket          text NOT NULL CHECK (bucket IN ('starting','ending')),
  name            text NOT NULL,
  price           numeric NOT NULL DEFAULT 0,
  sort_order      int NOT NULL DEFAULT 0,
  is_active       boolean NOT NULL DEFAULT true
);

-- Mains (and any "primary" items)
CREATE TABLE service_menus.menu_items (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_id         uuid NOT NULL REFERENCES service_menus.menus(id) ON DELETE CASCADE,
  name            text NOT NULL,
  short_desc      text,
  full_desc       text,
  image_url       text,
  price           numeric NOT NULL DEFAULT 0,
  is_popular      boolean NOT NULL DEFAULT false,
  is_active       boolean NOT NULL DEFAULT true,
  prep_notes      text,
  plating_notes   text,
  sort_order      int NOT NULL DEFAULT 0,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now()
);

-- Choice groups (unified modifier + side picker)
CREATE TABLE service_menus.choice_groups (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_item_id    uuid NOT NULL REFERENCES service_menus.menu_items(id) ON DELETE CASCADE,
  label           text NOT NULL,
  description     text,
  source          text NOT NULL CHECK (source IN ('custom','master_sides','master_starting','master_ending')),
  pick_mode       text NOT NULL CHECK (pick_mode IN ('exact','up_to','range','all_included')),
  pick_min        int NOT NULL DEFAULT 0,
  pick_max        int NOT NULL DEFAULT 1,
  is_required     boolean NOT NULL DEFAULT false,
  sort_order      int NOT NULL DEFAULT 0
);

-- Custom options inline within a group
CREATE TABLE service_menus.choice_group_custom_options (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id        uuid NOT NULL REFERENCES service_menus.choice_groups(id) ON DELETE CASCADE,
  value           text NOT NULL,
  emoji           text,
  price           numeric NOT NULL DEFAULT 0,
  sort_order      int NOT NULL DEFAULT 0
);

-- References to master items (sides, drinks) selected for this group
CREATE TABLE service_menus.choice_group_master_refs (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id        uuid NOT NULL REFERENCES service_menus.choice_groups(id) ON DELETE CASCADE,
  master_side_id  uuid REFERENCES service_menus.sides(id),
  master_drink_id uuid REFERENCES service_menus.drinks(id),
  sort_order      int NOT NULL DEFAULT 0,
  CHECK ((master_side_id IS NOT NULL)::int + (master_drink_id IS NOT NULL)::int = 1)
);
```

### 3.3 Order tables (mostly carry over from existing breakfast spec)

```sql
service_menus.sessions    -- one per booking per menu per date
service_menus.orders      -- one per session
service_menus.order_items -- per-person × per-section selections, plus pricing snapshot
```

`order_items.unit_price` snapshots the price at order time so future menu price changes don't retroactively reprice old orders.

### 3.4 Indexes

```sql
CREATE INDEX ON service_menus.menus (property_id, is_enabled);
CREATE INDEX ON service_menus.time_slots (menu_id, sort_order);
CREATE INDEX ON service_menus.menu_items (menu_id, is_active, sort_order);
CREATE INDEX ON service_menus.choice_groups (menu_item_id, sort_order);
CREATE INDEX ON service_menus.sessions (booking_id, session_date);
CREATE INDEX ON service_menus.orders (booking_id, order_date);
```

---

## 4. Edge Function Surface (per service menu)

Each menu needs the same set of edge functions, parameterised by `menu_id`:

- `get-menu-context` (loads sections + items + choice groups + items pricing for guest portal)
- `submit-menu-order` (validates choices, computes folio total, posts charges)
- `lock-menu-orders` (cron, every 5 min, locks sessions whose slots are within `lock_minutes_before`)
- `send-menu-form` (cron, fires the WhatsApp link for menus that schedule pre-orders)
- `send-menu-reminder` (cron, fires the morning reminder)
- `daily-menu-summary` (cron, posts the prep summary to Slack)
- `update-menu-status` (interactive — Preparing / Ready / Delivered button webhook)

For à la carte menus (lunch/dinner) without pre-order forms, `send-menu-form` and `send-menu-reminder` aren't scheduled. Instead the guest opens the menu on demand from the portal during service hours.

---

## 5. Build Order

| Phase | Work | Effort |
|---|---|---|
| 1 | Apply schema migration (rename + add tables) | 30 min |
| 2 | Seed Breakfast menu from current dev branch | 1 hr |
| 3 | Build `get-menu-context` + `submit-menu-order` (parameterised by menu_id) | 4 hr |
| 4 | Wire admin UI to Supabase (replace JS in-memory model with API calls) | 6 hr |
| 5 | Wire guest portal to read from `service_menus.*` | 3 hr |
| 6 | Plumb à la carte folio math through `post-charge` | 1 hr |
| 7 | Cron jobs for lock + summary + form | 2 hr |
| 8 | Slack KOT formatter handles per-menu config | 2 hr |
| 9 | QA against breakfast end-to-end on dev branch | 4 hr |
| **Total** | | **~24 hours** |

Hand-off ready. Spec stable.

---

## 6. Files

```
/Users/hassanateeq/menu-admin/
  ├── index.html              ← prototype (mirror of all decisions above)
  ├── DESIGN_LOCK.md          ← this file
  └── .claude/launch.json     ← preview server on port 4323

/Users/hassanateeq/breakfast-prototype/
  ├── index.html              ← guest-facing breakfast portal
  └── .claude/launch.json     ← preview server on port 4322
```

Both servers run side-by-side. The "Preview as guest ↗" button in the admin opens the guest portal in a new tab.

---

*Locked. Proceed to backend.*
