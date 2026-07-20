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
