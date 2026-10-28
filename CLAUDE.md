# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## State of the repo

Pre-implementation. There is no source code, build system, test runner, or git history yet; only:

- `docs/one-pager.md`: product brief (AskDB: plain-English questions over a database for business users; strictly read-only).
- `docs/SPEC.md`: v1 specification (the authoritative design; milestones in §14).
- `data/cloudmetrics.db`: SQLite sample database (CloudMetrics SaaS: customers, plans, subscriptions, invoices, support_tickets; Mar 2022 to Nov 2025).
- `data/cloudmetrics.yaml`: per-database config. The model sees exactly `description` (verbatim, includes metric definitions and SQLite date idioms) plus `tables` rendered as CREATE TABLE DDL.
- `data/cloudmetrics_eval.yaml`: validation cases (`question`, `category`, `expected_sql`). A case passes when generated SQL returns the same result set as `expected_sql`. Dates are explicit because the data ends Nov 2025.

No build, lint, or test commands exist yet. Don't invent them; add them here once tooling is created.

## Spec vs. sample data mismatch (resolve before building)

`SPEC.md` targets **Microsoft SQL Server** (T-SQL, sqlglot T-SQL dialect, `EXECUTE AS USER`, RLS, Entra ID OIDC, FastAPI + React). The `data/` files target **SQLite** (`mode=ro`, `strftime`/`julianday`, `${CLOUDMETRICS_DB_PATH}`) and reference files not in this repo (`config/databases/sample.yaml`, `config/validation/sample.yaml`, `uv run pytest tests/validation`, "SPEC 11.2"). Treat the data files as a fixture/dev database and ask the user which engine is the real target before writing engine-specific code.

## Architecture (from SPEC.md; spans many components)

Pipeline per request: authenticate, resolve DB principal from Entra groups, build prompt, LLM returns structured output, SQL guard, execute, post-process, audit, respond.

Key invariants to preserve:

- **DB-native access control is the real security boundary.** Each Entra group combination maps to a pre-created read-only DB principal; the app impersonates it per request and resets the connection. The SQL guard (single SELECT only, blocks INTO/EXEC/OPENROWSET/xp_/cross-DB etc.) is defense in depth only.
- **Result rows never go to the LLM.** Only per-role schema, glossary, the question, and previous question+SQL (for "refine") are sent. The schema/glossary shown to the LLM is filtered to what the principal can see.
- **Refuse rather than guess**: no matching glossary term/column means `refuse`; ambiguity means `clarify`.
- **Summaries and charts are deterministic** (templates over computed stats; chart type chosen by result shape). No LLM involvement.
- **Audit fails closed**: audit write goes first to an append-only table in a separate app DB (INSERT-only account); if it fails, the query doesn't run. No result rows stored.
- Limits: 30 s timeout, 10,000-row cap (fetch +1 to detect truncation, mark stats partial), 2 concurrent queries and 20 questions/min per user; configured via env/config, not the admin UI.
- Exports (CSV/XLSX) contain exactly the displayed capped result, with formula-injection escaping.
- Quality gate: golden question set in CI; confident-wrong must be 0 for access and write categories.

## Business-metric definitions

Metric semantics (MRR vs. revenue, subscription-level churn, ARPU per subscription, etc.) live in `data/cloudmetrics.yaml` `description`. Change them there, and keep `cloudmetrics_eval.yaml` expectations consistent with any change.
