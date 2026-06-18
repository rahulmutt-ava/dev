# Slice: Cryptography, staking, signatures & key handling

Areas: `staking/` (TLS certs, parse/verify), `utils/crypto/` (secp256k1, bls),
`vms/platformvm/warp/` and other signature-verification paths, anywhere keys,
signatures, nonces, or randomness are handled.

## What can go wrong
- **Verification bypass**: an early `return nil`/success path that skips the
  actual crypto check; verifying the wrong thing; treating a parse error as
  "valid".
- **Missing public-key validation**: feeding attacker-supplied or stored bytes to
  aggregation/verification without subgroup/key validation. Note the BLS API has
  a *validated* path (`PublicKeyFromCompressedBytes` → `KeyValidate`) and an
  *unvalidated* fast path (`PublicKeyFromValidUncompressedBytes`) used on the
  assumption keys were validated at registration. If a diff uses the unvalidated
  path, confirm the upstream validation invariant actually holds.
- **Domain separation / replay**: a signed message missing network ID, chain ID,
  height/epoch, or nonce, so a signature is replayable across networks, chains,
  or validator sets. Warp binds NetworkID + SourceChainID into the signed bytes
  and selects the validator set by P-chain height — check new signed payloads do
  equivalent binding.
- **Malleability**: accepting non-canonical signatures (e.g. secp256k1 high-S).
- **Weak/incorrect randomness**: key/nonce generation not from `crypto/rand`;
  conversely, using nondeterministic randomness where determinism is required.
- **Nil deref after deserialize**: a key/sig parse that can return nil used
  without a nil check.
- **Weak key acceptance**: relaxing RSA/ECDSA cert constraints (bit length,
  exponent, modulus parity) that the parser currently enforces strictly.

## Repo patterns to check against
- secp256k1 verifies by **recovery + address compare**, enforces canonical
  (low-S) signatures and a fixed 65-byte length, and rejects compressed keys on
  the recovery path. It signs deterministically (RFC6979) — no nonce reuse.
- BLS distinguishes the signature ciphersuite from proof-of-possession; the
  validated vs unvalidated key constructors above are an explicit invariant.
- Warp `BitSetSignature.Verify` runs the full sequence (network ID check, bitset
  validate, filter validators, sum weight, quorum check, parse sig, aggregate
  keys, `bls.Verify`) with no early success — a new verifier should mirror that.
- Staking cert parsing enforces RSA bitlen ∈ {2048,4096}, exponent 65537, odd
  modulus. Loosening any of these is a downgrade.

## Adversarial questions
- Can verification return success without the underlying crypto call executing?
  Walk every return path.
- Are attacker-supplied keys/signatures validated before use? If the unvalidated
  BLS path is used, where exactly were those keys validated, and is that
  guaranteed for this caller?
- Is the signed message fully domain-separated (network, chain, height/epoch,
  nonce as applicable)? Can the same signature be replayed elsewhere?
- Are signatures required to be canonical? Is length/format checked first?
- Is key/nonce material from `crypto/rand`? Is determinism preserved where needed?
- Any new type assertion / deref on parsed crypto material without a nil/ok check?

## Red-flag shapes
- `if err != nil { return nil }` (treating failure as valid) in a verify path.
- `PublicKeyFromValidUncompressedBytes(...)` on freshly received / untrusted bytes.
- A new signed payload struct lacking network/chain/height binding.
- Relaxed cert constraints, or `==` byte comparison on secret material.
- `sig, _ := SignatureFromBytes(b)` with no length/format validation first.
