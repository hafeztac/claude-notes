---
name: test
description: Write automated tests for a Python web app — unit, integration, and end-to-end — with pytest and pytest-playwright. On first run it authors a repo test plan; on every run it consults that plan, scopes to current feature/fix branch, updates dev-requirements file, keeps conftest.py clean, and writes tests that cover normal paths, edge cases, and hostile user behavior. Use when asked to write tests, add test coverage, set up a test suite, test a branch or feature, or write a regression test for a bug.
argument-hint: [path-or-scope]
allowed-tools: Bash, Read, Grep, Glob, Write, Edit
---

# Test

Scope: `$ARGUMENTS` — a path, module, route, feature name, or empty for "whatever this
branch changed".

## Non-negotiable: never test against production

**Production data integrity outranks every other goal in this skill, including finishing the
task.** Tests create, mutate, truncate, and delete rows; that is what they are for. Pointed at
a real database, this skill is a data-loss incident, and no test result is worth that.

- Tests run against a **dev/test database only** — a local instance, an ephemeral container,
  or a disposable CI database. Never production. Never staging that carries real user data.
  Never a shared database a colleague is using.
- Before the first test runs, **read the database URL you are actually about to use** and
  confirm it is local or clearly test-scoped. Do not infer it from a variable name; print it.
- Never point tests at a host you did not set up for testing, and never run a suite whose
  connection string you have not seen.
- If the only reachable database might be production — or you cannot tell — **stop and ask.**
  This is the one place in this skill where blocking on the user is correct: proceeding under
  an assumption that turns out wrong is unrecoverable.
- The same applies to every other real service: send no live email or SMS, charge no real
  card, write to no production bucket, call no third-party API in write mode. Use test modes,
  fakes, and sandboxes.
- Never copy production data into a test fixture. If realistic data is needed, generate or
  anonymize it.

Add the guard fixture from `references/conftest-guide.md` so this is enforced by the suite
itself, not just by care.

The single rule that outranks the rest: **a test exists to catch a regression a human would
care about.** If a test cannot fail for a reason that matters, it is noise — delete it.

**Scale the ceremony to the job.** The phases below are the full pass, for a branch's worth of
work. For "add a test for this one function", the honest version is Phase 0, then 6, 7 — read
the plan, write the test, run it. Skipping phases because they do not apply is correct;
skipping them because they are tedious is not. Say in the report which you skipped.

**If the user only wants to *run* existing tests**, do that directly — `pytest -q` — and stop.
This skill is for authoring tests. Do not author a plan or a suite for someone who asked for
a test run.

## Phase 0 — Consult the plan (always first)

Before reading source, running anything, or writing a line of test code, look for the plan:

```bash
ls tests/TEST_PLAN.md .claude/testing/TEST_PLAN.md 2>/dev/null
find . -name TEST_PLAN.md -not -path './.git/*' -not -path '*/node_modules/*' 2>/dev/null
```

**If a plan exists**: read it in full. It governs. Its conventions — directory layout, what
counts as a unit, what gets mocked, which fixtures already exist, what the team deliberately
does not test — override this skill's defaults. If something in it looks wrong, say so in
the final report; do not silently deviate.

**If no plan exists**: go to Phase 1 and write one. Do not skip ahead to writing tests. The
decisions the plan makes (what a unit is here, what gets a real database, what gets mocked)
determine the shape of every test that follows, and making them ad hoc per test is how a
suite becomes incoherent.

## Phase 1 — Author the test plan (first run only)

Survey the repo before writing the plan:

- App framework: FastAPI / Django / Flask / Starlette — from `pyproject.toml`,
  `requirements*.txt`, `manage.py`, `app/main.py`.
- Frontend: server-rendered templates, or a JS SPA in `frontend/`, `client/`, `web/`?
- Persistence: SQLAlchemy / Django ORM / raw driver? Which database?
- External I/O: HTTP clients, queues, caches, third-party SDKs, file/object storage.
- Existing tests, runner config, CI workflow, coverage settings.
- Entry points and the two or three flows the app exists to serve.

