# Slice: SAE VM correctness & gas/fee economic accounting

Areas: `vms/saevm/` — `sae`, `cchain`, `gastime`, `gasprice`, `proxytime`,
`worstcase`, `hook`, `saexec`, `blocks`, `txpool`. SAE = Streaming Asynchronous
Execution (ACP-194). The source of truth for invariants is
`vms/saevm/docs/invariants.md` — read it; "if the code differs there is a bug
until proven otherwise."

## Why this is dangerous
Bugs here are economic (mint funds from nothing, bypass fees, underflow a
balance) or correctness (execution diverges between nodes; a tx the mempool
admits fails at execution, or vice versa) — both can fork the chain or lose
funds. SAE separates **Accepted → Executed → Settled**; many bugs live in the
gap between those phases.

## The phase invariants (from invariants.md)
- Data equivalence: Accepted↔Canonical, Executed↔Head, Settled↔Finalized.
- Happens-before (witnessing the left lets you assume the right already happened):
  `Settled ⇒ Executed`; `Executed ⇒ Accepted`; external-indicator ⇒
  internal-indicator ⇒ memory ⇒ disk ⇒ condition; a side effect for block `n`
  happens after the same side effect for block `n-1` (in-order); a side effect of
  accepting `b_n` happens after the equivalent side effect for every block that
  acceptance settles.
- `MarkSettled()` must be called **in order** across the settled set; the
  last-settled pointer update may lag. Realize polling side effects (atomic
  pointer updates) before broadcasts (channel sends / API events).

## What can go wrong
- **Value non-conservation**: a `hook.Op` mint not tied to a corresponding burn /
  cross-chain import; total supply drifts.
- **Balance underflow / insolvency**: deducting more than available; a min-balance
  check that doesn't actually prevent the debit.
- **mempool ↔ consensus divergence**: the txpool admits against last-executed
  state, the builder simulates worst-case over unsettled blocks, and execution
  re-runs from parent state. If these three don't agree (balance, nonce, base-fee
  rules), a tx is admitted then fails (or block validity disagrees between nodes).
- **Worst-case bound too loose/tight**: `MaxBaseFee` / min-burner-balance bounds
  computed at build time must hold at execution; if actual base fee > the bound,
  execution-time balance checks can fail.
- **Gas accounting gaps**: per-block gas not counting end-of-block ops; excess /
  base-fee update (ACP-176 exponential) overflow or rounding error.
- **Nonce / replay**: nonce not incremented at `MaxUint64` (relies on the account
  being delegated) — confirm that assumption holds for the caller.
- **Settlement before execution**, or re-execution/re-settlement of a settled
  block, or out-of-order `MarkSettled`.
- **Nondeterministic execution**: wall-clock or map order leaking into executed
  state (see also consensus-determinism.md).

## Adversarial questions
- Does any `Op` increase a balance without an equal, traceable decrease? Walk
  mint vs burn.
- Do txpool admission, worst-case simulation, and actual execution use identical
  balance/nonce/base-fee logic? Where could they diverge across intervening
  blocks?
- Are the build-time worst-case bounds guaranteed ≥ what execution computes?
- Does per-block gas include *all* ops (txs + end-of-block)? Can excess/base-fee
  math overflow or round the wrong way?
- Is `MarkSettled` in-order? Can a block be settled before executed, or executed
  twice? Are atomic pointers updated before broadcasts?
- Is anything in the execution path dependent on local time or map iteration?

## Red-flag shapes
- A new `hook.Op` with a `Mint` not balanced by a `Burn`/import.
- `balance.Sub(amount)` without a checked `balance >= amount` (use safemath).
- Divergent validity logic between `txpool` and `worstcase`/`saexec`.
- A broadcast (channel close / event emit) before the corresponding atomic
  pointer / disk write (violates the happens-before order).
- New gas/excess arithmetic without overflow handling (see integer-math-safety.md).
- `time.Now()` / `range map` inside execution or settlement.
