# conftest.py Guide

A `conftest.py` is shared setup, not a junk drawer. The suite's readability is mostly decided
here: a good conftest makes each test a three-line statement of behavior, a bad one makes
every test a scavenger hunt.

## Layering

Put each fixture at the narrowest level that needs it.

```
tests/
  conftest.py              # cross-cutting: settings, tmp dirs, frozen clock, network block
  factories.py             # data builders (plain functions, not fixtures)
  unit/conftest.py         # in-memory fakes only; zero I/O
  integration/conftest.py  # database engine, per-test transaction, app client
  e2e/conftest.py          # live server, browser context, seeded accounts
```

A fixture belongs in the root `conftest.py` only if two or more tiers need it. Anything else
moves down. When in doubt, define it in the one test file that uses it.

## Scope

| Scope | Use for | Cost of getting it wrong |
|---|---|---|
| `session` | engine creation, browser launch, building an app instance | state leaks between tests |
| `module` | rarely — one costly read-only fixture shared by a single file | subtle ordering coupling |
| `function` (default) | anything a test mutates: rows, users, temp files | slow if truly expensive |

The pattern that gives you both speed and isolation: expensive resource at `session` scope,
plus a `function`-scoped transaction wrapped around each test that always rolls back.

## Root conftest

```python
# tests/conftest.py
import os
import pytest

os.environ.setdefault("APP_ENV", "test")


@pytest.fixture(scope="session")
def settings():
    from app.config import Settings
    return Settings(env="test", database_url=os.environ["TEST_DATABASE_URL"])


@pytest.fixture
def frozen_time():
    """Pin the clock so expiry logic is deterministic."""
    from freezegun import freeze_time
    with freeze_time("2026-01-15T12:00:00Z") as clock:
        yield clock


@pytest.fixture
def tmp_upload_dir(tmp_path):
    d = tmp_path / "uploads"
    d.mkdir()
    return d
```

## Integration conftest — the important one

```python
# tests/integration/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.db import Base, get_db
from app.main import create_app


@pytest.fixture(scope="session")
def engine(settings):
    engine = create_engine(settings.database_url)
    Base.metadata.create_all(engine)          # or run migrations here
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture
def db_session(engine):
    """A session inside a transaction that is always rolled back.

    Every test starts from the same clean database, and nothing a test writes can
    reach the next one — no truncation, no ordering dependency.
    """
    connection = engine.connect()
    transaction = connection.begin()
    session = sessionmaker(bind=connection)()
    try:
        yield session
    finally:
        session.close()
        transaction.rollback()
        connection.close()


@pytest.fixture
def app(db_session):
    app = create_app()
    app.dependency_overrides[get_db] = lambda: db_session
    yield app
    app.dependency_overrides.clear()


@pytest.fixture
def client(app):
    with TestClient(app) as c:
        yield c


@pytest.fixture
def authed_client(client, admin_user):
    client.headers["Authorization"] = f"Bearer {issue_token(admin_user)}"
    return client
```

Django: use `pytest-django`'s `db` / `transactional_db` fixtures rather than hand-rolling the
above, and its `client` / `admin_client` for requests.

## Unit conftest

Fakes and builders only. If it opens a socket, a file, or a database, it does not belong here.

```python
# tests/unit/conftest.py
import pytest


class FakeEmailer:
    def __init__(self):
        self.sent = []

    def send(self, to, subject, body):
        self.sent.append((to, subject, body))


@pytest.fixture
def emailer():
    return FakeEmailer()
```

Asserting on `emailer.sent` tests an outcome; asserting on `mock.send.assert_called_once()`
tests a call. Prefer the fake.

## Factories beat fixture sprawl

Ten fixtures differing by one field is a smell. Write one builder with defaults:

```python
# tests/factories.py
from app.models import User


def make_user(**overrides):
    return User(**{
        "email": "user@example.com",
        "is_active": True,
        "role": "member",
        **overrides,
    })
```

Each test then says exactly what matters to it:

```python
user = make_user(role="admin", is_active=False)
```

