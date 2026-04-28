# Reception Inbox · Table-Style Variant

Status: **comparison mockup — sibling of `reception-inbox/`**
Owner: Hassan Ateeq · Created: 2026-04-28
Prototype: [`index.html`](./index.html) · preview at `http://localhost:4328` (local) or via umbrella at `:4320/reception-inbox-table/`

---

## What this is

A table-style alternate of the [reception-inbox](../reception-inbox/) prototype. Same data, same workflow, same peek cards on the right — only the **left-column request rows** are rendered differently.

This exists for direct A/B comparison. Both prototypes are kept side-by-side; once a row-style is chosen for production, the other can be retired.

---

## Difference from `reception-inbox/`

| Element | `reception-inbox/` (cards) | `reception-inbox-table/` (this) |
|---|---|---|
| Row layout | CSS grid: 5 columns, multi-line content (request + meta below) | HTML `<table>`: 7 columns, single-line content per row |
| Header row | None | Yes — column labels (Time · Room · Request · Guest · Source · Status · Actions) |
| Row height | ~62px (two text lines) | ~44px (one text line) |
| Density | Medium | High |
| Information style | Narrative ("via Guest portal · 10m ago") | Compact ("Portal · 10m") |
| Hover treatment | Soft background tint | Soft background tint (same) |
| Mobile | Stacks naturally | Falls back to stacked on `< 760px` |

Everything else is unchanged: topbar, tab bar, overdue strip, peek cards, undo toast styling, 2-tap confirm pattern, color palette, typography.

---

## Why "table-like but not tabular"

The visual treatment splits the difference between a list and a spreadsheet:

✅ **Borrowed from tables:**
- Header row with column labels
- Fixed column widths and alignment
- Single-line per row (predictable height)
- Strict information hierarchy

❌ **Avoided from tables:**
- No hard cell borders / grid lines
- No alternating row stripes (zebra)
- No tight spreadsheet padding — keeps breathing room
- No data-entry feel

The intent is fast triage scanning without the clinical-form feel of an Excel sheet.

---

## Comparison decision

This is here so the operator can pick. Three outcomes possible:

1. **Adopt table-style** — `reception-inbox-table/` becomes the production design target. The original `reception-inbox/` is archived as a reference.
2. **Adopt card-style** — `reception-inbox/` stays the production target. This prototype is archived.
3. **Hybrid** — table-style for the high-density Active queue, card-style for the urgent Needs-Now panel and Done list.

Until the operator decides, both prototypes coexist.

---

## Files

```
reception-inbox-table/
├── index.html              ← table-style mockup
├── DESIGN.md               ← this file
└── .claude/launch.json     ← preview server on port 4328
```

---

*Reference only · do not modify after the operator picks a direction.*
