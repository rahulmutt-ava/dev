---
name: ava-adversarial-review
description: >-
  Adversarial code review for the avalanchego repo (the Go Avalanche node,
  module github.com/ava-labs/avalanchego, including graft/coreth, graft/evm,
  graft/subnet-evm, and vms/saevm). Use whenever reviewing a diff, branch, PR,
  or staged change in this repo and you want to actively hunt for bugs rather
  than rubber-stamp — consensus/determinism breaks, data races and deadlocks,
  codec/wire backward-incompatibility, state/DB corruption, integer overflow
  (gosec G115, stricter in vms/saevm), crypto/signature flaws, P2P DoS vectors,
  SAE gas/fee economic bugs, swallowed errors and reachable panics, and weak
  tests. Invoke this for "adversarial review", "review this change/PR/branch
  hard", "find bugs in this diff", "what could break", "is this safe to merge",
  or the /ava-adversarial-review command — even when the user doesn't name a
  specific slice. Prefer this over a generic code review when working inside
  avalanchego because it encodes this repo's invariants, lint rules, and
  failure modes.
---

# Adversarial Review for avalanchego

Review a change as an adversary: **assume the diff is wrong and try to prove it.**
This is a blockchain node — bugs here cause consensus forks, fund loss, node
crashes (DoS), or silent state divergence, not just a failed unit test. A
plausible-looking diff that compiles and passes the happy-path test is exactly
the kind of change that ships a mainnet incident. Your job is to find the path
the author didn't think about.

This skill does **not** duplicate the built-in `code-review` / `security-review`
skills (which are confidence-gated, low-false-positive, and defer lint/CI). It
goes the other way: it reasons hard about this repo's specific invariants and
failure modes, names the exact repo rule a change violates, and surfaces
uncertain-but-plausible risks so a human can adjudicate them.

## Workflow

Follow these steps in order. Create a todo per step.

### 1. Scope the diff

Determine exactly what changed. Default to the branch diff against `master`:

```sh
git -C <repo> merge-base HEAD master        # find fork point
git -C <repo> diff <merge-base>..HEAD --stat # changed files
git -C <repo> diff <merge-base>..HEAD        # full diff
```

If the user points at a PR, a working-tree change, or a specific path, scope to
that instead. Read the full diff plus enough surrounding context in each changed
file that you understand the *call sites* and *invariants*, not just the hunk.
A change is only safe in the context of everything that calls it.

### 2. Route changed files to slices

Map each changed file to the relevant review slices using the table below, then
read **only** the reference files for slices that are actually in play. Most
diffs touch 2–4 slices. When unsure, include the slice — a false include costs a
subagent; a false exclude misses the bug.

| If the diff touches… | Review slices (read these references) |
|---|---|
| `snow/`, `simplex/`, `vms/proposervm/`, block accept/reject, preference, anything ordering-sensitive | `consensus-determinism.md` |
| `sync.*`, `go func`, channels, `atomic.*`, shutdown/Close, lock fields, `network/`, background loops | `concurrency.md` |
| `codec/`, `message/`, `*.canoto.go`, block/header marshaling, wire formats, persisted encodings, `upgrade/` gating | `codec-serialization.md` |
| `database/`, `x/merkledb`, `x/blockdb`, `x/archivedb`, versiondb/prefixdb, batches, iterators, state commit/rollback | `state-database.md` |
| `staking/`, `utils/crypto/`, `warp/`, BLS/secp256k1, signature verify, key handling, randomness | `crypto-staking.md` |
| arithmetic on `uint*`/`int*`, conversions/casts, `utils/math` (safemath), gas/fee math, allocation sizes, **anything under `vms/saevm/`** | `integer-math-safety.md` |
| `network/` peers/handshake/throttling, `message/` parsing, gossip, per-peer resource accounting | `networking-dos.md` |
| `vms/saevm/` (sae, cchain, gastime, gasprice, worstcase, hook, saexec, txpool), fee/balance/nonce, execution | `vm-gas-economics.md` |
| error returns, `panic`/`recover`, type assertions `x.(T)`, nil derefs, `context.*`, shutdown paths | `error-panic-safety.md` |
| any production code change (always assess its tests) | `test-rigor.md` |
| any diff (cross-cutting; cite by name) | `repo-conventions.md` |

`test-rigor.md` and `repo-conventions.md` apply to essentially every review.

### 3. Dispatch adversarial reviewers (parallel)

This repo and environment favor heavy parallelism. For each in-play slice,
dispatch a subagent **in the same message** (use the `Explore` or
`general-purpose` agent) with a prompt of the form:

