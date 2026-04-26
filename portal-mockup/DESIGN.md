# Portal Mockup (Mini-Shop) — Design Lock

Status: **locked, embedded into guest portal request flow**
Owner: Hassan Ateeq
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4321`
Catalog data: [`items.json`](./items.json)

---

## 1. What this module is

The **guest-facing mini-shop / sundries picker** — accessed from the guest portal under the "Mini-shop" request category. Guests browse a small inventory of sundries (drinks, snacks, OTC meds, phone accessories, toiletries, packing supplies), tap to add to cart, hit submit. Reception fulfils, posts to folio.

This is the visual mockup; the live integration lives in [`hamsun-portal/src/pages/Request.tsx`](https://github.com/hassanateeq-collab/hamsun-portal) which renders dynamic items from `portal.catalog_items` via the `portal_list_catalog_items` RPC.

---

## 2. What's locked

### 2.1 Catalog structure (per `items.json`)

Property + room context surfaces at top: e.g. *"Hamsun Shahr-e-Faisal · Room 301"*.

7 categories:
- `all` — show everything
- `drinks` — juices, sparkling water, etc.
- `munchies` — chocolate, cookies, chips
- `otc_meds` — wellness items (most are `comp: true` — free of charge)
- `phone` — cables, adapters
- `toiletries` — eye mask, sanitary pads, nail cutter
- `packing` — packing tape

Sample seeded items (20 total):

| Category | Examples |
|---|---|
| Drinks | Rani Orange (PKR 400) · Day Fresh Belgian Chocolate Milk (PKR 200) |
| Munchies | Cadbury Dairy Milk (PKR 200) · Lays Classic (PKR 100) |
| OTC meds | Panadol · Imodium · ENO · ORS — all comp (free) |
| Phone | Lightning cable (PKR 850) · Type-C cable (PKR 800) · USB adapter (PKR 1,000) |
| Toiletries | Nail cutter (PKR 400) · Eye mask (PKR 200) · Sanitary pads (PKR 250) |
| Packing | Brown packing tape (PKR 250) |

### 2.2 Item schema

Each item has:

```json
{
  "sku": "MS-DRNK-002",
  "name": "Rani Orange Juice",
  "size": "240ml can",
  "sub": "drinks",
  "price": 400,
  "stock": 16,
  "low": false,
  "comp": false,
  "img": "https://images.unsplash.com/..."
}
```

- `comp: true` items are zero-priced (toiletry refills, OTC essentials)
- `low: true` flag surfaces a "low stock" tag
- `stock` drives availability — items with `stock = 0` and `tracks_stock = true` are hidden by `portal_list_catalog_items` (`hide_out_of_stock = true`)

### 2.3 Guest interaction

- Tap a category chip to filter
- Tap an item to add 1 unit; tap again to bump qty
- Cart panel slides up with total
- Checkout: confirms with reception, posts request via `portal-submit-request` edge function with `type_slug = 'mini_shop'`

### 2.4 Pricing semantics

Each item's `pricing_mode` (in the live `catalog_items` schema):

| Mode | Effect |
|---|---|
| `free` | `comp: true` — never charged |
| `fixed` | Charged at `default_unit_price` × qty |
| `manual` | Reception adjusts at delivery (rare for mini-shop) |
| `percent_of_nightly_rate` | Used by booking_action items (late_checkout, etc.) — not mini-shop |

---

## 3. What's parked

| Parked | Why |
|---|---|
| Real product photography pipeline | URL paste works; uploader is a separate Storage-bucket task |
| Per-property catalog differences | Each property has its own catalog rows now (`property_id` set) but admin UI for per-property editing is later |
| Loyalty discounts on mini-shop items | Not in scope |
| Guest-facing inventory level visibility ("Only 2 left!") | Causes anxiety; show `low: true` only |
| Wishlist / favourites across stays | Single-stay scope |

---

## 4. Live integration status

The catalog schema is already implemented in production. Catalog items load via `portal_list_catalog_items`. New requests submit via `portal-submit-request` edge function. The visual layer in this mockup has been ported into [`hamsun-portal/src/pages/Request.tsx`](https://github.com/hassanateeq-collab/hamsun-portal/blob/main/src/pages/Request.tsx) under the `mini_shop` request_type category.

The mockup remains as a frozen reference of the intended visual design + interaction pattern.

---

## 5. Files

```
portal-mockup/
├── index.html              ← visual mockup
├── items.json              ← seeded catalog data (20 items, 7 categories)
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4321
```

---

*Locked. Live equivalent in `hamsun-portal/src/pages/Request.tsx`.*
