# AskDB — Specification (v1)

Source: docs/one-pager.md plus interview decisions.

## 1. Purpose
Web app where business users ask plain-English questions of a SQL Server database and get a table, chart, templated summary, and export. Strictly read-only, role-enforced, audited.

## 2. Decisions
| Area | Decision |
|---|---|
| Database | Microsoft SQL Server, single instance, modest known schema (tens of tables) |
| Stack | Python FastAPI backend, React frontend; sqlglot (T-SQL dialect) for parsing; Anthropic SDK |
| Auth | Microsoft Entra ID via OIDC; app stores no passwords |
| Scale | Internal, tens of users, single instance, on-prem or one cloud tenant |
| Access enforcement | DB-native. Entra groups map to DB roles; union of permissions |
| LLM data flow | Only schema (per-role), glossary, question, and prior question+SQL go to the LLM. Result rows never do |
| Ambiguity | Admin-curated glossary; refuse when no matching term or column |
| Summaries | Deterministic templates over computed stats, no LLM |
| Charts | Deterministic rules from result shape, user can switch among valid types |
| Conversation | Stateless questions plus "refine this" on the last result |
| Audit | Append-only table in a separate app DB |
| Quality gate | Golden question set in CI |

Out of scope for v1: writes of any kind, dashboards, saved questions, scheduled reports, forecasting, multiple data sources/engines, multi-turn free conversation, multi-tenancy, uncapped export.

## 3. Users and roles
- **Business user**: asks questions within their data access.
- **Admin**: manages glossary, column annotations, group→role mapping.
- **Auditor**: reads the audit log. Distinct Entra group and permission from Admin; the same person may hold both and the app does not enforce separation of duties. Admin actions (glossary, metric, annotation and group→principal mapping edits, rollbacks) are written to the same audit table (user, timestamp, entity, version, diff) so Auditors can review them.

## 4. Access control (the central safety design)
1. Entra groups → a **role set**. Admin pre-creates one SQL Server database user (no login) per *role combination* in use, each a member of the relevant DB roles with GRANTs (tables, columns) and Row-Level Security predicates where rows are restricted.
2. For each request the app connects with a single low-privilege service login that has only `IMPERSONATE` on those principals, runs `EXECUTE AS USER = '<principal>'`, executes, then `REVERT` (connection reset on return to pool; never reuse an impersonated connection without reset).
3. A user whose groups map to no principal gets "no access configured" and cannot query.
4. All principals are read-only: no INSERT/UPDATE/DELETE/DDL/EXEC permissions. The app cannot widen access beyond the principal.
5. **Per-role schema context**: the schema/glossary given to the LLM is filtered to what the principal can see (derived from DB permissions, e.g. `sys.fn_my_permissions` / `HAS_PERMS_BY_NAME`), so the model cannot reveal or hint at hidden tables/columns. Glossary terms referencing inaccessible objects are omitted for that user.
6. Schema/context is cached per principal and invalidated on admin change or permission-change refresh.
7. RLS limitation to note: model cannot see RLS-filtered rows; counts reflect permitted rows only. Interpretation text must not claim "all customers".

## 5. Query pipeline
1. Receive question → authenticate → resolve principal.
2. Build prompt: per-role schema, glossary (approved version), question, optional refine context (previous question and SQL, never rows).
3. LLM returns structured output: `status` (answer | clarify | refuse), restatement, glossary terms used, filters/date range, SQL, and a refusal reason when applicable.
4. **Refusal rule**: if a required concept has no matching glossary term or column, return `refuse` with what is missing. Date phrases ("last quarter") resolve via a fixed fiscal/calendar definition in glossary config; if undefined, refuse/ask.
5. **SQL guard** (sqlglot T-SQL): exactly one statement; SELECT only; reject `INTO`, `EXEC`, `OPENROWSET`/`OPENQUERY`/`OPENDATASOURCE`, `xp_`/`sp_` calls, table hints that change locking, cross-database and four-part names, `WAITFOR`, variable assignment, and any object not in the role's visible schema. Guard failure → generic refusal, logged, never executed. Treat guard as defense in depth; the DB principal is the real control.
6. Execute with `SET LOCK_TIMEOUT`, **30 s query timeout** (driver-level and cancel on server), read-only principal. Fetch at most **10,000 rows + 1** to detect truncation.
7. Post-process: compute stats, choose chart, build summary, build interpretation panel.
8. Write audit record. Return response.

## 6. Prompt-injection and data-content handling
- Result rows are never fed to the LLM, which removes the data-as-instructions path. Glossary and column annotations are admin-authored and treated as trusted; user question is untrusted.
- Output is structured and validated against a schema; free-text fields rendered as plain text (no HTML/markdown execution).

