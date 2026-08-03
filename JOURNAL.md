# Module 3 Journal — PathReview Contribution

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/154

**Issue title:** Health check DB probe passes a raw SQL string, which fails under SQLAlchemy 2.x

**Tier:** [x] Tier 1 [ ] Tier 2 [ ] Tier 3

**Problem summary:**
The `/health` endpoint in `api/routes/health.py` probes PostgreSQL by
calling `await db.execute("SELECT 1")` — a bare Python string. SQLAlchemy
2.x rejects raw strings in `AsyncSession.execute()` and requires a
`TextClause` produced by `sqlalchemy.text(...)`. As a result, the postgres
probe raises inside the surrounding `try/except`, the endpoint always
reports postgres as unhealthy, and clients get a 503 even when the
database is fully up and reachable. A successful fix will make `/health`
correctly reflect the actual database state — returning 200 with
`dependencies.postgres == "healthy"` when Postgres is running, and 503
only when it truly isn't.

**Branch name:** `fix/154-health-check-sql-text`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

---

### Selection notes — "Is this right for me?" checklist

**Part 1 — Understanding the issue**

- [x] I can explain the problem and the expected behavior in 2–3 sentences
      without reading the issue.
- [x] I've located the relevant files: `api/routes/health.py` (endpoint),
      `core/database.py` (the `get_db` dependency being injected).
- [x] Concrete before/after: **before** — `curl http://localhost:8000/health`
      returns 503 with `postgres: "unhealthy"` even though `docker compose ps`
      shows the db container as `(healthy)` and accepting connections.
      **after** — the same request returns 200 with `postgres: "healthy"`.
      I confirmed the "before" state locally tonight before claiming the issue.

**Part 2 — Tier fit**

- [x] Tier 1 is right for me. This is my first contribution to an open
      source codebase of this scale, and #154 is well-scoped: the fix lives in
      one file, the change is a couple of lines, and the test surface is
      small. No incentive to reach higher — a clean Tier 1 PR beats a
      half-finished Tier 2 or 3.

**Part 3 — Codebase readiness**

- [x] I've read the full body of `api/routes/health.py`, not just grepped it.
      The bug is on the line `await db.execute("SELECT 1")` inside the try
      block around the postgres check.
- [x] I understand enough surrounding context to change it safely: the
      function is a straightforward async health probe with three independent
      `try/except` blocks (postgres, redis, vector-db) and a single return
      path. The fix is scoped to the postgres block. Rough plan: import
      `sqlalchemy.text`, wrap the SQL string, and add a test.
- [x] I've found the tests folder (`tests/`) and looked for existing
      health-check coverage. I'll follow the patterns already established in
      the API test files when I write my new test in Week 8.

**Part 4 — Scope and time**

- [x] There are several other students on this issue per the ledger, which
      is fine — claims are non-exclusive and I'm graded on my own PR, not on
      being first.
- [x] Time estimate: ~4–5 hours over Weeks 8 and 9, well inside the Tier 1
      budget of 3–6 hours. The fix itself is short; most of the time will go
      to writing a reliable async test and running the pre-submission checks
      (`make check`, `make test-unit`).
- [x] No open blockers or dependencies mentioned on the issue thread.

**Verdict:** All boxes checked — claim submitted, branch created, ready
for Week 8.

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/carloace16/pathreview/commit/cbc7894f73da9427c1b83da18a645d330799a23d

**Reproduction summary:**
Started the app locally (`docker compose up -d` → `make run`) and hit
`curl http://localhost:8000/health`. The endpoint returned 503 with
`"postgres": "unhealthy"`, and the backend log recorded the exact
SQLAlchemy 2.x error the issue predicts:
`postgres_health_check_failed error="Textual SQL expression 'SELECT 1' should be explicitly declared as text('SELECT 1')"`.
Docker confirmed the `db` container was `(healthy)` at the same time,
so the failure is entirely in query construction, not the database.

