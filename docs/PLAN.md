# AskDB v1: Implementation Plan (8 thin vertical slices)

## Context
Repo is pre-implementation: only `docs/SPEC.md`, `docs/one-pager.md`, and the CloudMetrics SQLite fixture (`data/cloudmetrics.db`, `.yaml`, `_eval.yaml`). SPEC targets SQL Server; the user chose **SQLite for slices 1-7 behind a DB-adapter interface, with the SQL Server adapter added in slice 8**. Each slice is end-to-end, independently testable, and sized for one short session. No code is written in this planning step.

Conventions across slices: Python 3.12 + FastAPI + uv + pytest backend in `backend/`; React + Vite + TS frontend in `frontend/`; config in `config/`. Update CLAUDE.md with real build/test commands as tooling appears (slice 1). Auth is a dev stub (header/env user + groups) until slice 8 swaps in Entra.

---

## Slice 1: Skeleton, DB adapter, fail-closed audit
- **Goal:** `POST /api/query` accepts a question, resolves a (stub) user to a principal, writes an audit row first, runs a hard-coded read-only `SELECT` via the adapter, returns rows. Proves the plumbing and the fail-closed rule.
- **Files:** `pyproject.toml`; `backend/app/main.py`, `config.py` (env/YAML limits: 30 s, 10k rows, 2 concurrent, 20/min); `db/adapter.py` (interface: `execute(principal, sql)`, `visible_schema(principal)`), `db/sqlite_adapter.py` (opens `mode=ro`); `auth/stub.py`; `principals.py` (group→principal map from YAML); `audit/store.py` (separate SQLite app DB, INSERT-only API, no update/delete methods); `tests/test_audit.py`, `tests/test_skeleton.py`; update CLAUDE.md commands.
- **Verify:** pytest: audit row exists before execution; forcing audit failure means the query is not executed; user with no group mapping gets "no access configured"; write statement against the `mode=ro` connection fails.
- **Depends on:** none.

## Slice 2: Glossary/config loader and prompt builder
- **Goal:** Load `data/cloudmetrics.yaml` into a typed model; render per-principal schema (CREATE TABLE DDL) and glossary into the prompt; define the structured-output schema (`status`, restatement, terms used, filters, SQL, refusal reason). No LLM call yet.
- **Files:** `backend/app/glossary/models.py`, `loader.py`; `prompt/builder.py`; `llm/schema.py` (Pydantic response model); `tests/test_prompt.py` (golden-text snapshots).
- **Verify:** prompt contains `description` verbatim and only the principal's visible tables/columns; hidden object names never appear; refine context includes previous question+SQL and never rows; invalid structured output is rejected by schema validation.
- **Depends on:** 1.

## Slice 3: SQL guard (standalone, heavily unit-tested)
- **Goal:** sqlglot-based guard: exactly one statement, SELECT only, reject INTO/EXEC/OPENROWSET/xp_/sp_/cross-DB/four-part names/WAITFOR/variable assignment/locking hints/objects outside the visible schema. Dialect-parameterized (SQLite now, T-SQL in slice 8).
- **Files:** `backend/app/guard/guard.py`, `rules.py`; `tests/test_guard.py` (table-driven allow/deny corpus incl. bypass attempts: comments, stacked statements, case tricks, CTE-wrapped writes, PRAGMA/ATTACH for SQLite).
- **Verify:** pytest corpus passes; every `expected_sql` in `data/cloudmetrics_eval.yaml` passes the guard; guard failure returns a generic refusal and is audited as `guard-blocked`.
- **Depends on:** 2 (visible-schema model).

## Slice 4: LLM pipeline end to end (answer / clarify / refuse) with a table result
- **Goal:** Real Anthropic call (model ID from config) → validated structured output → guard → execute (30 s timeout, cap 10,000+1, truncation flag) → audit (model id, prompt hash, SQL, guard result, duration, row count, tokens) → JSON response. LLM timeout/retry/outage message. Minimal React page: question box + plain result table.
- **Files:** `backend/app/llm/client.py` (injectable so tests use a fake), `pipeline.py`, `limits.py` (concurrency + rate limit), `routes/query.py`; `frontend/src/App.tsx`, `QuestionBox.tsx`, `ResultTable.tsx`, `api.ts`; `tests/test_pipeline.py`.
- **Verify:** unit tests with a fake LLM cover answer/clarify/refuse/invalid-output/guard-blocked/timeout/truncation/empty-result; manual run: ask "How many active customers?" in the browser and see the table plus an audit row; confirm 3rd concurrent and 21st/min requests are rejected.
- **Depends on:** 1, 2, 3.

