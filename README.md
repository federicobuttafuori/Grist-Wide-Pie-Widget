# Grist Wide Pie Widget

A custom Grist widget that draws a **pie chart from the row you are viewing**: you choose which numeric columns are slices, and the chart updates with the active record—no reshaping your table first.

## Using the widget in Grist

You do **not** need to publish or install anything yourself if you use the hosted build:

**[https://federicobuttafuori.github.io/Grist-Wide-Pie-Widget/](https://federicobuttafuori.github.io/Grist-Wide-Pie-Widget/)**

1. In your Grist document, add a widget to the page and choose **Custom widget** (or **Create custom widget**, depending on your Grist UI).
2. When asked for the widget URL, **paste the link above** into the **Custom widget URL** field.
3. Point the widget at the table you want, select the columns that should become slices, and use the chart as you move between rows.

If you self-host the files instead, use your own HTTPS URL in the same **Custom widget URL** field.

## The problem

In standard Grist charts, if you want to see how `Cost_Meat`, `Cost_Shipping`, and `Cost_Packaging` contribute to a total, you usually have to transform your data or create complex summary tables.

**Grist Wide Pie Widget** solves this by letting you pick columns directly from your **active record** and turn them into slices—**no transformation required**.

## What you get

- A pie chart tied to the **current row**; each selected metric is a slice.
- Legend, colors, and totals so proportions are easy to compare.
- A good fit for budgets, recipes, cost breakdowns, or any table where many numbers live on the same row.

## Who it is for

- People who analyze **one row at a time** (e.g. per-product or per-order breakdowns).
- Anyone who wants a quick visual instead of scanning many numeric cells.
