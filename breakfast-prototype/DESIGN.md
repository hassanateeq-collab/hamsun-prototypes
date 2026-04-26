# Breakfast Prototype — Design Lock

Status: **locked, mirrors the menu-admin schema and is the guest counterpart**
Owner: Hassan Ateeq
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4322`

---

## 1. What this module is

The **guest-facing** breakfast pre-order portal. Guests open this via a WhatsApp link the night before service. They pick a time slot, pick mains with choice groups (modifiers + sides), pick starting/ending drinks, and submit — the order posts to the kitchen and (for paid items) the guest folio.

Companion to [`menu-admin/`](../menu-admin) which configures *what* is on the menu. This prototype shows *how the guest sees it*.

---

## 2. What's locked

### 2.1 Flow

1. Guest taps WhatsApp link → opens portal with booking context (room, guests, dates pre-filled)
2. Time slot picker (7:30 / 8:00 / 8:30 / 9:00 — per per `menu-admin` time_slots config)
3. For each cover (allowance count from booking + extras), pick:
   - 1 main from menu_items
   - Choice groups attached to that main (sides / modifiers, with custom or master_sides source)
4. Starting drinks (chip select)
5. Ending drinks (chip select)
6. Review screen: line items + price breakdown + folio total
7. Submit → backend records order, kitchen gets KOT via Slack

### 2.2 Pricing semantics (from menu-admin DESIGN_LOCK)

Three pricing modes per menu, displayed differently:

- **Allowance** — N comp covers, extras charged at flat rate (currently breakfast)
- **À la carte** — every item priced; folio = sum of items
- **Hybrid** — comp covers basics; some items always carry surcharge

Charge breakdown shown to guest before submit, e.g.:

```
Aloo Paratha            (within allowance) free
Avocado Toast (Premium)              +850
Sides: Avocado                       +200
Extra cover (1 × 2,000)            +2,000
─────────────────────────────────────────
TOTAL                          PKR 3,050
```

### 2.3 Lock window

Per menu config, orders lock `lock_minutes_before` minutes before slot start. Past the lock, the menu is read-only ("Order window closed; please contact reception").

### 2.4 Visual design

Same palette as menu-admin (cream / gold / Fraunces serif + Inter sans).

---

## 3. What's parked

| Parked | Why |
|---|---|
| Per-cover individual ordering UI | v1 just collects total counts; per-person assignment is v2 |
| Photos on every menu item | URL paste works; uploader pipeline is later |
| Multi-language | English + Urdu in v2 |
| Allergy / dietary filters | Free-text notes for now |
| Save-for-later cart across sessions | Single-session order; no draft persistence |

---

## 4. Backend dependency

This portal reads from the schema spec'd in [`../menu-admin/DESIGN_LOCK.md`](../menu-admin/DESIGN_LOCK.md) §3. Specifically uses:

- `service_menus.menus` — menu config + lock window
- `service_menus.time_slots` — slot picker
- `service_menus.menu_items` — mains
- `service_menus.choice_groups` — modifier/side groups per main
- `service_menus.choice_group_master_refs` / `choice_group_custom_options`
- `service_menus.sides` — master sides list
- `service_menus.drinks` — starting + ending drinks
- `service_menus.sessions` / `orders` / `order_items` — submission target

Edge functions consumed:
- `get-menu-context` — load menu for guest's stay
- `submit-menu-order` — validate + post charges
- `lock-menu-orders` — server-side lock enforcement

---

## 5. Build status

The backend schema is locked (per menu-admin DESIGN_LOCK §5 build order). This prototype is the design reference for what the guest UX looks like once the backend is wired. **No new build phases unique to this prototype** — the work is part of the menu-admin project (Phase 5: "Wire guest portal to read from `service_menus.*`" — 3 hr estimate).

---

## 6. Files

```
breakfast-prototype/
├── index.html              ← guest portal mockup
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4322
```

---

*Locked. Backend wiring tracked in menu-admin/DESIGN_LOCK.md.*
