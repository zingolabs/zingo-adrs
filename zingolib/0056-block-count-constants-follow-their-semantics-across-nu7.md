# Block-count constants follow their semantics across the NU7 block-spacing change

## Status

accepted

## Context

NU7 changes the target block spacing from 75 seconds to 25 seconds
([ZIP 218](https://zips.z.cash/zip-0218), deployed by
[ZIP 259](https://zips.z.cash/zip-0259)). Every quantity the wallet
denominates in blocks therefore means a third of the wall-clock time it
meant before, from the activation height onward. ZIP 218's section
"Block-count-based constants" gives the rule for implementations: scale a
constant whose meaning is a wall-clock margin, and leave alone a constant
whose meaning is intrinsically a number of blocks. The consensus-level
constants are not in question: coinbase maturity stays at 100 blocks.

This record covers the constants whose meaning NU7 changes. The wallet
carries three kinds.

**The transaction expiry delta.** A transaction expires a fixed number of
blocks past its target height. The default is 40, which
[ZIP 203](https://zips.z.cash/zip-0203) sized as about 50 minutes at the
75-second spacing, and which ZIP 203 and ZIP 218 both now say should
become 120 from NU7 activation to keep that duration. The builder in
zcash_primitives still defaults to 40 (`DEFAULT_TX_EXPIRY_DELTA`, through
the 0.31.0-pre.1 of the NU7 cohort), and zingolib took that default at
every build site and derived the offline-signing cap in
`lightclient/send.rs` from it. One build site is different by design: the
backend gives a step shaped like a canonical ZIP 318 crossing the ZIP's
rolling expiry, which every crossing in a modulus period shares, and
refuses any other expiry for it.

**The ZIP 318 migration schedule.** The Orchard-to-Ironwood migration draws
its delays, its anchor-age cap and its expiry window from constants that
[ZIP 318](https://zips.z.cash/zip-0318) denominates in blocks on purpose:
bucket boundaries, anchors and expiry heights are all block heights, and
block height is what an on-chain observer measures. The ZIP sizes them in
wall-clock terms at 75 seconds (a 30-day expiry modulus, a 90-minute mean
transfer delay) and says nothing about NU7. The wallet takes them from
zcash_protocol and zcash_pool_migration, and its ZIP 318 tripwires fail
when they move.

**Confirmation and scan-shape counts.** The default of 3 confirmations,
which is also the anchor depth, the scan-area width the engine punches
around a found note, and the gap limits are counts of blocks or of
addresses whose meaning does not involve time. ZIP 218 says explicitly that
the anchor depth should stay at 3 after activation. The engine's own
timers, the ten-second new-block check and the mempool settle windows, are
already wall-clock.

One block-denominated constant is deliberately outside this record. The
sync engine's reorg allowance, `pepper_sync::sync::MAX_REORG_ALLOWANCE`,
documents itself as a mirror of Zebra's finalization boundary at 100
blocks, and Zebra raised that boundary from 99 to 1000 in release 5.2.0
and keeps 1000 through NU7. The mirror is stale on its own terms rather
than because of NU7, the allowance sizes what the engine retains in order
to rewind, and the cost of raising it has not been measured. It gets a
record of its own once it has.

## Decision

Each block-denominated constant is classified by what it measures, and
only the wall-clock ones change.

The transaction expiry delta becomes a function of the transaction's
target height: 40 blocks below the NU7 activation height, 120 blocks at
and above it, and 40 on a chain that never activates NU7. One function in
zingolib owns the rule, and every build site passes the expiry it computes
to the builder instead of taking the builder's default, with the
offline-signing cap derived from the same function. A proposal the backend
builds as a canonical ZIP 318 crossing is the exception: it leaves the
expiry to the backend, which applies the ZIP's rolling expiry, as the
wallet did for every proposal before. When zcash_primitives makes its own
default activation-aware the function collapses onto it.

The ZIP 318 constants stay exactly what the ZIP says. The wallet's
migration schedule is private only because every wallet draws from the
same distributions, so a wallet that rescaled them on its own would stand
out. The wall-clock drift at 25-second blocks is raised with the ZIP's
authors as an upstream question, and the tripwires keep adjudicating each
release of the canonical crate as they do today.

The confirmation count, the anchor depth, the scan-area width, the gap
limits and coinbase maturity do not change.

## Consequences

Sends made above the NU7 activation keep about 50 minutes to be mined, as
they have since Blossom, instead of 17. A send built below the activation
with a target above it carries the post-activation delta, since the rule
reads the target height and the network validates the transaction under
the branch of that height.

The migration schedule becomes three times faster in wall-clock terms
after NU7 while its block-denominated privacy properties stay as specified.
Changing that is the ZIP's decision, not the wallet's.

The in-process test era runs with NU7 active, so the expiry rule has a
boundary to be tested across. The regtest tier keeps NU7 off until an
indexer serves it, so its coverage of the rule waits on that.

The reorg allowance is decided separately, after the retention cost of a
deeper allowance has been measured on real wallets.
