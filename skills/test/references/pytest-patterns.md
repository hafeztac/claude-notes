# pytest Patterns

Unit and integration tiers. Browser tier is in `playwright-e2e.md`.

## Config

Keep it in `pyproject.toml` if the repo already uses one; otherwise `pytest.ini`.

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q --strict-markers --strict-config"
markers = [
    "slow: takes over a second",
    "e2e: drives a real browser",
    "external: needs live credentials; deselected in CI by default",
]
```

`--strict-markers` turns a typo'd marker into an error instead of a silently un-run test.

Consider `filterwarnings = ["error::DeprecationWarning"]` only once the suite is clean —
it is valuable on your own code and infuriating on a dependency's. Scope it if you add it:

```toml
filterwarnings = [
    "error::DeprecationWarning:app.*",
    "default::DeprecationWarning",
]
```

## Shape of a test

```python
def test_expired_token_is_rejected(token_service, frozen_time):
    token = token_service.issue(user_id=7, ttl_minutes=60)   # arrange
    frozen_time.tick(delta=timedelta(minutes=61))

    result = token_service.verify(token)                     # act

    assert result.ok is False                                # assert
    assert result.reason == "expired"
```

One behavior, three visible movements, a name that states the rule. If you need "and" in the
name, it is two tests.

## Parametrize

Same behavior, many inputs:

```python
@pytest.mark.parametrize(
    "raw, expected",
    [
        ("user@example.com", "user@example.com"),
        ("  User@Example.COM  ", "user@example.com"),
        ("USER+tag@example.com", "user+tag@example.com"),
    ],
)
def test_email_is_normalized(raw, expected):
    assert normalize_email(raw) == expected
```

Give ids when the values do not read well:

```python
@pytest.mark.parametrize(
    "payload",
    [pytest.param({}, id="empty"), pytest.param({"qty": -1}, id="negative-qty")],
)
def test_invalid_payload_is_rejected(client, payload):
    assert client.post("/orders", json=payload).status_code == 422
```

Do not parametrize across *different* behaviors — three cases that expect three different
exceptions are three tests.

## Expected failures

```python
def test_negative_quantity_is_rejected():
    with pytest.raises(ValidationError, match="quantity must be positive"):
        Order(quantity=-1)
```

Always constrain with `match=` — a bare `pytest.raises(Exception)` passes on a typo in your
own test setup.

## Mocking

Mock what you do not own and cannot run locally. Do not mock what you own.

```python
def test_checkout_records_failure_when_gateway_declines(monkeypatch, orders):
    monkeypatch.setattr(
        payments, "charge", lambda **kw: ChargeResult(ok=False, code="declined")
    )

    result = checkout(order_id=1)

    assert result.status == "payment_failed"
    assert orders.get(1).status == "payment_failed"
```

Prefer `monkeypatch` and hand-written fakes over `unittest.mock.MagicMock`: a `MagicMock`
accepts any call, so it happily passes after you rename the method it was standing in for.

Patch where the name is *used*, not where it is defined:

```python
monkeypatch.setattr("app.checkout.charge", fake_charge)   # not "app.payments.charge"
```

## HTTP boundaries

```python
# requests → responses
@responses.activate
def test_sync_handles_upstream_500():
    responses.add(responses.GET, "https://api.example.com/items", status=500)
    with pytest.raises(UpstreamError):
        sync_items()


# httpx → respx
@respx.mock
def test_sync_retries_once_on_timeout():
    route = respx.get("https://api.example.com/items")
    route.side_effect = [httpx.TimeoutException("t"), httpx.Response(200, json=[])]
    assert sync_items() == []
    assert route.call_count == 2
```

## Async

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"     # pytest-asyncio; no decorator needed per test
```

```python
async def test_fetch_returns_empty_on_404(async_client):
    resp = await async_client.get("/items/999")
    assert resp.status_code == 404
```

## Database (integration)

Assert on state, not on the ORM call:

```python
def test_deleting_a_project_cascades_to_tasks(db_session, client, project):
    resp = client.delete(f"/projects/{project.id}")

    assert resp.status_code == 204
    assert db_session.get(Project, project.id) is None
    assert db_session.query(Task).filter_by(project_id=project.id).count() == 0
```

Test the migration path itself if migrations are hand-written — apply, insert, downgrade.

## Authorization

The test people forget is the second one:

```python
def test_user_cannot_read_another_users_order(client, alice, bob_order):
    client.headers["Authorization"] = f"Bearer {issue_token(alice)}"

    resp = client.get(f"/orders/{bob_order.id}")

    assert resp.status_code == 404      # not 403 — do not confirm the id exists
```

## Time

```python
def test_trial_expires_after_14_days(frozen_time, account):
    frozen_time.move_to("2026-01-29T12:00:01Z")
    assert account.is_trial_active() is False
```

Never assert against `datetime.now()`; the test will pass at 09:00 and fail at 23:59 UTC.

## Concurrency and double-submit

```python
def test_double_submit_creates_one_order(client, cart):
    body = {"cart_id": cart.id, "idempotency_key": "k-1"}
    first = client.post("/orders", json=body)
    second = client.post("/orders", json=body)

    assert first.status_code == 201
    assert second.status_code in (200, 201)
    assert second.json()["id"] == first.json()["id"]
```

## Useful invocations

```bash
pytest -q                          # quiet, whole suite
pytest tests/unit -q               # fast inner loop
pytest -m "not e2e and not slow"   # what CI runs on every push
pytest -k "token and expired"      # by name
pytest -x --ff                     # stop at first failure, failures first next run
pytest --lf                        # only last-failed
pytest -q --durations=10           # find the slow ones
pytest --cov=app --cov-report=term-missing
pytest -p no:randomly              # rule randomization in/out of a flake investigation
```

## Anti-patterns

- `time.sleep()` in a test — wait on a condition or freeze the clock.
- `assert result` — assert the actual value; the failure message should not require a debugger.
- Tests that share module-level mutable state.
- `if` statements in tests. A branch means the test does not know what it expects.
- A `try/except` swallowing the assertion.
- Asserting on log text as a stand-in for behavior.
- Tests whose names describe the code (`test_process_v2`) instead of the rule.