Then fill in `references/test-plan-template.md` and write it to **`tests/TEST_PLAN.md`**
(if the repo's tests already live elsewhere, use that root instead). It belongs beside the
tests it governs: versioned, reviewable in a PR, and legible to teammates who never run
Claude — not tucked inside a tool directory.

Show the user a short summary of the plan and tell them where it landed. Then continue; do
not stop and wait for approval unless the plan had to make a genuinely contested call.

## Phase 2 — Scope to the branch

```bash
git branch --show-current

# Detect the base branch — do not assume "main"
BASE=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||')
[ -n "$BASE" ] || BASE=$(for b in main master develop trunk; do
    git show-ref --verify --quiet "refs/heads/$b" && echo "$b" && break
done)
echo "base: ${BASE:?could not determine base branch — ask the user}"

git diff "$BASE"...HEAD --stat
git diff "$BASE"...HEAD
```

List the **behaviors** that changed, not the files. "Password reset tokens now expire after
1 hour" is a behavior; "edited `auth/tokens.py`" is not. Each behavior earns tests; each
file does not.

If the branch has no diff against the base and `$ARGUMENTS` is empty, ask the user what to
scope to rather than guessing at the whole repo.

## Phase 3 — Dev dependencies, then a baseline

Everything you import in a test must be declared. Find the repo's dev-dependency location
and add to it — never create a second, competing one. See
`references/dev-dependencies.md` for each packaging format and the baseline package set.

Install, then confirm collection works:

```bash
python -m pytest --collect-only -q
```

**Record the baseline before writing anything.** You cannot report honestly on your own tests
if you never knew what was already red:

```bash
python -m pytest -q 2>&1 | tail -30
```

Note which tests already fail and why. Those failures are not yours; conflating them with
your own in the final report is how a broken suite gets blamed on the wrong change. If
collection itself fails, fix that first — a suite that cannot collect cannot be extended.

## Phase 4 — Assign each change to a tier

For every behavior from Phase 2, pick the **cheapest tier that can actually catch its
failure**. Full decision rules and the over/under-testing guardrails are in
`references/what-to-test.md`. The short form:

| Tier | Tests | Talks to | Speed |
|---|---|---|---|
| **Unit** | one function/class in isolation — logic, branches, validation, error paths | nothing real | ms |
| **Integration** | components wired together — route + service + DB, ORM queries, migrations, auth middleware | real DB, real app object, fake third parties | 10s–100s of ms |
| **E2E** | a user's journey through the real browser | everything real | seconds |

Most changes need one tier, some two. A change that needs all three is usually a change that
should be described as several behaviors.

Do not push a test up a tier to make it easier to write. Do not push one down a tier and mock
away the thing that was actually at risk.

## Phase 5 — conftest.py

Layered, one job each. Full guidance and worked examples in `references/conftest-guide.md`.

```
tests/
  conftest.py              # truly cross-cutting only: settings, tmp paths, freeze time
  unit/conftest.py         # pure builders and fakes; no I/O whatsoever
  integration/conftest.py  # db engine (session), per-test transaction, app client
  e2e/conftest.py          # live server, browser context, seeded accounts
```

Rules that keep it clean:

- A fixture in the root `conftest.py` must be needed by two or more tiers. Otherwise push it
  down.
- Widest useful scope, no wider: `session` for expensive setup, `function` for anything a
  test mutates.
- Every fixture that acquires something releases it via `yield` + teardown.
- Each test starts from a known state — a rolled-back transaction, not leftovers from the
  test before. Order-independence is a requirement, not a nicety: verify it with a shuffled
  run (`pytest -p randomly`, from `pytest-randomly`) and by running single tests alone.
- No conditionals branching on which test is running. No fixture that does two unrelated
  things. Name fixtures after what they *are* (`admin_user`), not what they do (`setup_user`).
- Varied data belongs in factory functions, not in ten near-identical fixtures.

## Phase 6 — Write the tests

Order: unit, then integration, then e2e. Patterns in `references/pytest-patterns.md`
(fixtures, parametrize, mocking, markers, async, DB, HTTP) and
`references/playwright-e2e.md` (browser tier).

- **Do not write tests to pass**. The goal of testing is checking against current and regression defects rather than programmatic theatre.
- Be legitimately nitpicky and poke holes.
- **On a bugfix branch, write the failing test first.** Run it, watch it fail with the bug
  present, then confirm it passes against the fix. A regression test you never saw fail is
  not yet a regression test.
- Name tests for the behavior asserted: `test_expired_token_is_rejected`, not `test_token_2`.
- Arrange / Act / Assert, visibly separated. One behavior per test.
- Assert on observable outcomes — return values, status codes, DB rows, rendered text — not
  on how the code got there. Do not assert a private method was called.
- `@pytest.mark.parametrize` for the same behavior over many inputs. Separate tests for
  genuinely different behaviors, even when the code path is shared.
- Failure messages must identify the problem without a debugger. Prefer a plain `assert` with
  a real expression over a bare truthiness check.

For each behavior, deliberately think through all three of these before moving on:

1. **Normal use** — what the feature is for, with realistic data.
2. **Edge cases** — empty, zero, one, maximum, boundary ±1, null/None, duplicates,
   unicode and emoji, very long strings, concurrent writes, expired/just-valid timestamps,
   timezone and DST, floating-point money.
3. **Bizarre and hostile behavior** — double-submit, back button mid-flow, refresh on a POST
   result, two tabs editing the same record, session expiring mid-form, tampered hidden
   fields and IDs belonging to another user, negative quantities, 10MB of text in a name
   field, SQL/HTML/script payloads in every input, pasted control characters, request
   arriving out of order, client disconnecting mid-upload.

`references/what-to-test.md` has the full catalog. Not every item becomes a test — pick the
ones that would actually break *this* code, and say in the report which risks you judged not
worth a test.

## Phase 7 — Run and verify

```bash
python -m pytest tests/unit -q
python -m pytest tests/integration -q
python -m pytest tests/e2e -q          # or: -m e2e
python -m pytest -q                    # whole suite
python -m pytest --cov=<pkg> --cov-report=term-missing
```

Then sanity-check that the tests are load-bearing: revert or break the code under test and
confirm the new tests go red. A suite that passes against broken code is worse than no suite.

Also confirm they pass in isolation and in a different order:

```bash
python -m pytest tests/unit/test_foo.py::test_bar -q
python -m pytest -q --ff
```

Then wire them into CI, or they only run on your machine:

```bash
ls .github/workflows/*.yml .gitlab-ci.yml Jenkinsfile 2>/dev/null
```

- New test directories must fall under what CI invokes (`testpaths`, or the explicit path in
  the workflow).
- New dev dependencies must be in the group CI installs.
- The e2e tier needs `playwright install --with-deps chromium` as its own CI step, and usually
  its own job — deselected from the fast job with `-m "not e2e"`.
- If CI has no test step at all, say so and propose one; do not add one silently to a repo
  that deliberately runs tests elsewhere.

Report real output. Compare against the Phase 3 baseline: failures already red before you
started are not yours, and must be labeled as pre-existing rather than folded into your
results.

## Phase 8 — Report and update the plan

1. **Plan**: created, or consulted (and anything it made you do differently).
2. **Scope**: which behaviors were tested, tier by tier.
3. **New/changed files**: tests, conftests, dev-requirements.
4. **Results**: real pytest output, coverage delta if measured.
5. **Deliberately not tested**: and the reason — cost, flakiness, no credentials, low risk.
   Being explicit here is what keeps "do not overtest" from becoming "quietly undertest."
6. Append anything durable you learned — a new fixture, a new convention, a flaky area — to
   `TEST_PLAN.md` so the next run inherits it.
