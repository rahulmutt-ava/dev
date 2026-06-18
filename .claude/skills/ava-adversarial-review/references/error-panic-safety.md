# Slice: Error handling, panic safety & graceful degradation

Areas: everywhere, but especially message/RPC/block-processing paths and
shutdown. In a node, **silence is dangerous**: a swallowed error can cause
silent state divergence, and a reachable panic is a remote DoS.

## What can go wrong
- **Swallowed/ignored error**: `_ = f()` or "log and continue" on a write or
  state mutation, so the code proceeds as if it succeeded → silent divergence.
- **Reachable panic from untrusted input**: unchecked type assertion `x.(T)`
  (panics on the wrong type), slice/index out of range, nil-map write, nil deref
  after a parse that can return nil — on any path reachable from a peer message,
  RPC arg, or decoded block.
- **Partial mutation on error**: an error midway through a multi-step state change
  leaves it half-applied with no rollback.
- **Wrong panic vs error choice**: panicking where an error should be returned
  (untrusted input), or returning an error where the design demands a fatal halt.
- **Context misuse**: `context.Background()`/`TODO()` on a path that needs
  cancellation/timeout, so a hung dependency blocks forever; a blocking op with
  no `<-ctx.Done()`.
- **Lost error chain**: `fmt.Errorf("%v", err)` instead of `%w`.

## Repo patterns to check against
- `errcheck` is enabled — every returned error must be checked; an intentional
  ignore needs `//nolint:errcheck` (or a documented `_ =`) with a reason.
- Prefer `errors.New` over `fmt.Errorf` when there's no `%` verb (a revive
  `string-format` rule enforces this); wrap with `%w` to preserve the chain.
- Type assertions in this codebase normally use comma-ok (`v, ok := x.(T)`) or a
  `switch x.(type)`. A bare `x.(T)` on an external-input path is the panic.
- Panics are acceptable in init-only code (e.g. `MustCommit`) and tests, and the
  chain handler uses structured `RecoverAndExit`/`RecoverAndPanic` per chain
  criticality. Outside those, prefer returning an error.
- `corruptabledb` fails closed on unexpected DB errors rather than continuing.

## Adversarial questions
- Is this error reachable from untrusted input? If so, does the code return an
  error (good) or panic/deref (bad)?
- Is every returned error checked and propagated? Any `_ =` / log-and-continue on
  a write or state change — is that intentional and documented?
- Every `x.(T)`: is it comma-ok or in a type switch? Can the dynamic type differ?
- Every index/slice/map-write on decoded or peer data: is the bound/non-nil
  guaranteed?
- On each error path through a multi-step mutation, is state left consistent
  (rollback / single batch / abort)?
- Do blocking calls honor a cancelable context with a timeout?
- Is wrapping `%w` (chain preserved) rather than `%v`?

## Red-flag shapes
- `v := x.(SomeType)` (no `, ok`) on a value from a message/RPC/decode path.
- `_ = something()` where something writes state, with no explanatory comment.
- `data[i]` / `data[n:]` where `i`/`n` derives from input without a length check.
- `context.Background()` / `context.TODO()` in a network/RPC/long-running path.
- `fmt.Errorf("...%v", err)` where the chain should be preserved (`%w`).
- `panic(...)` reachable from block/message processing rather than a returned error.

## Known false positives — do NOT report these
- **"Use-after-free / dangling stack pointer"**: Go is garbage-collected with
  escape analysis. Taking `&local`, `&param.field`, or `&loopvar` and storing or
  returning the pointer is **safe** — the value escapes to the heap and stays
  valid as long as anything references it. There is no stack reuse / UAF in pure
  Go. Only treat this as real when `unsafe`, `cgo`, raw `reflect` pointer
  arithmetic, or `sync.Pool` reuse is involved. Refute the claim otherwise.
- **Pointer aliasing**: storing `&param.field` *can* alias if the caller keeps and
  later mutates that exact struct, or the same struct is reused across iterations.
  Before reporting, confirm the source is actually retained/mutated/reused — a
  by-value parameter whose caller passes a fresh literal each call is not aliased.
- **`%w` vs `%v`** and other lint-caught style: note under "Lint/CI will catch",
  not as a finding.
