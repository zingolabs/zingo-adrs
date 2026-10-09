# Block-count constants follow their semantics across the NU7 block-spacing change

## Status

proposed

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

The wallet carries four kinds of block-denominated quantity.

**The transaction expiry delta.** A transaction expires a fixed number of
blocks past its target height. The default is 40, which
[ZIP 203](https://zips.z.cash/zip-0203) sized as about 50 minutes at the
75-second spacing, and which ZIP 203 and ZIP 218 both now say should
become 120 from NU7 activation to keep that duration. The builder in
zcash_primitives still defaults to 40 (`DEFAULT_TX_EXPIRY_DELTA` in
0.31.0-pre.1), and zingolib takes that default for every send and derives
the offline-signing cap in `lightclient/send.rs` from it.

**The reorg allowance.** `pepper_sync::sync::MAX_REORG_ALLOWANCE` is the
deepest reorganisation the sync engine follows. Below it the engine
rewinds. Past it the engine clears the wallet's scanned data and reports a
chain verification error. It also sizes what the engine retains in order
to rewind: the shard tree keeps one checkpoint per block across the
allowance, the wallet keeps that many recent blocks and their nullifiers,
transparent discovery re-searches that many blocks below the tip at the
start of every session, and zingolib treats a transaction confirmed deeper
than the allowance as settled. Its doc comment says it mirrors Zebra's
finalization boundary at 100 blocks. Zebra raised that boundary,
`MAX_BLOCK_REORG_HEIGHT`, from 99 to 1000 in release 5.2.0 as a defence
against sustained consensus splits, and Zebra 7.0.0-rc.0 keeps 1000
through NU7, documenting it as about 20.8 hours before activation and 6.9
hours after. ZIP 218's recommendation of 600 scales the 99 Zebra no longer
uses. The wallet's mirror is therefore stale on its own terms: an indexer
backed by Zebra can serve a reorganisation up to ten times deeper than the
wallet can follow.

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

## Decision

Each block-denominated constant is classified by what it measures, and
only the wall-clock ones change.

The transaction expiry delta becomes a function of the consensus branch
of the transaction's target height: 40 blocks below NU7, 120 blocks from
NU7 activation. One function in zingolib owns the rule, and every send passes
the expiry it computes to the builder instead of taking the builder's
default, and the offline-signing cap derives from the same function.
When zcash_primitives makes its own default branch-aware the function
collapses onto it.

The reorg allowance mirrors the validator's actual finalization boundary,
1000 blocks, and its documentation names the Zebra release the value comes
from. It is not conditional on NU7, because Zebra's boundary is not. The
alternative of ZIP 218's 600 is rejected: it would leave reorganisations
between 600 and 1000 blocks, which Zebra follows and an indexer serves,
unrecoverable for the wallet without a full rescan. Keeping 100 is
rejected for the same reason, with a far shorter margin: 42 minutes of
blocks after NU7.

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
reads the target height.

The wallet retains ten times as many shard-tree checkpoints, recent blocks
and recent nullifiers as before, and transparent discovery searches ten
times as many blocks below the tip when a session starts. The retained
items are small, a checkpoint and a block header each, and the search is a
bounded range per address, but the cost is real and is measured when the
constant moves. A wallet file written with the smaller window loads
unchanged, since the checkpoint window is a ceiling.

A reorganisation deeper than 100 blocks no longer forces a full rescan.
One deeper than 1000 still does, and that is Zebra's limit as well.

The migration schedule becomes three times faster in wall-clock terms
after NU7 while its block-denominated privacy properties stay as specified.
Changing that is the ZIP's decision, not the wallet's.

The regtest tier and the in-process era both run with NU7 active, so the
expiry rule has a boundary to be tested across. A test holds the reorg
allowance at the documented Zebra value, so a future divergence is a
deliberate edit of both.