The overridden fields are the ones under test; everything else is visibly irrelevant. Reach
for `factory_boy` once builders need relationships and sequences.

## Guard the database (add this first)

Tests destroy data. Make it structurally impossible to aim them at production — an autouse,
session-scoped check that runs before any test touches anything.

```python
# tests/conftest.py
import os
import re

import pytest

_ALLOWED_HOSTS = {"localhost", "127.0.0.1", "::1", "db", "postgres", "testdb"}
_FORBIDDEN = re.compile(r"prod|production|live|staging", re.IGNORECASE)


@pytest.fixture(scope="session", autouse=True)
def guard_database_url():
    """Abort the whole run rather than let tests touch a real database."""
    from urllib.parse import urlparse

    url = os.environ.get("TEST_DATABASE_URL") or os.environ.get("DATABASE_URL")
    if not url:
        pytest.exit("No database URL set; refusing to guess.", returncode=3)

    parsed = urlparse(url)
    host, name = (parsed.hostname or ""), (parsed.path or "").lstrip("/")

    if host not in _ALLOWED_HOSTS and not os.environ.get("ALLOW_REMOTE_TEST_DB"):
        pytest.exit(f"Refusing to run against non-local database host {host!r}.", returncode=3)
    if _FORBIDDEN.search(name) or _FORBIDDEN.search(host):
        pytest.exit(f"Refusing to run against {host}/{name} — looks like a real environment.",
                    returncode=3)
    if os.environ.get("APP_ENV") not in {None, "test", "testing", "dev", "local"}:
        pytest.exit(f"APP_ENV={os.environ['APP_ENV']!r} is not a test environment.", returncode=3)

    print(f"\ntests will use: {host}/{name}")      # visible in every run, on purpose
```

`pytest.exit` stops the session immediately — unlike a failing assertion, nothing else gets a
chance to write. Print the target on every run: a human noticing the wrong hostname scroll by
is the last line of defense, and it only works if the value is on screen.

The same instinct applies beyond the database: no live email or SMS, no real card charges, no
writes to a production bucket or third-party API. Fakes, sandboxes, and provider test modes.

## Rules

1. **Yield and clean up.** Any fixture that acquires something releases it after `yield` (in
   a `finally`). A leaking fixture is a flaky suite waiting to happen.
2. **No branching on which test is running.** A fixture that inspects `request.node.name` and
   behaves differently is hidden control flow — split it in two.
3. **One job per fixture.** `setup_everything` is not a fixture, it is a script.
4. **Name it as a noun**, after what it *is*: `admin_user`, `db_session`, `frozen_time` —
   not `setup_user`, `get_db`, `do_freeze`.
5. **Autouse sparingly.** Only for genuinely universal concerns (env isolation, blocking real
   network). Autouse acts at a distance and is hard to trace from a failing test.
6. **No `try/except ImportError`** papering over a missing dev dependency — declare it.
7. **Order-independence is a hard requirement.** If a test only passes when it runs second,
   a fixture is leaking. Verify with a shuffled run (`pytest-randomly`) and by running single
   tests alone.
8. **Block accidental *external* network** at the root, so a test cannot quietly depend on
   the internet — while leaving loopback open, because the integration tier's database and the
   e2e tier's server both live there. Blocking all sockets would break your own suite:

```python
@pytest.fixture(autouse=True)
def no_external_network(monkeypatch, request):
    """Loopback stays open (test database, e2e server); the internet does not."""
    if "external" in request.keywords:
        return

    import socket

    real_connect = socket.socket.connect
    local = {"127.0.0.1", "::1", "localhost", ""}

    def guarded(self, address, *args, **kwargs):
        host = address[0] if isinstance(address, tuple) else address
        if host in local:
            return real_connect(self, address, *args, **kwargs)
        raise RuntimeError(f"outbound network call to {host!r} — stub it instead")

    monkeypatch.setattr(socket.socket, "connect", guarded)
```

   Mark the rare test that genuinely needs the internet with `@pytest.mark.external`, and
   deselect that marker in CI's default run.
