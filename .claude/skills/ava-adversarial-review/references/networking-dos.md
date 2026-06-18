# Slice: Networking, P2P message handling & DoS resistance

Areas: `network/` (peers, handshake, routing, `throttling/`), `message/` parsing,
`network/p2p/gossip`, per-peer resource accounting. Threat model: peers are
potentially **Byzantine**; a remote attacker should not be able to OOM, panic,
or CPU-exhaust the node, nor amplify their input.

## What can go wrong
- **Unbounded allocation from peer input**: `make([]T, count)` where `count`
  comes from a decoded message (e.g. a peer-list / repeated field) with no
  per-message cap; a large `bytes` field; thousands of cheap entries that each
  trigger expensive work (cert parse, signature verify).
- **Missing/incorrect throttling**: acquiring resources *after* reading instead of
  before; releasing a quota more than once or not at all on error paths; per-call
  rather than per-peer accounting.
- **Slow-loris**: reading a length prefix then blocking on the body with a
  deadline that resets per read segment, letting a peer trickle bytes while
  holding quota.
- **Amplification**: gossip that re-broadcasts attacker-supplied items to N peers
  with no per-peer bound, multiplying input by the fan-out.
- **Remote-triggerable panic**: any unchecked index/assertion/`make` on a path
  reachable from a raw inbound message.
- **Handshake abuse**: re-doing expensive validation on repeated/duplicate
  handshakes or reconnects; trusting peer-claimed data (IP, bloom filter, subnet
  list) without bounding cost.

## Repo patterns to check against
- Inbound wire messages are capped (`DefaultMaxMessageSize`, 2 MiB) and the
  length is validated before allocation.
- Inbound bytes use **validator-weighted** quotas plus a capped at-large pool, so
  a non-validator/sybil peer is bounded regardless of how many nodes it runs.
  A new inbound path should hook into this throttling, not bypass it.
- The message-handling contract calls `onFinishedHandling()` exactly once —
  verify new error paths don't skip it or double-release.
- Deadlines are bounded (`deadline = min(claimed, maxTimeout)`); a peer can't
  request an arbitrarily long deadline.
- Outbound throttling drops rather than blocks.

## Adversarial questions
- Is every length/count from a message validated against a cap *before* it sizes
  an allocation or a loop, or triggers per-element expensive work?
- Is resource quota acquired before the network read, and released exactly once
  on every path (success, parse error, context cancel)?
- Are read/write deadlines enforced such that a peer can't trickle bytes
  indefinitely while holding a slot?
- Does any gossip/relay re-broadcast peer input without a per-peer bound?
- Is any new inbound code path reachable by an unauthenticated/Byzantine peer?
  What's the worst single message? The worst sustained stream?
- Does repeated handshake/reconnect let a peer force repeated expensive work?

## Red-flag shapes
- `make([]T, msg.GetSomeCount())` / `make([]byte, peerLen)` with no cap check.
- Throttler `Acquire` after `io.ReadFull`, or a return path that skips release.
- A `for _, item := range msg.Items` doing cert parse / sig verify with no bound
  on `len(msg.Items)`.
- Gossip handler adding every received item and queuing all for rebroadcast.
- A new `Handle*` method that indexes/asserts on fields without validating them.