## Slice 5: Eval harness and golden-set CI gate
- **Goal:** Runner that executes `data/cloudmetrics_eval.yaml`: result-set equality vs `expected_sql`, plus correct-or-refuse score and confident-wrong tracking by category. Add golden cases for access, write-phrasing, prompt injection, and unanswerable questions. Wire into CI (live-LLM job gated/nightly; fake-LLM job on every PR).
- **Files:** `tests/validation/runner.py`, `test_golden.py`; extend `data/cloudmetrics_eval.yaml` (new categories); `.github/workflows/ci.yml`; CLAUDE.md test commands.
- **Verify:** `uv run pytest tests/validation` produces a per-category report; fake-LLM run is deterministic; live run meets ≥95% correct-or-refuse and **0 confident-wrong in access/write**; a test asserts the principal connection cannot write.
- **Depends on:** 4. (Run early so later slices are regression-gated.)

## Slice 6: Interpretation panel, clarify/refuse UI, refine
- **Goal:** Always-visible interpretation panel (restatement, glossary terms with hover definitions, filters/date range, active role set, expandable read-only SQL); clarify with selectable options; refuse messaging that doesn't name hidden objects; "Refine this" sending previous question+SQL and linking audit rows via parent id; empty-result vs error states with request id; cancel button (cancels DB query); example questions from the glossary.
- **Files:** `backend/app/routes/query.py` (parent_id, cancel endpoint), `pipeline.py` (cancel token → adapter interrupt); `frontend/src/InterpretationPanel.tsx`, `ClarifyOptions.tsx`, `RefineBox.tsx`, `StatusViews.tsx`; `tests/test_refine_cancel.py`; Vitest component tests.
- **Verify:** pytest: refine audit row has `parent_id`, prompt contains no rows; cancel stops a slow query; UI tests for the 5 states (answer, clarify, refuse, empty, error); manual browser walkthrough.
- **Depends on:** 4 (5 for regression gate).

## Slice 7: Deterministic charts, templated summary, export
- **Goal:** Post-processing: stats (total, top/bottom, period-over-period with partial-period label, min/max, count; NULL shown distinctly); chart-type rules by result shape with user switch among valid types only, top-N + "other", data-table alternative, color-blind-safe palette; summary templates with exact figures, marked partial when truncated; CSV/XLSX export of exactly the displayed capped result with truncation note and formula-injection escaping; export audited.
- **Files:** `backend/app/postprocess/stats.py`, `chart_rules.py`, `summary.py`; `export/csv_xlsx.py`, `routes/export.py`; `frontend/src/Chart.tsx`, `Summary.tsx`, `ExportMenu.tsx`; `tests/test_chart_rules.py`, `test_summary.py`, `test_export.py`.
- **Verify:** unit tests: shape→chart-type table, summary strings against fixed result sets (including zero/NULL/division-by-zero), cells starting `= + - @` are escaped, truncation noted in file, export audit row written; no LLM import in `postprocess/` (assertion test); manual chart check in browser.
- **Depends on:** 4 (6 for UI shell).

## Slice 8: SQL Server adapter, Entra OIDC, impersonation/RLS
- **Goal:** Swap the stub for real access control per SPEC §4/§11: Entra OIDC login (state/nonce, short sessions, group claim + overage via Graph), group→principal resolution per request, SQL Server adapter with `EXECUTE AS USER`/`REVERT`, connection reset on pool return, per-principal visible-schema via `HAS_PERMS_BY_NAME`, T-SQL guard dialect, driver timeout + server cancel. SQL Server fixture (Docker) with the CloudMetrics data ported, two principals, one RLS predicate.
- **Files:** `backend/app/auth/entra.py`; `db/mssql_adapter.py`; `guard/` T-SQL rules; `fixtures/mssql/` (DDL, seed, principals/RLS scripts, docker-compose); `tests/integration/test_impersonation.py`, `test_rls.py`; deployment/config docs.
- **Verify:** integration tests with ≥2 principals: out-of-role table is invisible in schema and the query is rejected by the DB; RLS-limited counts; a connection is never reused un-reverted; principal lacks write permission; golden set re-run on the MSSQL adapter keeps confident-wrong = 0 for access/write.
- **Depends on:** 1, 3, 5 (all earlier slices run unchanged on the adapter interface).

---

## Deferred (not in the 8; follow-on after slice 8)
Admin UI with versioned glossary/rollback and metric validation (SPEC §9, milestone 5), auditor log search UI, nightly schema-drift job, load testing. They fit the same pattern but exceed "8 short slices"; flag if you'd rather swap one in (e.g. merge 3 into 4 to make room for admin UI).

## Cross-slice risks
- Engine divergence: keep `Dialect`/`Adapter` as the only engine-specific seams so slice 8 doesn't touch the pipeline.
- Eval data ends Nov 2025: "today" must come from config, not the clock, in tests.
- Metric semantics live only in `data/cloudmetrics.yaml`; change eval expectations together with it.
