# E2E with pytest-playwright

Playwright is the default here: it ships browsers, auto-waits (killing the sleep-based
flakiness that plagues Selenium), traces failures, and drives Chromium/Firefox/WebKit from one
API. Use **Selenium** only when the repo already standardizes on it, or when a legacy grid or
a browser Playwright does not support is a hard requirement — the structure below carries over.

## Install

```bash
pip install pytest-playwright
playwright install chromium        # add --with-deps in CI
```

Add `pytest-playwright` to the dev requirements (see `dev-dependencies.md`), and the browser
install step to CI.

## Serve the app for the test run

```python
# tests/e2e/conftest.py
import os
import socket
import subprocess
import time

import pytest


def _free_port():
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]


@pytest.fixture(scope="session")
def base_url():
    """Start the app once for the whole e2e session, on a port nothing else owns."""
    port = _free_port()
    env = {**os.environ, "APP_ENV": "test", "PORT": str(port)}
    proc = subprocess.Popen(
        ["python", "-m", "uvicorn", "app.main:app", "--port", str(port)],
        env=env,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
    )
    url = f"http://127.0.0.1:{port}"
    deadline = time.time() + 30
    while time.time() < deadline:
        try:
            with socket.create_connection(("127.0.0.1", port), timeout=0.5):
                break
        except OSError:
            if proc.poll() is not None:
                raise RuntimeError(f"server died:\n{proc.stdout.read().decode()}")
            time.sleep(0.2)
    else:
        proc.kill()
        raise RuntimeError("server did not start within 30s")
    yield url
    proc.terminate()
    proc.wait(timeout=10)
```

`base_url` is the name `pytest-playwright` already understands, so `page.goto("/login")`
resolves against it.

## Fail the test on console errors and bad responses

The single highest-value e2e fixture — it catches breakage no assertion was written for.

```python
@pytest.fixture(autouse=True)
def page_is_clean(page):
    problems = []
    page.on("console", lambda m: m.type == "error" and problems.append(f"console: {m.text}"))
    page.on("pageerror", lambda e: problems.append(f"pageerror: {e}"))
    page.on("requestfailed", lambda r: problems.append(f"failed: {r.method} {r.url}"))
    page.on(
        "response",
        lambda r: r.status >= 500 and problems.append(f"{r.status}: {r.url}"),
    )
    yield
    assert problems == []
```

## Reuse login instead of re-typing it

```python
@pytest.fixture(scope="session")
def storage_state(browser, base_url, tmp_path_factory):
    path = tmp_path_factory.mktemp("auth") / "state.json"
    context = browser.new_context(base_url=base_url)
    page = context.new_page()
    page.goto("/login")
    page.get_by_label("Email").fill("e2e@example.com")
    page.get_by_label("Password").fill("correct-horse")
    page.get_by_role("button", name="Sign in").click()
    page.wait_for_url("**/dashboard")
    context.storage_state(path=path)
    context.close()
    return str(path)


@pytest.fixture
def authed_page(browser, base_url, storage_state):
    context = browser.new_context(base_url=base_url, storage_state=storage_state)
    page = context.new_page()
    yield page
    context.close()
```

Log in through the UI once — that flow deserves its own explicit test too.

## State isolation — where e2e flake actually comes from

The transaction-rollback trick from the integration tier **does not work here**: the app runs
in another process with its own connections, so your test can neither see nor undo its writes.
Without a deliberate reset, test 3 passes alone and fails after test 2, and the suite becomes
something people re-run instead of trust.

Pick one, in order of preference:

**1. Unique data per test.** Nothing to reset if nothing collides. Cheapest and least flaky.

```python
@pytest.fixture
def unique():
    return lambda prefix: f"{prefix}-{uuid.uuid4().hex[:8]}"


def test_create_project(authed_page, unique):
    name = unique("proj")
    ...
```

**2. Truncate between tests**, when the app needs a known-empty state.

```python
@pytest.fixture(autouse=True)
def reset_db(engine):
    yield
    with engine.begin() as conn:
        tables = ",".join(t.name for t in reversed(Base.metadata.sorted_tables))
        conn.exec_driver_sql(f"TRUNCATE {tables} RESTART IDENTITY CASCADE")
```

**3. A test-only reset endpoint**, registered only when `APP_ENV == "test"`. Effective, but it
is a data-destroying route — guard it hard and never let it reach a real deploy.

