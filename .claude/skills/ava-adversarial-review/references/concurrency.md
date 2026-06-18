# Slice: Concurrency — races, deadlocks, lock ordering, goroutine leaks

Areas: anything with `sync.Mutex/RWMutex`, `sync.Once`, `atomic.*`, channels,
`go func()`, `errgroup`, shutdown/`Close`, background loops — heavily in
`network/`, `snow/`, `database/`, `vms/`, `x/`.

CI runs unit tests with `-race -shuffle=on` (`scripts/build_test.sh`). A race
the author didn't hit locally will surface in CI or, worse, only on mainnet.

## What can go wrong
- **Data race**: shared field read without the lock that guards its writes;
  `atomic.Bool/Pointer` field paired with a non-atomic sibling that's read in the
  same logical step (atomicity of each field ≠ atomicity of the pair).
- **Lock-ordering inversion / deadlock**: two locks acquired in opposite orders
  on two paths; cleanup/`Close` taking locks in the reverse order of the hot
  path. Look for documented invariants like `// Invariant: [A] is never grabbed
  while holding [B]` — then check every site actually obeys them.
- **Re-entrant deadlock**: a method holds `lock` then calls a helper that takes
  the same (non-reentrant) `lock`, often via the underlying DB.
- **Goroutine leak**: `go func()` not tracked by a `WaitGroup`/`errgroup` and not
  guaranteed to exit via context or a close channel; a goroutine blocked sending
  on a channel nobody drains after shutdown.
- **Channel misuse**: send-on-closed / double-close panic; unclear ownership of
  who closes a channel; `select` missing a `<-ctx.Done()` / `<-closed` branch so
  it blocks forever.
- **TOCTOU on shutdown**: `if !closed { use() }` where the check and use aren't
  under the same lock; an operation proceeds after `Close` set `closed=true`.
- **`sync.Once` ordering**: state expected to be set before/after the once'd
  closure isn't, so a concurrent caller sees a half-initialized object.

## Repo patterns to check against
- Lock invariants are documented in comments near the fields — a new lock or a
  new acquisition order must state and honor its ordering.
- Long-lived goroutines are typically registered (`wg.Add(1)` before `go`,
  `defer wg.Done()`) and exit on a `closeCh`/context; `Close` then `wg.Wait()`s.
  A new goroutine that skips this is the leak.
- `database/corruptabledb` and similar wrappers serialize via locks; wrapping or
  bypassing them changes the concurrency contract.

## Adversarial questions
- For every field touched: which lock guards it? Is every read and write under
  that lock? Is an `atomic` field ever read together with a non-atomic field as
  if the pair were consistent?
- If two locks can be held at once, is the order identical on every path,
  including error/cleanup/`Close`? Construct the opposite-order interleaving.
- For every `go func()`: who waits for it? How does it exit on shutdown? Can it
  block forever on a channel or lock during `Close`?
- For every channel: exactly one closer? Can a send happen after close? Does
  every blocking `select` also select on cancellation?
- Can `Close`/shutdown race an in-flight operation and cause use-after-close,
  double-release of a quota/semaphore, or a panic?

## Red-flag shapes
- `lock.Lock()` … call into a method/DB that may take `lock` again.
- `Close()` iterating a collection and locking each element while holding the
  outer lock, when the element's own method locks in the other order.
- `go func(){ ... ch <- x ... }()` with no `ctx`/`closed` branch and no tracking.
- `if t.closed { return } ... // long op` outside the lock that sets `closed`.
- `atomic.Bool` flag + non-atomic map/slice mutated in the same method.
- Verify locally: `go test -tags test -race -count=5 -run <Test> ./<pkg>`.

## Known false positives — do NOT report these
- **"Use-after-free / dangling stack pointer"**: Go is garbage-collected with
  escape analysis. `&local` / `&param.field` / `&loopvar` escaping to the heap
  (stored in a struct, returned, captured by a goroutine) is **safe**; the
  backing memory lives as long as it's referenced. There is no stack reuse / UAF
  in pure Go — only flag it when `unsafe`, `cgo`, or `sync.Pool` object reuse is
  involved. A data race is about *unsynchronized concurrent access* to live
  memory, which is a different (real) concern — keep the two distinct.
- **Loop-variable capture**: as of Go 1.22 each iteration has its own loop
  variable, so `for i := range xs { go func(){ use(i) }() }` no longer races on
  `i`. Don't report the classic pre-1.22 capture bug (this repo is on Go 1.25).
