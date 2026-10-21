# A session attaches to one indexer, the zingolabs endpoint by default

## Status

proposed

Ruled in session 2026-10-13, pending review.

Supersedes [ADR-0022](0022-broadcast-witness-never-the-sync-indexer.md),
[ADR-0034](0034-server-selection-is-a-mixnet-liveness-sweep.md),
[ADR-0040](0040-sends-escalation-is-a-hedged-race-of-full-paths.md),
[ADR-0042](0042-a-destination-reservation-levels-concurrent-transmissions.md),
and [ADR-0050](0050-indexers-are-classified-by-role-trust-and-location.md).
Amends [ADR-0011](0011-nym-mixnet-transmission.md) (witness rotation),
[ADR-0037](0037-broadcast-means-many-recipients.md) (the decoupled
migration endpoint), and
[ADR-0045](0045-boot-proves-four-exits-and-assigns-them-by-role.md) (the
number of exits boot proves). Reaffirms
[ADR-0001](0001-offline-by-default.md).

## Context

Since ADR 0011 the wallet has treated the indexer as the adversary and
spread a session's indexer traffic across operators so that no single
one could assemble a picture of it. The mechanisms grew in layers: a
sync indexer drawn by operator from a surveyed census (ADR 0034), a
Transmission Destination drawn under the rule that the sync operator
never receives a Transmission (ADR 0022), a hedged race that widened a
stalled Transmission across further Destinations (ADR 0040), a
reservation ledger that leveled concurrent Transmissions across
operators (ADR 0042), a role, trust, and location classification with a
draw rule that reconciled self-hosters, testnet, and nakednet (ADR
0050), and a separate migration endpoint so the sync indexer could not
correlate migration parts with sync (ADR 0037). Those layers hold about
four thousand lines across the destination, sweep, discovery, and
nakednet-selection modules.

The census that fed every draw rotted under them. ADR 0029 found every
leaderboard entry dead in August 2026, and the mainnet census carried
twenty-one foreign operators whose liveness nothing maintained.

zingolabs now operates its own indexer, `mainnet.zainod.zingolabs.dev`,
reachable over https on port 443 with a webpki-chained certificate, so
it passes the mixnet exit policy and ADR 0021 unchanged.

## Decision

A session holds exactly one indexer, the Session Indexer, and it serves
every operation on every transport: sync, every Transmission, migration
parts, and the boot health round trip. The library ships one Default
Indexer per chain, operated by zingolabs, which becomes the Session
Indexer when a consumer goes online without naming one. Mainnet's is
`https://mainnet.zainod.zingolabs.dev:443`. Testnet has none until a
zingolabs testnet host exists; a testnet consumer names a URI or
receives a typed refusal. A consumer may name any URI in place of the
default, so a self-hoster still points the wallet at their own node.

Over-mixnet indexer selection is retired. One indexer observes all of a
session's indexer traffic, and that correlation is accepted: the
privacy that rotation bought did not repay the logic it cost. The
mixnet still conceals the client's IP from that indexer.

Nothing of the retired machinery survives. The role, trust, and
location classification goes, because one indexer needs no
classification. The migration transmission endpoint goes, because the
decoupling it provided is correlation this record accepts. The
Server-Selection Sweep collapses into the attach health round trip: one
`GetLightdInfo` against the Session Indexer, which checks the chain and
proves the mixnet path end to end. The hedged race collapses into
retries against one endpoint under the existing resilience policy.
Health becomes one indexer's up-or-down, not a draw filter. The census
and the leaderboard are deleted and replaced by a table of Default
Indexers keyed by chain.

Boot proves three exits instead of four. The sweep's exit had no job
left, so the roles are PriceFetch, IndexerClient, and spare, assigned in
the order they confirm, and the health round trip is the IndexerClient
conduit's first act after its Sentinel. The spare's failover is
unchanged.

Every surface that takes an indexer takes a typed three-way choice:
Offline, the chain's Default Indexer, or a named URI. The Binding
Layer's IndexerConnection record carries it as a uniffi enum in place of
the `server_uri` string whose empty value meant Offline. The CLI's
`--server` names a URI, consent without `--server` means Default on
every build, and `change_server` takes the same three-way choice, where
Offline clears the Session Indexer through `go_offline`.

The client still starts offline. ADR 0001's reasoning stands: the
Default Indexer is the endpoint the wallet proposes once the consumer
consents, never one it dials unasked.

## Considered options

Keep the draw and mark the zingolabs operator trusted by default, so the
trusted branch of ADR 0050 sends to the sync indexer alone. Rejected:
it preserves every layer of the machinery to serve a registry of one
entry.

Keep the foreign census operators as untrusted Destinations beside the
default sync indexer, so the exclusion still has somewhere to draw.
Rejected: it keeps a census ADR 0029 found rotting, and it keeps the
complexity the ruling exists to remove.

Let the chain alone pick the endpoint, with no consumer-facing URI
outside a test feature. Rejected: the self-hoster keeps their own node,
and the tests attach to loopback indexers through the same surface.

Keep `testnet.zec.rocks` as testnet's Default Indexer until a zingolabs
host replaces it. Rejected: a foreign built-in contradicts the rule the
day it ships, and a typed refusal is clearer than an exception.

## Consequences

The destination, sweep, live-indexer-discovery, and nakednet-selection
modules are deleted, with their tests and the mock-chain tests that
drove the mainnet draw over three mock indexers. The glossary retires
Destination Selection, Destination Rotation, Destination Indexer, Sync
Indexer Selection, the Server-Selection Sweep, Indexer classification,
the Destination Server set, and the IndexerSweep exit, and adds Session
Indexer and Default Indexer.

The migration-part path and the Transmission path now differ only in
the txid echo check.

A consumer that previously relied on the broadcast rule to keep a send
away from its sync indexer no longer has that property on any
transport. A consumer that wants a different indexer for any operation
names it as the Session Indexer and accepts that it then serves every
operation.

zingo-mobile's static server list and its settings screen become a
consumer of the Default Indexer table plus a custom-URI entry; the
region keys the census carried for that screen have no replacement.
