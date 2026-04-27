# Master Portal — Design Lock & Architecture

Status: **architecture spine — read this before any module spec**
Owner: Hassan Ateeq · Last updated: 2026-04-27
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4327`

---

## 1. Purpose

This document is the architecture spine that ties every guest-facing module together. It answers three questions:

1. How does a guest enter the system, and what determines what they can do?
2. How do the sub-flows (breakfast, cleaning, mini-shop, etc.) plug into shared infrastructure?
3. What are the invariants that every module must respect to avoid breaking the others?

Each module's own `DESIGN.md` (housekeeping, workflows-admin, etc.) defers to this doc for the cross-cutting concerns.

---

## 2. The big picture

```
Guest on phone
   ↓
WhatsApp deep link with ?c=CODE
   ↓
Master Portal (hamsun.pk/?c=…)
   ↓
   ├─ Breakfast pre-order → service_menus.*
   ├─ Make a request → portal.requests (8 categories)
   ├─ Cleaning preferences → bookings columns
   └─ My stay / charges → bookings + charges
   ↓
Edge functions (portal-submit-request · submit-menu-order · post-charge)
   ↓
Database (portal.requests · service_menus.orders · bookings · charges · audit_log)
   ↓
Notifications (WhatsApp ack to guest · Slack to staff team)
   ↓
PMS Reception Inbox — staff acts on requests, workflow engine renders dynamic buttons
```

---

## 3. Entry · portal code → stay state

### 3.1 URL pattern

`https://hamsun.pk/?c=A4B7K2` — embedded in WhatsApp welcome messages.

### 3.2 Code resolution

Frontend calls RPC `portal_resolve_code(p_code text)` which returns:

- **booking** — id, beds24_booking_id, dates, status, checkin_status, room, property
- **guest** — id, name, phone, language preference
- **property** — id, code (CLF / FSL / EXT), name, time zone, default settings
- **state** — derived stay state (see §4)
- **preferences** — auto_daily_cleaning, cleaning_pref_time, dietary notes, etc.

### 3.3 Code lifecycle

- Generated when booking is CONFIRMED
- Embedded in welcome WhatsApp at confirmation + check-in
- Valid through entire booking + 7 days post-checkout
- One code per booking (never reused)

### 3.4 Browser context

`PortalContext` holds booking ID, guest ID, property ID, stay state. Every sub-flow reads from context — no re-resolution per page.

---

## 4. Stay state machine

```
pre_checkin → arrival_day → in_house → departure_day → post_checkout
```

Transitions:

- **pre_checkin** — booking creation through 1 day before check-in
- **arrival_day** — today is check-in date AND `checkin_status = PENDING`
- **in_house** — `checkin_status = CHECKED_IN`
- **departure_day** — today is check-out date AND not yet checked out
- **post_checkout** — `checkin_status = CHECKED_OUT` (7-day window)

**Important invariant:** Stay state is *derived*, never stored. Computing it from `bookings.check_in` + `checkin_status` + `check_out` ensures it stays correct when booking dates change.

### Features enabled per state

| Sub-flow | pre_checkin | arrival_day | in_house | departure_day | post_checkout |
|---|---|---|---|---|---|
| Welcome / stay info | ✓ | ✓ | ✓ | ✓ | ✓ |
| Breakfast pre-order | — | ✓ | ✓ | — | — |
| Request: F&B (kitchen ad-hoc) | — | — | ✓ | ✓ | — |
| Request: Mini-shop | — | ✓ | ✓ | ✓ | — |
| Request: Housekeeping items | — | — | ✓ | ✓ | — |
| Cleaning preferences | — | ✓ | ✓ | — | — |
| Request: Cleaning (one-off) | — | — | ✓ | — | — |
| Request: Maintenance | — | — | ✓ | ✓ | — |
| Request: Transport | ✓ | ✓ | ✓ | ✓ | — |
| Request: Concierge | ✓ | ✓ | ✓ | ✓ | — |
| Request: Booking action | — | ✓ | ✓ | ✓ | — |
| View charges / folio | ✓ | ✓ | ✓ | ✓ | ✓ |
| Leave feedback | — | — | — | ✓ | ✓ |

---

## 5. The 8 sub-flows

Each lives at its own URL, writes to specific backend tables, follows a typed state machine. All authenticate via the same `PortalContext`.

### 5.1 Inventory

