# Fix Plan — Issue #154

**Issue:** [Health check DB probe passes a raw SQL string, which fails under SQLAlchemy 2.x](https://github.com/ascherj/pathreview/issues/154)
**Branch:** `fix/154-health-check-sql-text`
**Target PR:** end of Week 9

---

## Problem Summary

The `/health` endpoint (in `api/routes/health.py`) probes PostgreSQL with a raw string:

```python
await db.execute("SELECT 1")
```

SQLAlchemy 2.x no longer accepts a bare Python string in `AsyncSession.execute()` — it requires a `TextClause` produced by `sqlalchemy.text(...)`. As written, the call raises `ArgumentError: Textual SQL expression 'SELECT 1' should be explicitly declared as text('SELECT 1')`, which is caught by the surrounding `try/except`. The exception is logged and the endpoint reports the postgres dependency as `"unhealthy"` — even when Postgres is actually up and responding.

I reproduced this locally against my running Docker stack: `curl http://localhost:8000/health` returned a 503 with `"dependencies": {"postgres": "unhealthy", ...}` while `docker compose ps` confirmed the `db` container was `(healthy)` and accepting connections. The failure is entirely in the query construction, not in the database itself.

## Intended Fix

- Import `sqlalchemy.text` at the top of `api/routes/health.py`.
- Change `await db.execute("SELECT 1")` to `await db.execute(text("SELECT 1"))`.
- Add a test that starts the app with a working database and asserts `/health` returns 200 with `dependencies.postgres == "healthy"` — currently the probe silently reports unhealthy on every request, so nothing in the existing test suite catches this regression.

## Out of Scope

The endpoint has a **separate** bug (`settings.redis_host` doesn't exist on the `Settings` class — this is issue #155) that also causes redis to always report unhealthy. That's tracked in a different issue and will not be addressed in this PR.
