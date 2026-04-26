# Reception Inbox v2 — Design Lock

Status: **prototype iterating, ready for React rebuild in PMS**
Owner: Hassan Ateeq · Last updated: 2026-04-27
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4324`

---

## 1. What this module is

The redesigned reception command center — a single-screen operational dashboard for front-desk staff to triage and resolve guest requests across all teams. Replaces the current `RequestsPage.tsx` in the PMS.

**Design intent** (per operator):
- Reception staff are non-technical — large tap targets, plain labels ("✓ Done" not "Mark request as completed")
- Reception sees ALL teams' work even though kitchen / housekeeping / cleaning have their own pages, so reception knows what's pending and overdue
- "Neat, clean, clear, with no mistakes possible" — undo on every action

---

## 2. What's locked

### 2.1 Tab structure — 4 teams, 4 dedicated views

| Tab | Icon | Owner | Content |
|---|---|---|---|
| Reception | 📋 | Reception | Reception's own queue (transport, F&B, mini-shop, concierge, etc.) |
| Kitchen | 🍳 | Kitchen | Tomorrow's breakfast: stat cards, per-slot blocks, "Not ordered yet" list |
| Housekeeping | 🧺 | Housekeeping team | Items requests (towels, linen, amenities) |
| Cleaning | 🧹 | Cleaning team | Cleaning jobs (daily, deep, turn-down, post-checkout) — once `housekeeping-module` ships, this folds into it |

Each tab shows live count badge in the tab. Reception tab pulses red when there are items in "Needs your attention now".

### 2.2 Reception view — command center layout

The reception view is the operator's primary workspace. Layout:

```
┌─────────────────────────────────────────────────────────────┐
│ OVERDUE STRIP (auto-shows when anything is overdue, anywhere)│
│   Cross-team list of overdue rows with team chip + jump btn │
├─────────────────────────────────────────────────────────────┤
│ NEEDS YOUR ATTENTION NOW (urgent reception items)            │
│   Pulsing dot, ember border, top of fold                    │
├──────────────────────────┬──────────────────────────────────┤
│ LEFT COLUMN (60%)        │ RIGHT COLUMN (40%, sticky)       │
│ Today's reception requests│ Kitchen peek card               │
│  (active list)           │   Stat row, per-slot summary,    │
│ Done today (collapsed)   │   "Remind 12 / Open kitchen"     │
│                          │ Housekeeping peek card           │
│                          │ Cleaning peek card               │
└──────────────────────────┴──────────────────────────────────┘
```

### 2.3 Cross-team overdue strip

- Hidden by default; renders only when ≥1 overdue item across teams
- Pulls from: reception NEEDS_ACTION urgency=overdue, housekeeping items >30m, kitchen reminded-no-reply
- Each row: team chip · room # · request · time-overdue · "Go to {team}" button

### 2.4 Team peek cards (right column)

Each team gets a sticky card showing:
- 3-stat row (new / in progress / done) with amber/red highlighting on backlog
- Top 3 oldest items as mini-rows
- Two foot buttons: **Nudge** team lead (or "Remind 12" for kitchen) + **Open team →** (jumps to that tab)

Card border + header turns ember when team has overdue items.

### 2.5 List row pattern (universal)

Every list across all 4 tabs uses the same row grid:

```
┌────┬─────┬─────────────────────┬────────┬─────────┐
│Time│Room │Main (icon, request, │ Status │ Actions │
│    │     │ price, meta line)   │ pill   │         │
└────┴─────┴─────────────────────┴────────┴─────────┘
50px  70px        1fr             110px    auto
```

Rows highlight: ember tint for `urgent`/`overdue`, moss flash on `recently-changed`, faded for `done`.

### 2.6 Action buttons — driven by workflow

Buttons rendered per row come from the request type's `workflow.actions` filtered by current status (matches the existing `WorkflowActionBar.tsx` pattern in PMS). Button kinds:

| Kind | Style | Use |
|---|---|---|
| `primary` | Black solid | Main affirmative action (Confirm, Done, Approve) |
| `secondary` | White with border | Helper actions (Assign, Triage, Set time) |
| `danger` | White with ember border + text | Destructive (Reject, Cancel) — gets 2-tap confirm |
| `more` | Transparent ⋯ | "More options" overflow |

### 2.7 Undo system — no accidents possible

Every state-changing action shows an undo toast at bottom-right for **7 seconds**:

- Black for routine ("✓ Done · R208 Toiletries refill")
- Green for approvals ("Approved · R G09 late checkout — folio +PKR 6,000")
- Red for cancellations ("✕ Cancelled · R110 Mini-shop")
- Big "↶ Undo" button + animated timer bar
- Single toast at a time (replaces previous)

Snapshot taken before mutation; undo restores from snapshot.

### 2.8 2-tap confirm for destructive actions

Reject / Cancel / Decline use a two-tap pattern:

1. First tap → button shakes, turns ember/red, label changes to "⚠ Tap again to reject"
2. 3-second timer to second tap
3. Second tap → action commits, undo toast appears for 7s
4. No second tap → button silently reverts

Net effect: a single accidental tap on Reject does nothing; an intentional double-tap can still be undone.

---

## 3. What's parked

| Parked | Why |
|---|---|
| Multi-property switching from this view | Property selector in topbar handles it; no in-flight property change |
| Drag-to-reorder rows | Spec is fixed by time/urgency; no reordering needed |
| Inline charge editing on a folio row | Folio mutations go through booking detail page, not the inbox |
| Realtime push updates (websocket) | Polling at 15s interval is enough for v1 (matches existing `useRequestList` hook config) |
| Voice mode | Not on roadmap |
| Multi-staff assignment per task | Single assignee; team handoff via reassignment |

---

## 4. Data model

No new schema. The reception inbox reads from existing `portal.requests`, `portal.request_types`, `bookings`, and `rooms` via existing RPCs:

- `portal_dashboard_summary` — counts for tab badges + needs-attention
- `portal_list_requests` — main list with filters (category, status, only_active, only_needs_attention)
- `portal_get_request` — detail drawer
- `portal_update_request_status` — fires on every action button click
- `portal_create_request` — for the "+ New manual entry" buttons

These all exist today (see `src/lib/portalRequestApi.ts`). No backend work needed for the inbox itself — the current PMS already has them wired.

---

## 5. Differences from current PMS `RequestsPage.tsx`

The existing page is functional but doesn't surface cross-team awareness or the command-center layout. Specific deltas:

| Aspect | Current PMS | This redesign |
|---|---|---|
| Layout | Single column scroll | 2-column grid + overdue strip |
| Cross-team visibility | None — reception has to switch pages | Right-column peek cards + overdue strip |
| Action confirmations | Browser `confirm()` | 2-tap inline + undo toast |
| Mistake recovery | None for marked-done | 7-second undo on every action |
| Urgent items | Listed in main flow | Dedicated "Needs your attention now" panel at top |
| Status pills | Standard shadcn badges | Custom typography pills with color tokens matching design language |
| Done queue | Mixed with active | Collapsed by default, separate block |

---

## 6. Build order — rebuilding as React in PMS

| Phase | Work | Effort |
|---|---|---|
| 1 | Add design tokens to PMS Tailwind config (colors, fonts, --r-pill etc.) | 1 hr |
| 2 | Build new components: `OverdueStrip.tsx`, `NeedsNowPanel.tsx`, `TeamPeekCard.tsx`, `UndoToast.tsx`, `ListRow.tsx` (replaces existing request rows) | 8 hr |
| 3 | Refactor `RequestsPage.tsx` into the 4-tab structure | 4 hr |
| 4 | Wire the existing `useRequestList` hook to drive each tab + peek cards | 3 hr |
| 5 | Add undo system: snapshot pattern + toast manager + connection to mutations | 4 hr |
| 6 | Add 2-tap confirm pattern to destructive `WorkflowActionBar` buttons | 2 hr |
| 7 | Build the cross-team overdue aggregator (logic for which items qualify) | 3 hr |
| 8 | Mobile responsive (single column collapses, peek cards stack below) | 2 hr |
| 9 | A11y pass (keyboard nav, ARIA, focus management for toasts) | 3 hr |
| 10 | E2E test on dev branch with real production-like data | 4 hr |
| **Total** | | **~34 hours** |

---

## 7. Visual design

Color tokens (matching the menu-admin / breakfast-prototype palette):

```css
--ink: #1A1A1A;           /* primary text */
--ink-2: #2C2C2C;
--ink-3: #5A5A5A;
--ink-4: #8B8B8B;         /* meta text */

--gold: #C8A45E;
--gold-mist: #F5E6C3;
--gold-dark: #9E7F43;     /* accent + emphasis */

--cream: #FAF8F4;         /* background */
--paper: #FFFFFF;         /* card */
--edge: #EBE6DD;          /* borders */

--moss / moss-mist        /* success / done */
--ember / ember-mist      /* destructive / overdue */
--amber / amber-mist      /* pending / new */
--orange / orange-mist    /* warn / immediate */
```

Typography: Fraunces (display) + Inter (text). No emojis in production code per `CLAUDE.md` — emojis in the prototype are for visual demo only and may be replaced with icon set in React rebuild.

---

## 8. Files

```
reception-inbox/
├── index.html              ← prototype (4 tabs, command center, undo toasts)
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4324
```

---

*Locked. Proceed with Phase 1 of build order when ready to migrate to PMS React.*
