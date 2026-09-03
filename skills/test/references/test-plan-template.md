# Test Plan Template

Fill this in from an actual survey of the repo, then write it to `tests/TEST_PLAN.md`.
Delete sections that do not apply. Every claim must be true of *this* repo — a plan full of
generic advice is worse than no plan, because the next run will trust it.

---

# Test Plan

_Last updated: <today's actual date — run `date +%F`, do not leave the placeholder> ·
Maintained by the `/test` skill; edit freely, it is read first._

## 1. What this app is

One paragraph: what it does, who uses it, what breaking it would cost.

## 2. Stack

| Layer | Choice |
|---|---|
| Language / runtime | Python 3.x |
| Web framework | FastAPI / Django / Flask |
| Persistence | Postgres via SQLAlchemy 2.x / Django ORM |
| Frontend | Jinja templates / React SPA in `frontend/` |
| Background work | Celery / APScheduler / none |
| External services | Stripe, SendGrid, S3, … |
| Test runner | pytest x.y |
| Browser driver | pytest-playwright (Chromium) |

## 3. Layout

```
tests/
  TEST_PLAN.md
  conftest.py
  unit/
  integration/
  e2e/
  factories.py
```

Naming: files `test_<module>.py`, functions `test_<behavior>`, classes `Test<Subject>`.

## 4. Definitions for this repo

State these explicitly — they are the decisions everything else follows from.

- **Unit** = one function or class, no I/O of any kind. Here that means: …
- **Integration** = … (e.g. a real Postgres in a transaction, real app object, HTTP mocked)
- **E2E** = … (e.g. Playwright against `make dev` on port 8000 with a seeded database)

## 4b. Databases and environments (safety)

Record this explicitly; it is what stops someone running the suite against the wrong host.

| | Value |
|---|---|
| Test database URL | `postgresql://localhost:5432/app_test` (from `TEST_DATABASE_URL`) |
| How it is created | `make db-test` / `docker compose up -d testdb` |
| E2E database | separate: `app_e2e`, truncated between tests |
| Guard in place | `guard_database_url` in `tests/conftest.py` — aborts on a non-local host |

**Tests never run against production or staging.** The suite destroys data by design. Any
change that points tests at a shared or deployed database is a defect, not a configuration
choice.

## 5. Boundaries: what is real, what is faked

| Dependency | Unit | Integration | E2E |
|---|---|---|---|
| Database | fake/in-memory | real, rolled back per test | real, seeded |
| Outbound HTTP | stubbed | `respx`/`responses` | stubbed at the edge |
| Clock | `freezegun` | `freezegun` | real |
| Email / SMS | fake sink | fake sink | fake sink |
| Payments | fake | provider's test mode | provider's test mode |

Rule of thumb recorded here so it is not re-litigated per test: mock what you do not own and
cannot run; do not mock what you own.

## 6. Fixtures that already exist

List them so the next run reuses instead of duplicating.

| Fixture | Scope | Defined in | Gives you |
|---|---|---|---|
| `db_session` | function | `tests/integration/conftest.py` | transaction rolled back after each test |
| `client` | function | `tests/integration/conftest.py` | app test client bound to `db_session` |
| `admin_user` | function | `tests/conftest.py` | persisted admin account |

## 7. Markers

```ini
[pytest]
markers =
    slow: takes over a second
    e2e: drives a real browser
    external: needs network credentials; skipped in CI by default
```

## 8. Critical paths

The flows that must never silently break. These always get E2E coverage; everything else has
to earn it.

1. …
2. …

## 9. Deliberately not tested

With reasons, so nobody "fixes" the gap by accident.

- Third-party SDK internals — not ours.
- `…/admin/` scaffolding — generated, changes with the framework.
- Exact CSS/layout — brittle; covered by review, not tests.

## 10. Known flakiness

Areas that have burned us, and the mitigation in place.

## 11. Commands

```bash
make test              # everything
pytest tests/unit -q   # fast loop
pytest -m "not e2e" -q # what CI runs on every push
```

## 12. Coverage stance

What the number means here, what it is measured on, and what it is *not* used for.
E.g.: "≥85% on `app/`, but coverage is a smoke alarm, not a goal — no test exists solely to
raise it."
