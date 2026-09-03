# What to Test — and What Not To

## Picking a tier

Ask: *if this breaks in production, what is the smallest test that would have caught it?*

**Unit** when the risk is in the logic itself: branching, arithmetic, parsing, validation
rules, state machines, error mapping, formatting. Cheap enough to cover every branch.

**Integration** when the risk is in the wiring: does the route call the service with the right
arguments, does the ORM query actually return what the model promises, does the migration
apply, does the auth middleware reject the request, does the transaction roll back. Bugs here
are invisible to unit tests because unit tests replace exactly the part that is broken.

**E2E** when the risk is in the whole assembled system as a user meets it: login, checkout,
signup, the one or two flows the business dies without. Slow and flaky by nature — spend them
on critical paths, not on coverage.

Heuristic for a healthy suite: many unit, a solid layer of integration, a handful of E2E.
If E2E is where most of your tests are, they are compensating for missing lower tiers.

## Do not overtest

Delete or never write:

- Tests of the language, the framework, or a third-party library. `assert 1 + 1 == 2`.
- Getters, setters, plain dataclasses, `__repr__`, constants, config literals.
- The same behavior asserted at two tiers because both were easy. Pick the tier that owns it.
- Tests that restate the implementation line by line — they fail on every refactor and catch
  nothing. If renaming a private method breaks a test, the test was wrong.
- Assertions that a mock was called, when the observable outcome is right there to assert on.
- Snapshot tests of large blobs nobody reads before re-blessing.
- Tests written to move a coverage number.
- Ten parametrize cases that all exercise one branch. Cover branches, not permutations.

Signals you have overtested: the test file is longer than the module and mostly setup;
a one-line behavior change reddens fifteen tests; nobody reads the failures, they just
re-record them.

## Do not undertest

Always cover:

- Every error path the code explicitly raises or handles. Untested `except` blocks are where
  bugs hide, because they only run on a bad day.
- Every boundary the code names: limits, quotas, expiries, page sizes, retry counts.
- Every branch of a conditional that encodes a business rule.
- Authorization: not just "logged-out is refused" but "a logged-in user cannot touch another
  user's record."
- Anything that has broken before. A bug that shipped once gets a permanent test.
- The contract at each seam you own: what the service returns, what the route responds,
  what shape the serializer emits.
- Data-destroying operations: delete, overwrite, bulk update, migration down-path.

Signals you have undertested: you cannot refactor without manual verification; a green suite
does not make you willing to deploy on a Friday.

## The three passes

Run every behavior through all three before you call it covered.

### 1. Normal use
Realistic data, the path the feature exists to serve, the way the happy demo goes.

### 2. Edge cases
- Empty: `""`, `[]`, `{}`, `None`, zero rows, no results.
- One, and exactly-at-the-limit, and limit ± 1.
- Maximum: max int, max length, max upload size, max page.
- Duplicates: same item twice, same email registering twice, replayed request.
- Text: unicode, emoji, RTL, combining characters, `\r\n`, leading/trailing whitespace,
  case differences in emails and usernames.
- Numbers: negative, zero, float precision, money as float (a bug in itself), rounding.
- Time: expired-by-one-second, not-yet-valid, timezone boundaries, DST, leap day, clock skew.
- Ordering: results with equal sort keys, pagination while rows are inserted.
- Concurrency: two writes to one row, double-click submit, race between check and act.

### 3. Bizarre and hostile behavior
Real users do all of this by accident; attackers do it on purpose.

- Submits the form twice; hits back and submits again; refreshes on the POST result.
- Opens two tabs, edits the same record in both, saves the stale one.
- Session expires mid-form; token expires between page load and submit.
- Edits the hidden `user_id` in the payload to someone else's; guesses sequential IDs.
- Skips step 2 of a wizard by deep-linking to step 3.
- Sends the request with fields missing, extra, null, wrong type, or deeply nested.
- Pastes 10MB into a name field; uploads a 0-byte file; uploads a `.jpg` that is a script.
- Types `'; DROP TABLE`, `<script>`, `../../etc/passwd`, `{{7*7}}` into every input.
- Disconnects mid-upload; sends requests out of order; retries a completed payment.
- Uses the API directly, ignoring every constraint the UI enforced.

Pick the ones that can actually break *this* code. Then say in your report which you
considered and skipped, so the gap is a decision rather than an oversight.

## A test that is worth keeping

- Fails for exactly one reason, and the message names it.
- Would catch a plausible mistake a competent person could make here.
- Reads as a specification of the behavior, not a transcript of the code.
- Passes alone, in any order, on any machine, every time.
- Costs less to maintain than the bug it prevents.
