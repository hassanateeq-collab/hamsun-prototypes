# Hamsun · Prototypes

Umbrella repo for design-stage prototypes and mockups across the Hamsun PMS, guest portal, and staff tools. Each subfolder is a self-contained module with its own interactive `index.html` mockup + a `DESIGN.md` spec ready for backend handoff.

The PMS itself lives separately at [`hassanateeq-collab/hamsun-guest-manager`](https://github.com/hassanateeq-collab/hamsun-guest-manager). Anything here is design / spec / mockup — not production code.

---

## Live online (GitHub Pages)

Once Pages is enabled in repo settings, every prototype is accessible at:

```
https://hassanateeq-collab.github.io/hamsun-prototypes/                          ← landing page (index)
https://hassanateeq-collab.github.io/hamsun-prototypes/master-flow/                ← architecture spine (read first)
https://hassanateeq-collab.github.io/hamsun-prototypes/menu-admin/
https://hassanateeq-collab.github.io/hamsun-prototypes/breakfast-prototype/
https://hassanateeq-collab.github.io/hamsun-prototypes/portal-mockup/
https://hassanateeq-collab.github.io/hamsun-prototypes/reception-inbox/
https://hassanateeq-collab.github.io/hamsun-prototypes/workflows-admin/
https://hassanateeq-collab.github.io/hamsun-prototypes/housekeeping-module/
```

Open on phone, desktop, share with a designer, demo to anyone. Auto-redeploys on every push to `main`.

---

## Modules

| Folder | What it is | Status | Design lock |
|---|---|---|---|
| [`master-flow/`](./master-flow) | **Architecture spine** — entry point, stay state, 8 sub-flows, data model, edge functions, build sequence, invariants | **Read first** | [`DESIGN.md`](./master-flow/DESIGN.md) |
| [`menu-admin/`](./menu-admin) | Multi-menu admin (breakfast / lunch / dinner / snacks) — pricing modes, time slots, choice groups, KOT config | Locked | [`DESIGN_LOCK.md`](./menu-admin/DESIGN_LOCK.md) |
| [`breakfast-prototype/`](./breakfast-prototype) | Guest-facing breakfast pre-order portal (linked via WhatsApp) | Locked | [`DESIGN.md`](./breakfast-prototype/DESIGN.md) |
| [`portal-mockup/`](./portal-mockup) | Mini-shop / sundries item picker (referenced by guest portal request flow) | Locked | [`DESIGN.md`](./portal-mockup/DESIGN.md) |
| [`reception-inbox/`](./reception-inbox) | Reception command center — 4-tab inbox with cross-team peek cards, undo toasts | Iterating | [`DESIGN.md`](./reception-inbox/DESIGN.md) |
| [`workflows-admin/`](./workflows-admin) | Editor for `portal.request_types.workflow` — statuses, actions, charge points, flags. Generates SQL migration | Iterating | [`DESIGN.md`](./workflows-admin/DESIGN.md) |
| [`housekeeping-module/`](./housekeeping-module) | Auto-cleaning module — daily cron, post-checkout trigger, WhatsApp opt-in, guest portal toggle | Spec'd, awaiting Q1–Q8 | [`DESIGN.md`](./housekeeping-module/DESIGN.md) |

---

## Running locally

**One server, all modules** — recommended (matches GitHub Pages URL structure):

```bash
git clone git@github.com:hassanateeq-collab/hamsun-prototypes.git
cd hamsun-prototypes
python3 -m http.server 4320
# now open http://localhost:4320
```

The landing page at `:4320/` has cards linking to each module. Cross-links between modules use relative paths, so they work the same locally and on GitHub Pages.

**Per-module servers** — only needed if you want each module on a dedicated port:

| Module | Port |
|---|---|
| `portal-mockup` | `:4321` |
| `breakfast-prototype` | `:4322` |
| `menu-admin` | `:4323` |
| `reception-inbox` | `:4324` |
| `workflows-admin` | `:4325` |
| `housekeeping-module` | `:4326` |
| `master-flow` | `:4327` |

Each folder has a `.claude/launch.json` configured for its dedicated port. Run `python3 -m http.server <port>` from inside the module folder.

---

## Adding a new prototype

1. Create a sibling folder at the umbrella root (e.g. `cleaning-display/`).
2. Add `index.html` with the mockup.
3. Add `.claude/launch.json` with the next free port (`4327`+).
4. Write `DESIGN.md` capturing what's locked, what's parked, schema, edge functions, build order. Use [`menu-admin/DESIGN_LOCK.md`](./menu-admin/DESIGN_LOCK.md) as the canonical example.
5. Add the module as a card on the umbrella `index.html` landing page.
6. Update this README's modules table.
7. Commit + push — the new module is live on Pages within ~30 seconds.

---

## Pattern · why prototype here first

Every Hamsun module gets a paired prototype + design lock before backend code is written:

1. Build the HTML mockup interactively (in this repo).
2. Iterate with the operator until the UX is right.
3. Convert the locked mockup into a `DESIGN.md` spec — what's in v1, what's parked, schema, edge functions, build order, effort estimate.
4. Hand off to backend implementation in `hamsun-guest-manager`.
5. Keep this prototype + spec frozen as historical reference. Update only when the design changes.

This avoids the trap of building backend infrastructure for an unclear UX, then having to reshape it three weeks later.

---

## Conventions

- One `index.html` per module — entry point. No build step, no bundlers.
- One `DESIGN.md` per module — spec + decisions + schema + build order.
- Cross-module links use **relative paths** (`../module-name/`), never absolute or `localhost:`.
- Tailwind / shadcn / React etc. live in the production PMS, not in prototypes. Prototypes are vanilla HTML/CSS/JS for fast iteration.
- Design tokens are duplicated per module HTML (cream / gold / Fraunces / Inter) — intentional, keeps each prototype standalone.
