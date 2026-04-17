# Grist Wide Pie Widget

**What problem it solves:** Grist tables are often **wide**—many numeric measures live as **columns on one row**. Most chart defaults assume **long** data (one measure per row). This widget maps the **active row** to a pie so you can compare slice shares (e.g. cost breakdown) **without pivoting** the sheet or maintaining helper tables.

A single-file HTML **custom widget** for [Grist](https://www.getgrist.com/). It draws a **pie chart** from the **active row** (wide layout), using the numeric columns you select. It is designed for “one record at a time” analysis: each chosen numeric column in the current row becomes a slice.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Full widget (UI, logic, canvas, Grist integration) |
| `docs/debug-learnings.md` | Notes on Grist host quirks and historical fixes |

---

## What this widget does

- Reads the **active record** from Grist.
- Detects numeric columns and lets you choose which to include (manual list, include regex, exclude regex).
- Skips empty, null, non-numeric, and **zero** values.
- Draws a pie on `<canvas>` with legend, hover tooltip, optional on-slice labels (name), and a **total** overlaid at the top-left of the chart.
- **Labels and colors:** per selected column, optional custom label and color.
- **Number formatting:** prefix and decimals for legend and tooltip; percentages with one decimal place in the tooltip.
- **Column descriptions:** when Grist exposes them (including via internal metadata), legend rows can show the description on hover.

---

## Quick start (Grist)

1. Open your Grist document and add a **Custom widget**.
2. Paste the full contents of `index.html` into the widget code editor (or upload the file if your deployment supports it).
3. The HTML **includes** `https://docs.getgrist.com/grist-plugin-api.js` so `window.grist` exists when the widget is served as an external URL. Do not remove that tag. (Self‑hosted Grist: if your CSP blocks that origin, point the `src` at your instance’s `grist-plugin-api.js` instead.)
4. Point the widget at the target table.
5. Select a **row** in that table. If no row is selected, there is no active record to visualize.

---

## Configuration

- **Manual selection** — checkbox list of detected numeric columns.
- **Include regex** — auto-include columns whose name matches.
- **Exclude regex** — remove matching columns from the current set.
- **Per-column** — custom label and color for selected columns.
- **Number format** — legend/tooltip prefix and decimal places.
- **On-slice labels** — shown when the slice is large enough (minimum percent threshold); uses column name styling.

---

## Persisting settings

1. **Primary:** Grist document options (`grist.setOptions` / `grist.onOptions`) when available.
2. **Fallback:** browser `localStorage` key `grist-wide-pie-widget:options:v1` — especially useful in the **Custom Widget Builder**, where `setOptions` can be unreliable while you paste new code.

---

## Behavior in the Custom Widget Builder

The active row is driven by **`grist.onRecord` / `grist.onRecords`** only (no polling). The widget **merges** snapshots for the same row so partial records with `undefined` cells do not overwrite already valid values (avoids chart flicker). See `docs/debug-learnings.md` for details.

---

## Debug

When the settings panel is open, a debug button opens a log window with **Copy** / **Clear**, draggable header, and close (**X**). The panel shows a short **summary** (host, row, column counts, chart state) plus a small rolling log for errors.

---

## Requirements

- Grist with the widget API (`grist.ready` with **`requiredAccess: "full"`** so `docApi` can load table metadata and internal `_grist_*` tables for column descriptions) and custom widget support.
- A modern browser with Canvas and `localStorage` (for the builder fallback).

---

## Development notes

- The widget is intentionally a **single HTML file** for easy copy/paste into Grist.
