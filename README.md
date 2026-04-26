# Hamsun · Prototypes

Umbrella repo for all design-stage prototypes and mockups across the Hamsun PMS / guest portal / staff tools. Each subfolder is a self-contained module with its own preview server + design lock document.

The PMS itself lives separately at [`hassanateeq-collab/hamsun-guest-manager`](https://github.com/hassanateeq-collab/hamsun-guest-manager). Anything in this repo is design / spec / mockup — not production code.

---

## Modules

| Folder | What it is | Status | Preview port | Design lock |
|---|---|---|---|---|
| [`menu-admin/`](./menu-admin) | Multi-menu admin (breakfast / lunch / dinner / snacks / etc.) — pricing modes, time slots, choice groups, KOT config | Locked, ready for backend | `:4323` | [`DESIGN_LOCK.md`](./menu-admin/DESIGN_LOCK.md) |
| [`breakfast-prototype/`](./breakfast-prototype) | Guest-facing breakfast pre-order portal (linked via WhatsApp) | Locked | `:4322` | [`DESIGN.md`](./breakfast-prototype/DESIGN.md) |
| [`portal-mockup/`](./portal-mockup) | Mini-shop / sundries item picker (referenced by guest portal request flow) | Locked | `:4321` | [`DESIGN.md`](./portal-mockup/DESIGN.md) |
| [`reception-inbox/`](./reception-inbox) | Reception command center — 4-tab inbox with cross-team peek cards, undo toasts | Iterating | `:4324` | [`DESIGN.md`](./reception-inbox/DESIGN.md) |
| [`workflows-admin/`](./workflows-admin) | Editor for `portal.request_types.workflow` jsonb — statuses, actions, charge points, flags | Iterating | `:4325` | [`DESIGN.md`](./workflows-admin/DESIGN.md) |
| [`housekeeping-module/`](./housekeeping-module) | Auto-cleaning module — daily cron, post-checkout trigger, WhatsApp opt-in, guest portal toggle | Spec'd, awaiting Q1–Q8 | `:4326` | [`DESIGN.md`](./housekeeping-module/DESIGN.md) |

---

## How to run a prototype

Each folder has a `.claude/launch.json` that boots a Python static-file server on its assigned port. To run any prototype manually:

```bash
cd menu-admin
python3 -m http.server 4323
# now open http://localhost:4323
```

Multiple prototypes can run concurrently — they're on different ports.

---

## How to add a new prototype

1. Create a sibling folder (e.g. `cleaning-display/`).
2. Add `index.html` with the mockup.
3. Add `.claude/launch.json` with a fresh port (next free one — `4327`+).
4. Write `DESIGN.md` capturing what's locked vs. parked, schema additions, edge functions, build order. Use [`menu-admin/DESIGN_LOCK.md`](./menu-admin/DESIGN_LOCK.md) as the canonical example.
5. Update this README's table.
6. Commit.

---

## Pattern · why prototype here first

Every Hamsun module gets a paired prototype + design lock before backend code is written:

1. Build the HTML mockup interactively (in this repo).
2. Iterate with the operator (Hassan) until the UX is right.
3. Convert the locked mockup into a `DESIGN.md` spec — what's in v1, what's parked, schema, edge functions, build order, effort estimate.
4. Hand off to backend implementation in `hamsun-guest-manager`.
5. Keep this prototype + spec frozen as historical reference. Update only when the design changes.

This avoids the trap of building backend infrastructure for an unclear UX, then having to reshape it three weeks later.
