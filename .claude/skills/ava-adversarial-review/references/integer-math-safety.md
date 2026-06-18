# Slice: Integer overflow, math safety & resource bounds

Areas: any arithmetic on `uint*`/`int*`, type conversions, `utils/math`
(imported as `safemath`), gas/fee math, allocation sizing, loop bounds — and
**all of `vms/saevm/`**, which is held to a stricter bar.

## The saevm G115 bar (important)
`.golangci.yml` excludes gosec **G115** (integer-conversion overflow) repo-wide,
but `Taskfile.yml`'s `lint-saevm` task runs gosec over `./vms/saevm/...` with
**G115 enabled**. So inside `vms/saevm/`, every narrowing/sign-changing
conversion must be provably safe or carry a justified `//#nosec G115` comment.
See `vms/saevm/docs/invariants.md`.

## What can go wrong
- **Overflow/underflow** in `+`/`-`/`*` on fixed-width ints: a wrapped gas/fee
  value diverges between nodes (fork), inflates funds, or panics in a loop. For
  unsigned types, `a - b` with `a < b` wraps to a huge number.
- **Unsafe narrowing (G115)**: `uint64 → uint32/int`, or `int64 → uint64` of a
  possibly-negative value (e.g. `uint64(rpcBlockNumber)` without a `>= 0` check;
  `uint64(t.Unix())`), silently truncating or wrapping.
- **Division by zero**: dividing by a config/derived value that isn't guaranteed
  non-zero (`MulDiv`/`CeilDiv` panic on zero denominator).
- **Attacker-controlled allocation/loop bound**: `make([]T, n)` or `for i<n` with
  `n` from RPC/peer/decoded input and no hard cap → OOM or panic.
- **Float→int casts** of an unbounded product.

## Repo patterns to check against
- `safemath.Add/Sub/Mul` return an error on overflow/underflow — use them for any
  value derived from untrusted or unchecked input, and **handle the error**.
- `vms/saevm/intmath` offers `BoundedAdd/Sub/Multiply` (clamp to a ceil/floor),
  `MulDiv`, `CeilDiv`. Bounded variants clamp instead of erroring — confirm the
  caller actually wants clamping, not an error.
- saevm gas/time code validates config non-zero (`GasPriceConfig.Validate`:
  `TargetToExcessScaling != 0`, `MinPrice != 0`) and clamps targets to
  `[MinTarget, MaxTarget]` to avoid div-by-zero and rate overflow.
- RPC fee-history-style endpoints clamp requested `blocks` to a config max
  (`HistoryMaxBlocks`) and against underflow before allocating.
- Legitimate `//#nosec G115` always states *why* it's safe ("non-negative check
  above", "bounded by config", "Unix() is always ≥ 0"). A bare suppression is a
  red flag.
- Never write `time.Add(...TauSeconds)` — a CI grep (`tausecondslint`) fails on
  it; use a `time.Duration` (`params.Tau`).

## Adversarial questions
- For each `+`/`-`/`*` on a fixed-width int from untrusted/unchecked input: can
  it overflow/underflow? Is `safemath`/`intmath` used and is the error handled?
- For each conversion: can the source exceed the destination's range, or be
  negative when the destination is unsigned? Is there a guard or a justified
  `//#nosec G115`? (Mandatory scrutiny inside `vms/saevm/`.)
- For each division: is the denominator guaranteed non-zero (by validation or a
  documented invariant)?
- Does any `make`/loop bound come from RPC/peer/decoded input without a cap?
- Any unsigned `for n := count; n > 0; n--` where `count` could be huge/wrapped?

## Red-flag shapes
- `uint64(x)` where `x` is `int64`/`rpc.BlockNumber`/signed, no `>= 0` check.
- `uint32(bigUint64Value)` / `int(uint64Value)` with no bound.
- `a - b` on `uint*` without a prior `a >= b` guard.
- `x / y` / `MulDiv(_, _, y)` where `y` is config-derived and unvalidated.
- `make([]T, n)` with attacker-influenced `n` and no `min(n, cap)`.
- `//#nosec G115` with no justification, anywhere — especially in `vms/saevm/`.
