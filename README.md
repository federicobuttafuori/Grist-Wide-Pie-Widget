# Grist Wide Pie Widget

A single-file HTML **custom widget** for [Grist](https://www.getgrist.com/). It draws a **pie chart** from the **active row** (wide layout), using the numeric columns you select.

## Files

| File | Purpose |
|------|---------|
| `grist-wide-pie-widget.html` | Full widget (UI, logic, canvas) |
| `docs/debug-learnings.md` | Notes on Grist host quirks and historical fixes |

## Installing in Grist

1. Open a Grist document and add a **Custom widget**.
2. Paste the contents of `grist-wide-pie-widget.html` into the widget editor (or upload the file if your Grist build supports it).
3. Point the widget at the right table and ensure a **row is selected** (active record).

## Features

- **Column selection**
  - Manual checkboxes over detected “numeric” columns.
  - Optional **include regex**: adds columns whose name matches.
  - Optional **exclude regex**: removes from the current set.
- **Values**: skips empty, non-numeric, null, and zero.
- **Labels and colors**: per selected column, custom label and color.
- **Number formatting**: prefix and decimals for legend / tooltip values.
- **Total**: overlaid at the top-left of the chart (does not consume layout space).
- **On-slice labels**: for slices above a minimum size (percentage threshold), column name in a darker tint of the slice color with a light drop shadow.
- **Slice tooltip** on hover: name, value, percentage (percentage shown with one decimal place).

## Persisting settings

- **Grist document**: when available, options are saved with `grist.setOptions` and restored with `grist.onOptions`.
- **Custom widget builder**: `setOptions` may be unreliable; the widget also uses **`localStorage`** (`grist-wide-pie-widget:options:v1`) so settings survive pasting new code into the builder.

## Behavior in the builder

In Grist’s custom widget builder, the active row may arrive via **polling** (`fetchSelectedRecord`). The widget **merges** snapshots for the same row so `undefined` cells do not overwrite already valid data (avoids chart flicker).

See `docs/debug-learnings.md` for details.

## Debug

When the settings panel is open, a debug button opens a log window with **Copy** / **Clear**, draggable via the header, and closable with **X**.

## Requirements

- Grist with the widget API (`grist.ready`, read access to the table).
- A modern browser with canvas and `localStorage` (for the builder fallback).
