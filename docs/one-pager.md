# AskDB: One-Pager

A web app where business users ask questions of a database in plain English.

## Who it's for
- Analysts and managers in business roles.
- They are not database experts, and they need numbers for reports and decisions.

## What they ask
- **Totals and KPIs**, such as revenue by region.
- **Trends over time**, such as signups month over month.
- **Lookups of specific records**, such as open orders for one customer.
- **Comparisons and rankings**, such as the biggest drops versus last year.

## What they need to see
- A table of results.
- Charts, where they suit the question.
- A plain-English summary of what the numbers mean.
- Export, such as CSV or Excel.
- A clear view of how the question was interpreted, so users can check and trust the answer.

## What must never happen
- Changing or deleting data. The app is strictly read-only.
- Users seeing data their role doesn't permit.
- Confident wrong answers. When a question is ambiguous or can't be answered, the app says so instead of guessing.
- Runaway queries or untraceable use. Every question and answer is logged for audit.

## Out of scope
- Writing data back. That means no edits and no actions such as sending emails or creating orders.

## Open points
- Dashboards, scheduled reports, forecasting and multiple data sources were not ruled out, but they were not requested either. Treat them as not in the first version unless decided otherwise.
- "Seeing data they shouldn't" means we need to know how roles map to data access. This is to be settled in the plan.