**PLAN.md link:** https://github.com/carloace16/pathreview/blob/fix/154-health-check-sql-text/PLAN.md

**Walkthrough video (recommended):** _(not recorded — will be shared in Slack if time permits during Week 9)_

**Blockers or open questions:**
None going into Week 9. The `settings.redis_host` error visible in the
same log is a separate, known bug (issue #155) that's out of scope for
this PR. Also worth flagging for my own reference: I had to renamed the
project folder from `Week 7` to `Week 7-10` between weeks, which broke
the venv (hardcoded paths). Fixed by deleting `.venv/` and re-running
`make setup`. Documenting here so I don't lose an hour to it again.

### Reproduction steps (for the record)

```bash
# 1. Start services
docker compose up -d

# 2. Start app
make run   # in a separate terminal, keep running

# 3. Hit the endpoint
curl http://localhost:8000/health
# → 503, dependencies.postgres == "unhealthy"

# 4. Check backend log for the caught exception
# → postgres_health_check_failed
#      error="Textual SQL expression 'SELECT 1' should be
#             explicitly declared as text('SELECT 1')"
```

---

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the fix (`fix: wrap health check SQL probe in text() for
SQLAlchemy 2.x compatibility`) — a 2-line change in `api/routes/health.py`
that imports `sqlalchemy.text` and wraps the `"SELECT 1"` string. Verified
manually with `curl http://localhost:8000/health`: the postgres dependency
now correctly reports `"healthy"` when the DB is up. Wrote and passed a
regression test at `tests/unit/test_health.py`. Ran the full test suite
and confirmed my changes introduce zero new failures — the 53 pre-existing
failures on `main` are all unrelated to my issue (they cover other Tier 1
issues like #146, #148, #149, #150, #153, #157 that other students are
claiming).

**Next steps:**
Open the PR against `ascherj/pathreview:main` with a full description
covering what was fixed, how to verify, and a note about the pre-existing
mypy and lint failures in the codebase that are outside the scope of this
issue. Then submit the branch URL.

**Blockers:**
Pre-commit hooks (mypy in particular) fail against 44 pre-existing type
errors across 7 files, none of which I modified. Bypassed with
`--no-verify` for my two commits, and I'll document this explicitly in
the PR description so the maintainer knows the failure isn't caused by
my change.

---

### Check-in 2 (end of week)

**PR link:** _(to be filled in after opening the PR)_

**Branch:** `fix/154-health-check-sql-text`

**What you built:**
Fixed the `/health` endpoint's PostgreSQL probe in `api/routes/health.py`
by wrapping `"SELECT 1"` in `sqlalchemy.text()`. SQLAlchemy 2.x rejects
bare strings in `AsyncSession.execute()` and requires a `TextClause`; the
old code raised `ArgumentError` on every request and the surrounding
`try/except` silently reported postgres as `"unhealthy"` regardless of
actual database state. With the fix, the endpoint honestly reflects
Postgres health.

**Tests added or updated:**
Added `tests/unit/test_health.py` with a single `TestClient`-based
regression test (`test_health_check_postgres_reports_healthy`). It hits
`GET /health` and asserts `dependencies.postgres == "healthy"` when the
database is reachable, tolerating either a 200 or 503 top-level status
because Issue #155 (out of scope) currently forces the endpoint to 503.

**Self-review confirmation:**

- [x] `make test-unit` passes with my new test — 53 pre-existing failures
      on main, 53 after my changes (same set, none introduced by this PR),
      1 new passing test.
- [ ] `make check` — fails on 182 pre-existing lint errors and 44 mypy
      errors in unrelated files. Ran ruff and black on just my two files
      and confirmed both are clean apart from one pre-existing `B008`
      warning inside `api/routes/health.py:15` (the `Depends()`-in-default
      pattern used across all FastAPI routes in the project).

**Draft PR feedback received from:** none — went straight to a
ready-for-review PR given the small scope of the change.