| # | Sub-flow | URL | Status | Backend |
|---|---|---|---|---|
| 1 | Breakfast pre-order | `/portal/breakfast` | Spec'd, awaits backend | `service_menus.*` |
| 2 | Request · F&B (ad-hoc) | `/portal/request/kitchen_fnb` | Live | `portal.requests` (type=`kitchen_fnb`) |
| 3 | Request · Mini-shop | `/portal/request/mini_shop` | Live | `portal.requests` (type=`mini_shop`) |
| 4 | Request · Housekeeping items | `/portal/request/housekeeping` | Live | `portal.requests` (type=`housekeeping`) |
| 5 | Request · Cleaning + preferences | `/portal/request/cleaning` and `/portal/cleaning` | Spec'd | `portal.requests` (type=`cleaning`, new) + `bookings` columns |
| 6 | Request · Maintenance | `/portal/request/maintenance` | Live | `portal.requests` (type=`maintenance`) |
| 7 | Request · Transport | `/portal/request/airport_transport` | Live | `portal.requests` (type=`airport_transport`) |
| 8 | Request · Concierge / general | `/portal/request/concierge` | Live | `portal.requests` (type=`concierge`) |
| (9) | Request · Booking action | `/portal/request/booking_action` | Partial | `portal.requests` (type=`booking_action`) — needs amount-capture work |

### 5.2 Common contract for every "Request" sub-flow

1. Guest selects category (or arrives via direct link with `?type=…`)
2. Item picker (if catalog-driven) or freeform notes
3. Submit
4. Edge function `portal-submit-request` validates + inserts row into `portal.requests`
5. Edge function fires `request_received` WhatsApp ack to guest + Slack to relevant team channel
6. Reception inbox (PMS) shows the new row immediately (15-second polling)
7. Staff progresses status via `WorkflowActionBar` buttons
8. On terminal success status, `request_done` WhatsApp auto-fires
9. If action has `posts_charge: true`, charge goes to folio via `post-charge` edge fn

---

## 6. Data model · ownership

| Table | Owns | Read by | Written by |
|---|---|---|---|
| `public.bookings` | Stay record + guest preferences | All sub-flows · PMS · Beds24 sync | PMS · Beds24 webhook · WhatsApp parser (preferences only) |
| `portal.request_types` | Workflow JSONB per category | Portal · PMS · workflow engine | Workflows admin (or SQL migration) |
| `portal.requests` | Every guest request across all categories | Portal · PMS reception inbox | `portal-submit-request` · `portal-update-request-status` · DB triggers · cron |
| `portal.catalog_items` | Item library | Portal request flows | Catalog admin in PMS |
| `service_menus.menus` | Menu config per service | Breakfast portal | Menu admin |
| `service_menus.orders` | Submitted menu orders | Kitchen Slack · PMS | `submit-menu-order` |
| `charges` | Folio line items | PMS · invoicing | `post-charge` only |
| `portal.request_status_history` | Audit trail of status changes | PMS detail drawer | `portal-update-request-status` (auto) |
| `notification_log` | WhatsApp/email/Slack record | PMS · debugging | All notification edge fns |
| `audit_log` | Trigger-based change log | Integrity Center | DB triggers |
| `housekeeping.daily_cron_log` | Per-booking-per-day cleaning cron decision | Housekeeping team | `daily-cleaning-cron` (planned) |

---

## 7. Edge function topology

### Inbound (portal → backend)

- `portal-submit-request` — all "Request" sub-flows
- `submit-menu-order` — breakfast portal
- `portal_resolve_code` (RPC) — master portal landing

### Outbound (backend → 3rd party)

- `send-whatsapp` — guest notifications (current WhatsApp provider)
- `send-slack` — staff notifications
- `send-email` — booking confirmations, invoices, receipts

### Triggers (DB event → side effect)

- Post-checkout cleaning task auto-creation *(planned, on `bookings.checkin_status → CHECKED_OUT`)*
- Audit log trigger (existing, on ~17 tables)
- Request status history (existing, on every `portal.requests.status` update)

### Cron (scheduled)

- `daily-cleaning-cron` — 09:00 PKT daily *(planned)*
- DND reset — 23:59 PKT daily *(planned)*
- `lock-menu-orders` — every 5 min
- `send-menu-form` / `send-menu-reminder` / `daily-menu-summary` — per menu config

---

## 8. Notification topology