> Adversarially review this diff for the **<slice>** slice of avalanchego. Read
> `~/.claude/skills/ava-adversarial-review/references/<slice>.md` and apply its
> checklist and red flags to the diff below. For each concern, try hard to
> construct a concrete failing scenario (input, interleaving, or sequence of
> blocks) that breaks an invariant — if you can't construct one, say so and
> lower confidence. Return findings as: title, file:line, severity
> (critical/high/medium/low), the concrete failure scenario, the repo rule or
> invariant violated, and a confidence 0–100. Verify claims against the actual
> code — do not invent file:line references.
>
> DIFF: <the diff, or the file list + instruction to read them>

Give each subagent the diff (or, for large diffs, the changed-file list and let
it read them). Tell each to ground every finding in the real code.

For a quick/small review you may run the slices inline instead of via subagents,
but always cover every in-play slice.

### 4. Adversarially verify findings

Do not pass findings through verbatim — subagents over-report. For each
**critical/high** finding, verify it against the real code (re-read the cited
lines; the scenario must actually be reachable). A finding survives only if you
can trace a concrete path to the bad outcome. Drop findings that depend on
invariants the surrounding code already guarantees, and say so. When two slices
report the same root cause, merge them.

Be honest about confidence. It is fine — and useful — to report a *plausible*
risk you couldn't fully confirm, clearly labeled as unverified, so the human can
judge. Do not manufacture certainty, and do not pad the report with nitpicks
that the linter would catch anyway (those go in a short "lint/CI will catch"
note, not the findings).

### 5. Report

Use this structure:

```
# Adversarial review: <branch/PR>
Scope: <files / diff range>. Slices reviewed: <list>.

## Verdict
<Safe to merge | Merge after addressing High+ | Do not merge> — one-line why.

## Critical  (consensus fork, fund loss, remote crash, state corruption)
### <title>  — `file:line`  [confidence N]
- Failure scenario: <concrete steps/input/interleaving>
- Invariant/rule violated: <name it — e.g. "saevm invariant G1", "G115", "deterministic iteration">
- Fix direction: <short>

## High / Medium / Low
<same shape, grouped>

## Test gaps
<missing error-path / fuzz / round-trip / determinism / property coverage>

## Lint/CI will catch (not counted as findings)
<one-liners: header, importas alias, forbidigo, etc.>

## Verification run
<commands the reviewer/author should run — from repo-conventions.md>
```

Order findings by severity, then confidence. If you found nothing in a slice,
say so explicitly — "reviewed X for Y, no issues" is a real result.

### 6. Address findings (only when the user wants fixes)

A review's deliverable is the report — many reviews end at step 5. But when the
user decides to act on the surviving (filtered, verified) findings, do **not**
jump straight to patches. The "Fix direction" lines in the report are pointers,
not designs: in this repo a fix can itself break consensus, drift an encoding,
or reintroduce an overflow, so the *how* deserves the same scrutiny as the bug.

Invoke the **`superpowers:brainstorming`** skill to turn the findings into an
agreed plan before any code changes. Feed it the verified findings as the
problem statement — for each one its failure scenario, the invariant violated,
and the fix direction — and let it explore approaches, surface trade-offs, and
get the user's sign-off on a design. This keeps fix selection collaborative and
prevents over-eager patching of findings the user may want to handle differently
(accept the risk, fix in a follow-up, redesign more broadly).

Scope what you hand to brainstorming to what the user wants fixed — usually the
Critical/High findings, or whichever subset they name. Brainstorming terminates
in `writing-plans`, which carries the work into implementation from there; you do
not need to design the patches yourself in this skill.

## The slices (reference index)

Each file is a self-contained adversarial checklist: what breaks, repo-specific
patterns/anti-patterns, probing questions, and red-flag code shapes.

- `references/consensus-determinism.md` — safety, liveness, non-determinism
- `references/concurrency.md` — races, deadlocks, lock ordering, goroutine leaks
- `references/codec-serialization.md` — wire/state compat, versioning, bounds
- `references/state-database.md` — atomicity, crash-consistency, iterators
- `references/crypto-staking.md` — signatures, BLS/secp256k1, key validation
- `references/integer-math-safety.md` — overflow, G115, safemath, the saevm bar
- `references/networking-dos.md` — remote-triggerable resource exhaustion
- `references/vm-gas-economics.md` — SAE value conservation, gas/fee, mempool↔consensus
- `references/error-panic-safety.md` — swallowed errors, reachable panics
- `references/test-rigor.md` — judging whether the tests are actually sufficient
- `references/repo-conventions.md` — citable lint rules, invariants, verify commands

## Guiding principle

The most dangerous changes in this repo are the ones where the code is *correct
for the case the author tested* and wrong for a case they didn't: a different
map ordering on another node, a peer sending a hostile length prefix, a uint
subtraction that underflows, an encoding that drifts one byte across an upgrade,
an error that's logged-and-ignored instead of halting. Spend your effort there.
