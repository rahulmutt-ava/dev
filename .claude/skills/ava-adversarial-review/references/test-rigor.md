# Slice: Test quality & rigor

Judge whether the tests shipped with a change are actually sufficient — not
whether tests *exist*. In this repo a passing happy-path test on a codec,
consensus, math, or DB change is a false sense of safety.

## What inadequate testing looks like here
- **No error-path coverage**: the function returns `ErrX` but no test triggers it
  with `require.ErrorIs(t, err, ErrX)`.
- **No fuzz / round-trip for serialization**: codec, message, or block-format
  changes without a `Fuzz*` and a `unmarshal→marshal` byte-identity test (and,
  for persisted/wire formats, golden vectors from the prior release).
- **No determinism guard**: a change to consensus/validator/ordering logic with
  no property test and nothing that would fail under `-shuffle=on -race`.
- **Mocks that assert nothing**: a `gomock` controller with no `EXPECT()`, or
  `EXPECT()` set on a path the test never exercises.
- **Assertion that ignores the result**: `require.NoError(t, db.Put(k,v))` with no
  later `Get` to confirm the value landed.
- **Interface impl not run through the shared suite**: a new DB not run through
  `database/dbtest`, a new codec not through `codec/codectest`.
- **Boundary values untested**: math/bounds code tested only mid-range (skip 0,
  1, max, overflow, empty/nil/oversized slice).

## Repo conventions (enforced)
- `require`, not `assert`; pass `t` explicitly — **no `require.New(t)`**.
- `forbidigo`: use `require.ErrorIs` (not `require.Error`/`ErrorContains`),
  `Equal` (not `EqualValues`), `t.Fatal` (not `require.Fail/FailNow`).
- `usetesting`: `t.TempDir()`, `t.Setenv()`, `t.Context()` — not the `os.*` /
  `context.Background()` forms.
- `testifylint` runs near enable-all (prefer `require.NoError`, `require.Len`, …).
- Table tests with `t.Run(name, …)`; descriptive case names; fixed RNG seeds for
  reproducibility; shared suites for interface implementations; narrow local
  (unexported) mocks generated to `*mock` / `mocks_test.go`.
- Assertion messages should name the operation under test, e.g.
  `require.NoError(t, err, "snapshot.New()")`.

## Adversarial questions about a diff's tests
- Is every new error return exercised by a test asserting the specific error?
- For serialization changes: is there a fuzz test, a round-trip test, and (for
  persisted/wire types) a backward-compat golden test?
- For consensus/ordering/validator changes: is determinism tested (property test
  / explicit ordering assertions / passes shuffled+raced)?
- Do tests assert the *effect* (read back the state), not just "no error"?
- If mocks are used, do they have configured expectations that the test actually
  triggers, and would the test fail if the call didn't happen?
- Are boundary/edge inputs covered (0, max, overflow, empty, nil, oversized)?
- Does a new interface implementation run the shared conformance suite?

## Red-flag shapes
- `require.New(t)` (forbidden style), `assert.*`, `require.Error(err)` w/o type.
- A codec/block/message change with zero `Fuzz*` / round-trip tests.
- `gomock.NewController(t)` with no `EXPECT()`, or unused expectations.
- `t.Skip(...)` with no issue link/justification.
- Table loop with no `t.Run` (first failure hides the rest).
- New DB/codec type with its own ad-hoc tests instead of the shared suite.