## 7. Response UI
- **Question box** with example questions drawn from the glossary.
- **Interpretation panel** (always visible): plain-English restatement, glossary terms used (with definitions on hover), filters and date range, and the active role set. Expandable "Show SQL" (read-only).
- **Result table**: sortable, paginated client-side, row count, **truncation banner** if >10,000 rows ("showing first 10,000; totals below may be incomplete") — summary stats are marked partial when truncated.
- **Chart**: default by rule — time column + measure → line; category + measure → bar (top N with "other" noted); two measures → scatter only when requested via switch; single value → KPI tile; otherwise table only. User may switch among valid types only.
- **Summary**: templated sentences from computed stats (total, top/bottom, period-over-period % change, min/max, count) with exact figures; never speculative causes.
- **Clarify/refuse states**: clarifying question with selectable options (when 2+ plausible glossary interpretations); refusal states what is missing and, if relevant, "you may not have access to that data" without naming hidden objects.
- **Refine this**: free-text refinement on the last result; sends previous question + SQL; each refine is a new audited question linked via parent id.
- **Empty result** distinguished from error; zero rows shown as "no matching rows", with filters repeated.
- Loading with cancel button (cancels the DB query). Errors never show raw SQL errors; show a request id.

## 8. Export
- CSV and XLSX of **exactly the displayed (capped) result**; truncation noted in the file and filename/metadata.
- Cell values beginning with `=`, `+`, `-`, `@` are escaped against formula injection.
- Each export is audited (user, question id, format, row count). No uncapped re-query in v1.

## 9. Glossary and admin UI
- Entities: **terms** (name, synonyms, description), **metrics** (name, SQL expression, grain, filters, owning tables), **date conventions** (fiscal year start, week start), **column annotations** (description, units, sensitivity note), **group→principal mappings**.
- All edits versioned (who, when, diff), with rollback. Metric SQL is validated on save against the DB (parse + dry run `SET FMTONLY`/`sp_describe_first_result_set`) and run under each affected principal to confirm visibility.
- Glossary version is recorded on every audit row.
- Admin UI authorization requires the Admin role (from Entra group), separate from query access.

## 10. Audit log
Append-only table in separate app DB; app account has INSERT only (no UPDATE/DELETE). Fields: id, parent id, timestamp, user (Entra oid + UPN), groups, principal used, question, status (answer/clarify/refuse/error/guard-blocked), glossary version, model id/version, prompt hash, restatement, generated SQL, guard result, duration, row count, truncated flag, result hash, export events, error code. **No result rows stored.** Auditor/Admin can search/filter/view all; users cannot view others. Retention: 1 year. Expired rows are purged by a separate privileged scheduled DBA job outside the app; the app account stays INSERT-only. Retention period is configured in that job, not in the app.
Failure policy: if the audit write fails, the query is not run (fail closed).

## 11. Non-functional
- Limits (final v1 launch values): 30 s timeout, 10,000 row cap, per-user concurrency of 2 queries and 20 questions/min, LLM timeout and retry with backoff, graceful LLM-outage message. All are set via env/config file (restart to change), not the admin UI; revisit after the pilot as a normal change.
- Connection pooling per principal with reset on checkout; secrets in env/secret store, never logged.
- Unicode and locale: dates ISO in storage, displayed in user locale; currency/units from annotations.
- Accessibility: keyboard navigable, chart has data-table alternative, color-blind-safe palette.
- Security: HTTPS, CSRF/OIDC state/nonce, short sessions, token validation of groups claim (note Entra group-overage claim >200 groups → resolve via Graph).
- Session expiry or group change mid-session takes effect on next request (principal resolved per request).

## 12. Edge cases to handle explicitly
- Group removed in Entra after login; group overage claim; user in groups with no combined principal (deny, tell admin).
- Ambiguous or colliding synonyms in glossary (admin warning on save).
- Query returns >10k rows, wide tables (column cap, e.g. 100), huge cell values (truncate display).
- Division by zero / NULL in metrics; NULL shown distinctly from zero and blank.
- Period comparisons with incomplete current period (label as partial).
- Time zones for "today"/"this month": single business time zone, **UTC** (no DST handling needed). Fiscal year = calendar year (starts 1 January; quarters Jan–Mar, Apr–Jun, Jul–Sep, Oct–Dec). Week starts **Monday**. These are the initial date-convention values in glossary config (§9) and remain admin-editable there.
- Duplicate-row inflation from joins (metric definitions own the join paths; guard cannot verify this → golden set covers it).
- Schema drift: column dropped after glossary written → nightly validation job flags broken metrics, affected questions refuse rather than guess.
- Cancel mid-query; client disconnect; timeout shows "too slow, try narrowing" suggestion.
- Cost: LLM token spend per user tracked in audit.

## 13. Quality gate
- **Golden set** (fixture SQL Server DB with synthetic data): question → expected result set / expected refusal. Categories: totals/KPIs, trends, lookups, comparisons/rankings, ambiguous terms, unanswerable, out-of-role access (must refuse or return only permitted rows), prompt injection in question, write-attempt phrasing ("delete all orders"), SQL-guard bypass attempts.
- Release gates (initial targets, to be tuned): correct-or-refuse ≥ 95%; **confident-wrong = 0 on the golden set for access and write categories**; zero writes possible (also verified by test that the principal lacks write permission).
- Unit tests for the SQL guard and export escaping; integration tests for EXECUTE AS/RLS with at least two principals.

## 14. Milestones
1. Skeleton: Entra login, principal resolution, fixture DB, execute-as plumbing, audit table.
2. Pipeline: prompt, structured output, guard, execution, table result.
3. Interpretation panel, glossary (files first), refusal/clarify.
4. Charts, templated summary, export.
5. Admin UI with versioning and validation.
6. Golden set, hardening, load/limits.

## 15. Open items for owner
None.
