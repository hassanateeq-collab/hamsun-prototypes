# Workflows Admin — Design Lock

Status: **prototype editor working in-browser, awaiting workflow vocabulary sign-off**
Owner: Hassan Ateeq · Last updated: 2026-04-27
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4325`

---

## 1. What this module is

A visual editor for the per-category workflow JSONB stored in `portal.request_types.workflow`. Lets the operator add/remove statuses, define actions, set the `posts_charge` flag, configure flag requirements, and see a live preview of what reception will see — without writing SQL or jsonb by hand.

The data model already exists in production. Seven workflows are seeded today. This module provides the UI to edit them safely.

---

## 2. What's locked

### 2.1 Concepts the editor exposes

For each request type, the editor lets you edit:

- **Category metadata** — icon, display name, default team, paid (with charge category), default SLA in minutes
- **Statuses** — label, color (8 options), `is_active`, `is_terminal`, reorder
- **Actions** — label, from-state (or "any active"), to-state, all flags, reorder
- **Subtypes** — chip-based add/remove
- **Severity levels** — chip-based add/remove (optional, only on maintenance currently)

### 2.2 Action flags (the configurable surface)

Each action has independent boolean / text flags:

| Flag | Effect on reception UI / backend |
|---|---|
| `posts_charge` | When fired, automatically posts the category's charge to the guest folio |
| `destructive` | Renders red, requires 2-tap confirm before firing |
| `requires_note` | Staff must enter explanation before save |
| `requires_assignee` | Staff must pick assignee from staff list |
| `requires_scheduled_for` | Requires future date/time (e.g. wake-up call, pickup) |
| `requires_role` | Only staff with named role (e.g. `manager`) can trigger |
| `notify_guest_on_action` + `notification_template` | Sends WhatsApp to guest using named template |

### 2.3 Live preview

For every non-terminal status, the editor shows: "When status = X, reception sees these buttons" — auto-recomputes as you edit. Buttons that post a charge get a 💰 marker; destructive buttons render in red.

### 2.4 Unsaved-state model

- Edits autosave to `localStorage` (key `hamsun-workflows-editor-v1`)
- Refreshing preserves edits
- Top-right pill shows "N unsaved" (or "All saved")
- Floating bottom bar appears when anything is dirty:
  - **⊘ Discard all** — reset everything to seeded
  - **{ } Copy JSON** — copies modified workflows
  - **↓ Generate SQL migration** — produces a runnable migration

### 2.5 SQL migration generator

The "Generate SQL migration" button outputs a single `BEGIN; UPDATE … ; COMMIT;` block:

- One `UPDATE portal.request_types SET …` per modified workflow
- Updates `display_name`, `icon`, `default_team`, `is_paid`, `charge_category`, `default_sla_minutes`, `workflow`, `updated_at`
- **Auto-derives the `transitions` array** from action `from`→`to` pairs (so user never edits transitions manually — they're a function of actions)

The output is plain SQL ready to paste into Supabase SQL editor or apply via `mcp__supabase__apply_migration`.

### 2.6 Per-card actions

- **↺ Reset to seed** — undo all edits to one category
- **{ } Copy this workflow's JSON** — view + copy the single workflow JSON in a modal

---

## 3. What's parked

| Parked | Why |
|---|---|
| Save directly to DB from the editor | Prototype only — protects prod against editor bugs. SQL migration step is the safety valve. Once the editor is rebuilt as a real React page in PMS, we add the RPC. |
| Edit transitions independently from actions | Auto-deriving transitions from actions is simpler and matches reality 95%+ of the time. Manual transition editing in v2 if a real case emerges. |
| Custom color picker | 8-color palette is enough; matches the design system. |
| Status key renaming | Status `key` is immutable (referenced by FK-like in payloads + audit_log). Only `label` is editable. New keys are auto-generated. |
| Multi-property workflow override | Single global workflow per request type. Per-property variation in v2. |
| Diff view (current vs seed vs DB) | Useful but not v1; the dirty state visualisation is enough for the operator. |
| Audit log of who edited what | Comes when this is rebuilt in PMS proper with auth. |

---

## 4. Data model

No new schema. The editor reads + writes `portal.request_types.workflow` (jsonb) which already exists.

**Schema reminder** (see [`/lib/portalRequestTypes.ts`](https://github.com/hassanateeq-collab/hamsun-guest-manager/blob/main/src/lib/portalRequestTypes.ts) for full TS):

```ts
RequestWorkflow {
  initial_status: string
  subtypes?: string[]
  severity_levels?: string[]
  statuses: WorkflowStatus[]
  transitions: WorkflowTransition[]
  actions: WorkflowAction[]
}

