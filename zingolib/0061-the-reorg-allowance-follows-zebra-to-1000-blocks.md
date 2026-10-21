# The re-org allowance follows zebra to 1000 blocks

## Status

proposed

## Context

`pepper_sync::sync::MAX_REORG_ALLOWANCE` is the deepest chain
reorganization the wallet tolerates and the repository's single source of
truth for that depth. Its documentation pins it to zebra's
`MAX_BLOCK_REORG_HEIGHT`, the depth below the tip at which zebra finalizes
a block and follows no fork beneath it. Zebra raised that limit from 100
to 1000 in release 5.2.0 and keeps it at 1000 through v7.0.0-rc.0, the
validator this workspace runs. The wallet stayed at 100, so the mirror
was stale. [ADR 0056](0056-block-count-constants-follow-their-semantics-across-nu7.md)
left the constant alone because the cost of raising it had not been
measured, and promised it a record once it had.
[zingolib#2537](https://github.com/zingolabs/zingolib/issues/2537) argued
for keeping 100 and re-anchoring the documentation to
`zcash_client_backend`'s private `PRUNING_DEPTH`, which is also 100.
[zingolib#2454](https://github.com/zingolabs/zingolib/issues/2454) asks
for the shard tree checkpoint retention to derive from the allowance,
which it does.

The allowance sizes five things. A fork within it is truncated and the
blocks above it scanned again, and a fork deeper than it clears the
wallet's chain data and returns the chain error, after which the next
session rescans from the birthday. Each pool's shard tree keeps a rolling
window of one checkpoint more than the allowance, which the truncation
rolls back to and a spend anchors in. The wallet keeps the wallet blocks
of that many scanned blocks, whose hashes the re-org detection compares.
Transparent address discovery searches that many blocks below the last
known chain height at the start of every session. A wallet whose last
known chain height is more than the allowance above the server's returns
the chain error.

The measurement exposed a cost that the raise magnified tenfold rather
than created. The scan placed a checkpoint window on every load of a
scan task, hanging from the load's own last block, and the shard tree
update did the same from the wallet's highest scanned height at
processing time. Loads land in ascending order inside a task, so every
load above the highest scanned height placed a full window that the next
load's inserts pruned, re-hashing the folded leaves on the way. At 101
checkpoints that churn was cheap. At 1001 every block of a load was a
checkpoint, and the sync rate over 124,000 mainnet blocks fell by 64%.
The two sides could also disagree whenever another task moved the
wallet's highest scanned height between a load's scan and its update.

A re-org is measured from the top of the chain, so the window is only
needed there. A session stopped between the loads of a task still needs
to spend and to anchor a ZIP 318 crossing, and zingolib anchors a spend
at the lesser of the newest checkpoint and the tip less the confirmations,
so a stopped wallet needs its newest checkpoint to be recent rather than
a checkpoint at a particular depth.

The cost was measured with `makers sync-ab`, which for this record gained
peak resident memory and wallet file readings beside the rate and a run
to completion ([zingolib#3041](https://github.com/zingolabs/zingolib/pull/3041)).
Both arms scan the same window of the same live chain with a fundless
wallet, interleaved, so the comparison holds the indexer, the seed, the
window and the build fixed and varies the commit alone.

## Decision

`MAX_REORG_ALLOWANCE` is 1000, documented as the mirror of
`zebra_chain::parameters::constants::MAX_BLOCK_REORG_HEIGHT`, with the
rolling checkpoint window, the retained wallet blocks, the transparent
discovery depth and the chain error bound following it as they did. The
wallet follows the validator it syncs against rather than
`zcash_client_backend`'s store policy, because a re-org the validator
performs and the wallet cannot follow costs every affected wallet a
rescan from its birthday.

The checkpoints are placed so that the raise costs nothing in rate. Each
load of a scan task places three things, on the scan side and the shard
tree update side alike, through one function. The rolling window hangs
from the session's chain height, so only the loads inside the top 1001
blocks of the chain place it, whatever task they belong to. The load's
own last block is placed, so a session stopped after the load anchors a
spend there. The anchor retention boundaries are placed as of the load's
last block, or the wallet's highest scanned height when that is higher,
so such a session anchors a crossing too. A load's last block without an
output of a pool copies the
checkpoint below or fetches the tree state once, as the window's first
block always did. The task carries the chain height and the highest
scanned height from its selection, every load carries them to the update,
and `update_shard_trees` takes both in place of reading the wallet at
processing time, so the two sides cannot drift apart.

A wallet stopped between the loads of a task that then meets a fork
inside the scanned part of that task truncates cleanly only where the
fork sits at a load's last block or a boundary, and otherwise takes the
clear-and-rescan path. That case is accepted as rare twice over, and a
planner that rolls back to the newest checkpoint below the fork and
rescans from there is the follow-up that closes it.

## Considered options

Keeping 100 and re-anchoring the documentation to `PRUNING_DEPTH`, as
zingolib#2537 proposed, costs nothing today. It leaves every wallet one
re-org between 100 and 1000 blocks deep away from a full rescan, on a
validator that follows such a re-org without comment.

Raising the allowance with the checkpoint placement as it was measured
64% slower at the tip, which is the cost this record was written to
weigh.

Hanging the window from the task's last block alone scans the 49,000
blocks of one diagnostic window in 16 seconds against the base's 31, but
leaves a session stopped between loads with no checkpoint in the part it
scanned, so it cannot spend until a later session completes the task,
and a fork meeting it rescans. Wallets on phones are stopped mid-sync
constantly, so this was rejected.

The chosen placement scans the same window in 21 seconds, the difference
being the tree-state fetches for last blocks that carry no output of a
pool, mostly ironwood, and keeps a stopped session able to spend.

## Consequences

Measured on mainnet against dev, which also lacked
[zingolib#2984](https://github.com/zingolabs/zingolib/pull/2984), so the
rate figure carries both changes. A run to completion over 124,000
blocks and 1,192,000 outputs, two sessions per arm with zero drift:

| | dev (100) | allowance 1000 | change |
|---|---|---|---|
| sync rate (outputs/s) | 8,650 | 29,321 | +239% |
| peak RSS (MiB) | 182 | 196 | +7.7% |
| wallet file (KiB) | 137 | 403 | +194% |

A run bounded to 300 seconds over the nightly's historic window read
9,985 against 9,984 outputs per second and 718 against 715 MiB, since the
historic ranges below the window are untouched either way. Testnet, where
blocks are sparse and per-block work dominates, gave 5,652 against 12,632
outputs per second to completion, 147 against 144 MiB, and 52 against
208 KiB, with the bounded run level to the unit.

The wallet file grows by the retained blocks and checkpoints, about
270 KiB on a fundless wallet. Memory moves within the run-to-run spread.
Transparent address discovery asks the server for ten times the blocks
at the start of a session. A wallet more than 1000 blocks above the
server's chain height returns the chain error where the bound was 100.

The pinned ZIP 318 boundaries, six intervals of 144 blocks, now fall
inside the rolling window, so the pins keep nothing the window would
prune. The mechanism stays, since it would again if the allowance were
lowered, and the total `MAX_SHARDTREE_CHECKPOINTS` counts up to six more
than the distinct heights a tree holds.

The mock chain can mine blocks of undecryptable outputs, so a scan task
splits into loads and a test stops a session between them, spends from
the last load's checkpoint and completes the task in the next session,
in about seven seconds. The mock test that mines past the allowance went
from nine seconds to fifteen.

The raise lands in [zingolib#3040](https://github.com/zingolabs/zingolib/pull/3040),
which closes zingolib#2537 and answers zingolib#2454. The 150-block cache
margin that zingolib#2537 also named belonged to the retired regtest
tier and needs no reconciling.
