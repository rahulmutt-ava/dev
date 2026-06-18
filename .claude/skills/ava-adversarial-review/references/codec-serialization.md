# Slice: Serialization, codec & wire/state backward-compatibility

Areas: `codec/` (linearcodec, reflectcodec, manager), `message/`, `*.canoto.go`
generated types, block/header marshaling, `vms/*/block`, `upgrade/` gating, and
any encoding that is hashed, signed, sent on the wire, or persisted.

## Why this is dangerous
Encoding is consensus- and storage-critical. If new nodes encode the same
semantic value into different bytes than old nodes, blocks hash differently and
the network **forks**. If an encoding changes without upgrade gating, persisted
state becomes unreadable on restart. Attacker-controlled length fields are a
classic OOM/DoS vector.

## What can go wrong
- **Encoding drift across versions**: reordering struct fields, changing a
  field's type/width, inserting a field mid-struct (vs append-only), or changing
  embedding/flattening — all change the byte layout and break old blocks/state.
- **Missing upgrade gate**: a new field marshaled unconditionally instead of
  behind `Upgrades.IsXActivated(blockTimestamp)`, so pre-upgrade and post-upgrade
  nodes disagree. (And the gate must use the block timestamp, not `time.Now()`.)
- **Codec version mismatch**: marshal path and unmarshal path disagree on
  version; a new concrete type implementing a registered interface not added via
  `RegisterType`, so it silently mis-decodes.
- **Unbounded allocation**: a length/count field read from untrusted bytes used
  to size `make(...)` without a max bound — single message OOM.
- **Nondeterministic encoding**: maps serialized without sorted keys; reliance on
  reflection field order; any nondeterminism makes the same value hash two ways.
- **Round-trip failure**: `unmarshal → marshal` not byte-identical (default
  values, trimmed slices, nil vs empty), so persisted bytes mutate on rewrite.
- **canoto pitfalls**: reused/decreased field numbers; regenerating not committed;
  generated code out of sync with the canoto library version.

## Repo patterns to check against
- `codec.Manager` prefixes a `uint16` version; codecs are registered per version
  with a single `CodecVersion` const and all `RegisterType` calls in `init()`.
- reflectcodec bounds slices at `MaxInt32` and enforces **sorted map keys** on
  both marshal and unmarshal — new manual encoders must preserve canonicalization.
- The manager enforces a max message size, but that's the envelope; per-field
  bounds inside a struct still matter.
- Upgrade-gated fields branch on the block/parent timestamp via the `upgrade/`
  config. Backward compat is verified with round-trip/fuzz tests and golden bytes.
- This branch's own history (`settled block codec`) shows the pattern: new header
  fields are added behind a marker and codec round-trips are integration-tested.

## Adversarial questions
- Did any existing serialized field move, change type/width, or get inserted
  (not appended)? Can a node from the previous release still parse old bytes?
- Is every new serialized field gated on an upgrade activation (by block
  timestamp)? Does the parser handle both pre- and post-upgrade encodings?
- Is there a round-trip test (`unmarshal→marshal` byte-identical) and a fuzz
  test? For a persisted/wire type, are there golden vectors from the prior format?
- Is every new interface implementation registered? Does the codec version match
  on both marshal and unmarshal?
- Is every length/count from untrusted input bounded before allocation?
- Do any maps get encoded with sorted keys? Any reliance on field/iteration order?

## Red-flag shapes
- A struct field reordered/retyped, or a new field not at the end.
- `make([]T, n)` / `make(map[..], n)` where `n` comes from decoded bytes, no cap.
- New `RegisterType` missing for a new concrete type, or a bumped codec version
  used in only one of marshal/unmarshal.
- A new serialized field with no `if upgrades.IsXActivated(...)` guard.
- No `Fuzz*`/round-trip test accompanying a codec/block-format change.
- Regenerated `*.canoto.go` / `*.pb.go` not committed (CI fails on dirty tree).
