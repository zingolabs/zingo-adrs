# The rescan is deprecated in favor of a contractual, pure, and deterministic sync

## Status

accepted

Ruled in session 2026-11-22. The work is tracked in zingolib#3208.

[ADR-0015](0015-landing-in-dev-ships-the-wallet-file-format.md) and
[ADR-0061](0061-the-reorg-allowance-follows-zebra-to-1000-blocks.md) name
the rescan as a recovery. They stand until the removal this record
orders, and this record supersedes neither.

## Context

A rescan clears every piece of wallet data that sync derived from the
chain and then syncs again from the wallet birthday. zingolib offers it
as `LightClient::rescan`, and the code reaches for it in four places.

- A user runs it when the wallet shows a state the user does not
  believe.
- Truncation orders it. When a shard tree holds state above the
  truncation height and has no checkpoint at that height, the outcome
  is `PoolTruncation::RequiresRescan`, and the error tells the user
  that the wallet data is cleared and a rescan is required. ADR-0061
  widened the re-org allowance to keep more re-orgs away from that
  outcome.
- The documentation of the transparent address discovery settings
  recommends it after a change to those settings.
- A restore from seed forces it, in the words of ADR-0015.

Rescans are already rare. Each of the four uses is a remedy for a state
that sync could not explain or could not repair, and none of them is a
step of normal operation.

## Decision

The rescan is deprecated. A rescan is the clearing of all chain-derived
wallet data followed by a sync from the birthday.

1. No new code offers, orders, or recommends a rescan. That covers a
   return value, an error message, a log line, a command, and a
   documentation sentence.
2. `LightClient::rescan` and `LightClient::rescan_and_await` are marked
   deprecated, and they are removed when the last of the uses above is
   retired.
3. Each existing use is replaced by a repair bounded to the block range
   that a violated contract names. The repair derives that range again
   and leaves the rest of the wallet as it stands.
4. A restore from seed is the first sync of a new Wallet File. It is
   not a rescan, and this record does not deprecate it.

Sync is held to three properties, which make the rescan unnecessary.

- **Contractual.** Each step of sync states the condition it requires
  and the condition it leaves. A broken condition is a typed error that
  names the invariant and the block range.
- **Pure.** The chain-derived state of a wallet is a function of its
  keys, its birthday, and the chain. It does not depend on the history
  of Sync Sessions, on the order of batches, or on the order in which
  an indexer answers.
- **Deterministic.** The same keys, birthday, and chain produce the
  same state and the same requests.

## Why the rescan is the inferior remedy

A rescan repairs without a diagnosis. It destroys the wallet state that
is the evidence of the defect, and the defect stays in sync, unlocated,
to produce the same state again. A contract that fails names the
invariant and the range, so the defect can be found and the repair can
be bounded.

Under a pure and deterministic sync, a rescan computes a value the
wallet already holds. Where sync is correct, the rescan is redundant.
Where sync is incorrect, a rescan that happens to succeed proves that
the state depended on something other than the keys and the chain, and
it hides that dependence. A remedy that works only when the process is
impure cannot be the remedy of a pure process.

A rescan repeats every request the wallet has made. An indexer that
sees the same wallet ask twice can intersect the two sessions, and any
cover that differs between them falls away to leave the wallet's own
transactions. The Cover Group of zingolib#3184 pays for that
repetition: it must be a function of chain data alone, which costs a
download of up to 1,000 compact blocks for each area that holds a
wallet transaction. A wallet that requests each transaction once in the
life of its Wallet File leaves cheaper cover sound.

A rescan loses what the chain does not hold. ADR-0015 records that a
restore forfeits local transaction metadata, and the send path logs
that after a rescan the wallet will not display the sent-to addresses
that did not fit in the memo. A bounded repair keeps that data for
every transaction outside its range.

A rescan costs the whole history. It downloads and trial-decrypts
every block from the birthday to repair a fault that a contract would
confine to one range.

## Consequences

Each use of the rescan needs its bounded repair before the removal.
The truncation outcome `PoolTruncation::RequiresRescan` is the hardest
of them, because a shard tree with no checkpoint at the target height
has, today, no state to return to.

A bounded repair repeats the requests of its range, so it carries a
smaller form of the exposure described above. The design of each
repair states what an indexer learns from it.

A restore from seed still repeats the requests of the Wallet File it
replaces. This record leaves that exposure to the design of cover for
transaction requests.

The user loses a command that was a comfort. The typed error that
replaces it must say what is wrong and what the wallet did about it.
