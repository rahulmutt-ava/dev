# Slice: State management & database integrity

Areas: `database/` (memdb/leveldb/pebbledb, versiondb, prefixdb, linkeddb,
corruptabledb, batch), `x/merkledb`, `x/blockdb`, `x/archivedb`, and VM state
commit/rollback paths.

## Why this is dangerous
State must be atomic and crash-consistent, and identical across nodes. A
non-atomic multi-key update, a swallowed write error, or a partial commit on
crash produces divergent or corrupt state — which surfaces as a fork or a node
that can't restart.

## What can go wrong
- **Non-atomic multi-key update**: related keys written via separate `Put`s or
  separate batches instead of one batch; a crash or mid-loop error leaves a
  partial update (e.g. a linked-list node update that touches prev/cur/next
  without a single batch).
- **Swallowed DB error**: a `Put/Delete/Write/Get` error ignored or logged-and-
  continued, so the code believes a write happened that didn't.
- **`errors.Join`-style "do everything then return"**: collecting errors from
  several write steps but still committing — a failed early step leaves partial
  in-memory state that gets flushed.
- **versiondb abort gaps**: not `defer Abort()` on every `Commit`/`CommitBatch`
  path, so aborted changes leak into the next commit; writing a batch after abort.
- **Iterator misuse**: missing `defer it.Release()` (FD/lock leak); using an
  iterator after modifying or closing its DB; reading a stale in-memory snapshot
  taken outside the lock.
- **Prefix collision**: inconsistent prefixing (Put with prefix, Get without);
  nested prefixdb whose concatenation can alias another namespace.
- **State root / ordering**: recording history or updating the root *before* the
  value batch is durably written; intermediate and value nodes flushed in
  separate batches with no crash marker, leaving orphans.

## Repo patterns to check against
- `versiondb.Commit` is `defer s.Abort()` then `CommitBatch().Write()` — the
  abort runs even if the write fails. New commit paths should keep this shape.
- `merkledb` uses a clean-shutdown marker: set "dirty" before writes, "clean"
  only after everything is durable; on startup an unclean flag triggers rebuild.
  The root/rootID are updated only *after* the value batch writes successfully.
- `corruptabledb` fails closed: after any unexpected error it rejects all future
  ops to prevent cascading corruption. Don't bypass that contract.
- `prefixdb` pools prefixed keys and releases them; iterators strip the prefix.
- Iterators are released via `defer it.Release()` right after creation.

## Adversarial questions
- Are all keys belonging to one logical state change written in a single batch
  (all-or-nothing)? What state remains if the process dies mid-update?
- Is every DB op's error checked and propagated (not joined-then-committed)?
- Does every `Commit`/`CommitBatch` have a matching `Abort` on all paths?
- Does every iterator have `defer Release()`? Is the DB modified while iterating?
  Is any in-memory snapshot taken under the lock that protects it?
- Is the state root / history recorded only after the value write is durable?
- Are prefixes applied consistently? Can nested prefixes alias another namespace?
- Is there a crash marker (or single atomic batch) covering multi-step writes?

## Red-flag shapes
- Two+ `Put`/`Delete` for related keys not in one `batch`, then `batch.Write()`.
- `_ = db.Put(...)` / ignoring `Write()`'s error; `err` from a write only logged.
- `errors.Join(writeA(), writeB(), ...)` where a partial failure still commits.
- `NewIterator...()` with no `defer Release()`; iterator used post-`Close`.
- `root = newRoot` / `history.record(...)` before the corresponding `batch.Write()`.
- `CommitBatch()` without a `defer Abort()`.
