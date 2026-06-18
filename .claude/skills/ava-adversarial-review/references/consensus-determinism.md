# Slice: Consensus safety, liveness & determinism

Areas: `snow/` (snowman, snowball, consensus engines), `simplex/`,
`vms/proposervm/`, block accept/reject, preference selection, anything whose
output feeds a decision all nodes must agree on.

## Why this is the highest-stakes slice
Every honest node must compute **bit-identical** results from the same inputs. A
single nondeterministic branch — map iteration order, wall-clock time, float
math, goroutine scheduling leaking into state, an unstable tie-break — means two
nodes derive different state/block IDs and the network **forks**. Liveness bugs
(a validator that can never make progress, a falter flag that never resets) halt
the chain. Acceptance must be final and irreversible.

## What can go wrong
- **Nondeterministic iteration**: `for k := range someMap` where the loop order
  affects a decision, accumulated value, selection, or even logged/metered state
  that later gates behavior. Go randomizes map order by design.
- **Unstable tie-breaking**: picking a "winner" (mode, max, first) among equal
  candidates without a total order (e.g. compare by ID bytes).
- **Wall-clock in consensus logic**: branching on `time.Now()` / local clock
  instead of the block timestamp; clock skew → different code path per node.
- **Float / `map`-backed math** in a consensus-critical computation.
- **Acceptance not idempotent / reversible**: a block re-added after rejection,
  `Accept` callable twice, parent accepted after child.
- **Height/ancestry not validated**: trusting a VM-supplied height without
  checking `child.Height == parent.Height + 1`, allowing a malformed tree.
- **Liveness**: a "should falter / retry" flag that can be set but never cleared,
  or a slot/turn calculation that can permanently exclude an honest proposer.

## Repo patterns to check against
- The codebase deliberately uses deterministic structures (height-keyed maps,
  sorted iteration, `slices.Sort`/`bytes.Compare` before ranging). A new
  `range map` in a decision path is a regression against this norm.
- proposervm gates behavior on upgrade activation by **block timestamp**
  (`Upgrades.IsXActivated(blockTime)`), never `time.Now()`. Verify any new
  time-based branch uses the block/parent timestamp.
- Engines generally serialize Add/RecordPoll/Accept; if a diff introduces
  concurrent access to consensus state, that's a new invariant — challenge it.

## Adversarial questions
- Does any new loop iterate a map (or other unordered set) and use the order to
  decide, select, accumulate, or order side effects? If yes: would two nodes
  with identical history produce identical output? Prove it.
- On a tie (equal votes/weights/counts), is the winner chosen by a stable total
  order? What happens with 2+ equal candidates?
- Does any branch depend on local time, scheduling, or `Math.rand`? Is the seed
  fixed / is the value derived from consensus data?
- Is `Accept` guaranteed once-per-block and never reversible? Can a rejected
  block or its descendants reappear in the active set?
- Are heights/parent links validated before the block enters the tree?
- Can any "retry/falter/wait" state be set and never reset, blocking finality?

## Red-flag shapes
- `for id := range node.children { ... decide ... }` — sort keys first.
- `mode, _ := bag.Mode()` used to pick preference — confirm tie determinism.
- `if time.Now().After(x)` / `clock.Time()` inside accept/verify/select logic.
- `delete(blocks, id)` near where `preference`/last-accepted is dereferenced.
- New `float64` in a path that feeds a block ID, state root, or vote tally.