WorkflowStatus { key, label, color, is_active, is_terminal, sla_minutes? }
WorkflowAction { key, label, from?, from_any_active?, to, requires_note?, requires_assignee?, requires_scheduled_for?, requires_role?, destructive?, posts_charge?, notify_guest_on_action?, notification_template? }
WorkflowTransition { from, to: string[] }
```

The seeded workflows for all 7 request types are inlined in `index.html` `SEEDED` constant — kept verbatim as fetched from the production DB via `mcp__supabase__execute_sql` on 2026-04-27.

---

## 5. Outstanding observations on seeded workflows

These are the issues identified during review. Each needs operator decision before the migration is applied:

### Housekeeping
- `cleaning` subtype belongs to the new housekeeping module, not items — remove
- Terminal label `done` → consider `delivered` for clarity on item delivery
- No `notify_guest_on_action` set anywhere — guest never gets a "your request is done" message

### Kitchen / F&B
- SLA 15m may be tight at peak breakfast time
- Linear flow forces an extra click for verbal kitchen orders that arrive already confirmed
- `breakfast` subtype overlaps with separate breakfast pre-order menu system — ad-hoc only?

### Airport / Transport
- Team set to `concierge` but reception currently handles transport — rename to `reception`
- `assign_driver` action missing `requires_assignee` flag
- `no_show` posts no charge — consider 50% no-show fee action
- No surfacing of upcoming pickups (next 24h band) — UI concern, not workflow

### Maintenance
- Strongest workflow of the seven; recommend keep as-is
- `severity_levels` overlaps `urgency_tier` — pick one
- No notification on `resolve` — guest deserves WhatsApp confirmation

### Booking actions
- `requires_role: manager` permission key not enforced anywhere yet
- No amount-capture field on `approve` (late checkout / extension price)
- Mixed flows — late_checkout / extend_stay / room_change have different needs; consider splitting

### Concierge / General
- Wake-up call needs `requires_scheduled_for` — split out as own request_type
- Complaint needs escalation paths — split out
- Lost-and-found has its own lifecycle — split out
- No notification on `cannot_fulfill` — guest waits wondering

### Mini-shop
- "Approve" reads wrong; rename to "Confirm" + status `confirmed`
- "Start preparing" doesn't fit pre-stocked items; collapse or rename to "Picking"
- Stock decrement on `delivered` not wired
- Tobacco subtype — Pakistani regulations / time-of-day restrictions

---

## 6. Build order — once vocabulary is locked

| Phase | Work | Effort |
|---|---|---|
| 1 | Operator goes through editor, makes vocabulary edits per §5 observations | 1 hr |
| 2 | Click "Generate SQL migration" → hand to me to apply via `mcp__supabase__apply_migration` | 15 min |
| 3 | Verify reception's `WorkflowActionBar` renders correct buttons against new vocabulary | 30 min |
| 4 | Build real React admin page in PMS (`src/pages/RequestTypesSettingsPage.tsx`) replacing this prototype | 8 hr |
| 5 | Wire to new RPC `portal_update_request_type_workflow(slug, jsonb)` with permission gate `settings.workflows` | 2 hr |
| 6 | Audit log: capture old vs new workflow on every save into `audit_log` table | 1 hr |
| **Total** | | **~12.75 hours** (after vocabulary lock) |

---

## 7. Files

```
workflows-admin/
├── index.html              ← editor (every field editable, generates SQL)
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4325
```

---

## 8. Reference data

The editor's `SEEDED` constant in `index.html` is a frozen snapshot of `portal.request_types.workflow` from the production DB on **2026-04-27**. If workflows are modified directly in prod (via SQL migration applied), this snapshot must be regenerated by re-running the SELECT query in the editor's seed logic.

Production DB project ID: `zanpnhfcuqznmmokbchv`.

---

*Locked once vocabulary fixes are applied. Then proceed to Phase 4 (rebuild as React page in PMS).*