| Event | WhatsApp | Slack | Email |
|---|---|---|---|
| Booking confirmed | Welcome + portal link | — | Confirmation receipt |
| Check-in | Welcome at property + auto-cleaning opt-in | Reception ack | — |
| Request received | `request_received` ack to guest | Per-team channel | — |
| Request status change | If `notify_guest_on_action=true` | Optional escalation | — |
| Request done (terminal) | `request_done` auto-fires | — | — |
| Charge posted to folio | Receipt with running balance | — | — |
| Manager approval needed | — | Approval channel with action buttons | — |
| Cleaning opt-in toggle | Confirmation reply | — | — |
| Checkout | Folio summary + feedback link (post 24h) | — | Final invoice if requested |

### WhatsApp templates needed

Every notification message uses a pre-approved WhatsApp template. Templates needed:

- `welcome_with_portal`
- `checkin_welcome_cleaning_opt_in`
- `request_received`
- `request_done`
- `request_cancelled`
- `cleaning_done`
- `cleaning_skip_confirmed`
- `cleaning_pref_changed`
- `folio_charge_posted`
- `checkout_summary`
- `feedback_request`

---

## 9. Build sequence · dependency order

| # | Step | Depends on | Effort |
|---|---|---|---|
| ✓ | Master portal landing exists in `hamsun-portal` | — | done |
| ✓ | `portal.requests` + workflow engine | — | done |
| 1 | Workflow vocabulary cleanup (Slice 1) | nothing strictly | ~2 hr |
| 2 | Cleaning sub-flow / housekeeping module | Q1–Q8 answered | ~32 hr |
| 3 | Reception inbox v2 React rebuild | step 2 (peek cards need real data) | ~34 hr |
| 4 | Workflows admin React in PMS | step 1 (vocabulary stable) | ~12 hr |
| 5 | Menu admin backend (independent track) | — | ~24 hr |
| 6 | Breakfast portal wired to menu backend | step 5 | ~8 hr |
| 7 | WhatsApp BSP migration (whapi.cloud + Meta) | nothing technical, but blocks live notifications today | ~16 hr + template approval lead time |
| 8 | Concierge subtype split-out | step 4 | ~8 hr |

---

## 10. Critical invariants · what never to break

These must remain true across all module builds. Violating one causes silent data corruption or production breakage.

### Hard invariants

1. **Every request goes through `portal.requests`.** No new tables for "cleaning tasks" or "transport requests". The category is `type_slug`, the lifecycle is `workflow.status`. Guarantees consistent reporting, audit, and reception inbox surfacing.

2. **Charges always go through `post-charge` edge fn.** No direct INSERT into `charges` from any portal flow. The action with `posts_charge: true` calls `post-charge` which handles tax mode, currency, audit log, Beds24 backup posting.

3. **No secrets in the prototypes repo.** This repo is public. No API keys, booking IDs, guest names, phone numbers, or production data.

### Soft invariants (warn-level)

4. **Portal code resolves to exactly one booking** — `portal_resolve_code` must never return multiple or null when valid. Add tests; don't change casually.

5. **Workflow status keys are immutable once seeded** — labels editable, keys not. Renaming a key requires migrating `portal.requests.status`, `request_status_history`, and template names.

6. **Stay state derives from booking, never stored on `portal.requests`** — state is computed from booking dates + checkin_status. Denormalising would lie when booking changes.

7. **Notifications must be idempotent.** Edge functions dedupe internally. Calling twice must not send twice. Notification logic in the edge fn, not the React caller.

8. **Cleaning module reuses `portal.requests`** — no parallel `housekeeping.cleaning_tasks` table. Use `portal.requests` with `type_slug='cleaning'`. The `daily_cron_log` is a side log, not a parallel store.

9. **Every status change logs to history.** Always go through `portal_update_request_status` RPC. Direct UPDATE on `requests.status` breaks audit trail.

10. **Per-property differences live in `app_settings` or property_id columns, not in code.** Hamsun has 3 properties (CLF, FSL, EXT) and may add a 4th. No hardcoded "9am cron for FSL only" in edge functions.

---

## 11. Files

```
master-flow/
├── index.html              ← visual architecture doc (10 sections, sticky TOC)
├── DESIGN.md               ← this file (locked spec)
└── .claude/launch.json     ← preview server on port 4327
```

---

*Locked. This is the spine. All module specs must be consistent with it.*
