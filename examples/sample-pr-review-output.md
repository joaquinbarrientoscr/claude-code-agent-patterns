# Example output — `code-review-architect` in DEEP mode

This is an illustrative example of what the agent produces when reviewing a small feature PR. The hypothetical PR adds a new endpoint `POST /api/v1/reports/export` that generates a CSV export of user report data and emails it to the requesting user.

---

# Review: Add `/api/v1/reports/export` endpoint

**Mode:** DEEP
**Verdict:** REQUEST CHANGES
**Scope:** 4 files, ~180 LOC (api/routes/reports.py, services/export.py, services/mailer.py, tests/test_reports.py)

## Summary
Endpoint generates a CSV of the requesting user's reports and emails it. The CSV generation and mailer wiring are clean and reuse existing helpers. Two blocking issues: (1) the SQL query doesn't scope results to the authenticated user, exposing other users' data via IDOR; (2) the generated file path is constructed from a user-controlled `filename` parameter without sanitization, enabling path traversal. One high-severity issue: the mailer is invoked synchronously on the request thread, which will time out for users with >10k reports.

## Findings

### 🔴 Blocking (must fix before merge)

- **services/export.py:42** — Missing user scope on report query (IDOR)
  The query `SELECT * FROM reports WHERE id IN (...)` uses report IDs from the request body without verifying those reports belong to the authenticated user. An attacker can enumerate `report_id` values to export any user's data.
  **Fix:** add `AND user_id = :current_user_id` to the WHERE clause, or fetch reports through the existing `ReportRepository.list_for_user(user_id)` helper which already enforces scope.

- **services/export.py:61** — Path traversal via `filename` parameter
  `output_path = f"/tmp/exports/{filename}.csv"` where `filename` comes from the request body. An attacker sends `filename="../../../etc/passwd"` to overwrite arbitrary files the service user has write access to.
  **Fix:** derive the filename server-side (e.g., `f"export_{user_id}_{uuid4()}.csv"`) and ignore the client-supplied value. If the client needs to suggest a display name, store it as metadata only, not in the path.

### 🟠 High (strongly recommend fixing)

- **api/routes/reports.py:78** — Synchronous email send on request thread
  `mailer.send(...)` is called inline, blocking the response until SMTP completes. For users with ~10k reports the CSV generation + send takes 30-60s; the request will time out at the load balancer (30s limit). The user will see a 504 even though the export eventually succeeds, leading to duplicate requests.
  **Fix:** enqueue the export as a background job (the codebase already uses `rq` for this pattern — see `services/report_scheduler.py:23`). Return `202 Accepted` with a job ID the client can poll.

### 🟡 Medium (consider)

- **services/export.py:54** — No size cap on CSV generation
  A user with pathological data could trigger generation of a multi-GB CSV, consuming disk on the app server. Cap at a reasonable row count (50k?) and return an error asking the user to narrow the date range if exceeded.

- **tests/test_reports.py:18** — Test asserts on response shape but not on the email actually being sent
  The mailer is mocked and the test checks the endpoint returns 200, but doesn't assert `mailer.send` was called with the expected recipient. A regression that silently drops the email would pass this test.

### 🟢 Low / nits

(none — skipped)

## What the diff does well

- CSV generation correctly uses the `csv.writer` module with `quoting=csv.QUOTE_MINIMAL` rather than manual string concatenation. Good defense against injection into downstream spreadsheets.
- The new `ExportJob` dataclass is scoped to `services/export.py` and not exposed through the public API, which keeps the seam clean for future changes to the export format.

## Out of scope but noticed

- The `reports` table lacks an index on `user_id`. This PR's query will table-scan for users with significant report history. File a separate ticket — adding the index isn't part of this change's scope but will become a production issue soon.
