# Grist Wide Pie Widget

A custom Grist widget that draws a **pie chart from numeric columns of the row you are viewing**, or (optionally) from values across table records—no reshaping your table first.

## Using the widget in Grist

You do **not** need to publish or install anything yourself if you use the hosted build:

**[https://federicobuttafuori.github.io/Grist-Wide-Pie-Widget/](https://federicobuttafuori.github.io/Grist-Wide-Pie-Widget/)**

1. In your Grist document, add a widget to the page and choose **Custom widget** (or **Create custom widget**, depending on your Grist UI).
2. When asked for the widget URL, **paste the link above** into the **Custom widget URL** field.
3. Point the widget at the table you want. Open the widget settings (⚙) and choose **Data mode**:
   - **Columns of selected record** (default): pick numeric columns as slices; the chart follows the active row.
   - **Records in the table**: pick a **Value column** and a **Label column**; each visible/filtered row becomes a slice.

If you self-host the files instead, use your own HTTPS URL in the same **Custom widget URL** field.

## The problem

In standard Grist charts, if you want to see how `Cost_Meat`, `Cost_Shipping`, and `Cost_Packaging` contribute to a total, you usually have to transform your data or create complex summary tables.

**Grist Wide Pie Widget** solves this by letting you pick columns directly from your **active record** and turn them into slices—**no transformation required**.

## What you get

- A pie chart from the **current row** (column slices) or from **all visible rows** (record slices).
- Legend, colors, and totals so proportions are easy to compare.
- A good fit for budgets, recipes, cost breakdowns, or any table where many numbers live on the same row.

## Who it is for

- People who analyze **one row at a time** (e.g. per-product or per-order breakdowns).
- Anyone who wants a quick visual instead of scanning many numeric cells.