Whichever you choose, seed via the API or a factory rather than clicking through the UI: a
signup flow used as setup for thirty tests will eventually fail thirty tests for one reason.

**The e2e tier is the most dangerous one.** It runs a real server against a real database with
no transaction to roll back — the truncate fixture above will happily erase whatever it is
pointed at. Never let e2e tests share a database with anything a human uses, and never run
them against a deployed environment. Boot the server yourself against a dev database, as the
`base_url` fixture does, with `APP_ENV=test` set explicitly. State the e2e database in
`TEST_PLAN.md` so nobody later points the suite at staging.

## Locate by what the user sees

```python
page.get_by_role("button", name="Save").click()
page.get_by_label("Email").fill("user@example.com")
expect(page.get_by_text("Saved")).to_be_visible()
```

Roles, labels, and text. `data-testid` only when nothing user-visible identifies the element.
Never CSS classes — they change for cosmetic reasons and the test lies about why it failed.

Use `expect()` (auto-retrying) rather than a bare `assert` on a value read once:

```python
from playwright.sync_api import expect

expect(page.get_by_role("alert")).to_have_text("Card declined")
```

## Force the states users actually hit

```python
def test_shows_error_when_api_fails(page):
    page.route("**/api/items", lambda r: r.fulfill(status=500, body="{}"))
    page.goto("/items")
    expect(page.get_by_role("alert")).to_be_visible()


def test_shows_empty_state(page):
    page.route("**/api/items", lambda r: r.fulfill(status=200, body="[]"))
    page.goto("/items")
    expect(page.get_by_text("Nothing here yet")).to_be_visible()
```

## Bizarre user behavior — the tier where it belongs

```python
import re

from playwright.sync_api import expect


def test_double_click_submit_creates_one_record(authed_page):
    authed_page.goto("/items/new")
    authed_page.get_by_label("Name").fill("Widget")
    button = authed_page.get_by_role("button", name="Create")
    button.dblclick()
    authed_page.goto("/items")
    expect(authed_page.get_by_role("row", name="Widget")).to_have_count(1)


def test_back_button_after_logout_does_not_show_data(authed_page):
    authed_page.goto("/dashboard")
    authed_page.get_by_role("button", name="Log out").click()
    authed_page.go_back()
    expect(authed_page).to_have_url(re.compile("/login"))


def test_stale_tab_save_does_not_clobber(browser, base_url, storage_state):
    a = browser.new_context(base_url=base_url, storage_state=storage_state).new_page()
    b = browser.new_context(base_url=base_url, storage_state=storage_state).new_page()
    for p in (a, b):
        p.goto("/items/1/edit")
    b.get_by_label("Title").fill("from B")
    b.get_by_role("button", name="Save").click()
    a.get_by_label("Title").fill("from A")
    a.get_by_role("button", name="Save").click()
    expect(a.get_by_role("alert")).to_contain_text("changed since you opened")
```

Also worth covering at this tier: session expiring mid-form, deep-linking past a wizard step,
refresh on a POST result, and a protected URL opened while logged out.

## Responsive and accessibility

```python
@pytest.mark.parametrize(
    "width, height",
    [(375, 812), (768, 1024), (1440, 900)],
    ids=["mobile", "tablet", "desktop"],
)
def test_no_horizontal_overflow(page, width, height):
    page.set_viewport_size({"width": width, "height": height})
    page.goto("/")
    overflow = page.evaluate(
        "() => document.documentElement.scrollWidth > window.innerWidth"
    )
    assert not overflow


def test_no_accessibility_violations(page):
    from axe_playwright_python.sync_playwright import Axe

    page.goto("/")
    results = Axe().run(page)
    assert results.violations_count == 0, results.generate_report()
```

## Running

```bash
pytest tests/e2e -q
pytest tests/e2e --headed --slowmo 300      # watch it
pytest tests/e2e --browser firefox --browser webkit
pytest tests/e2e --tracing retain-on-failure
playwright show-trace test-results/**/trace.zip
pytest tests/e2e --video retain-on-failure --screenshot only-on-failure
```

## Keep this tier small

E2E tests are the slowest and flakiest thing you own. Spend them on the handful of journeys
the app exists for; push everything else down to integration or unit. If an e2e test is the
only thing covering a validation rule, that rule is under-tested at the tier that should own it.

Never `page.wait_for_timeout()` — it is a sleep wearing a costume. Wait on a condition:
`expect(...)`, `page.wait_for_url(...)`, `page.wait_for_load_state(...)`.
