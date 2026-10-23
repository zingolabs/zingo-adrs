# A wallet loader returns an outcome and one owner settles it

Date: 2026-10-12

## Status

proposed

Ruled in a grilling session on 2026-10-12 over the review of zingo-mobile
PR #1604 "feat(ui): route init connection failures to the wallet error
screen". The record moves to `accepted` when the stacked pull request that
carries the change merges.

## Context

The wallet error screen runs a failed step again on Retry. The three
loaders, which open, create and restore a wallet, each ended that retry
by writing `retrying: false` and `actionButtonsDisabled: false` into the
screen state themselves, on every exit they took. Two exits forgot. A
create that failed with an alert left the flag set, so the next first
failure shook the icon and reported a second failure. A restore whose key
was rejected before the native call returned without clearing anything,
and the error screen stayed busy with every link disabled.

Each new exit in a loader is a place to forget the flags again. A flag the
loaders own cannot be checked by the compiler.

## Decision

A loader returns one `LoadOutcome` value and writes no screen state on
the way out. The union is closed: `loaded`, `error`, `alert`, `rejected`
and `offline`. One method, `settle`, applies an outcome with an exhaustive
switch, and it is the only writer of the busy flags, the error, and the
screen for these paths. The retry owner sets the busy state, awaits the
loader, waits out the hold for an in-place retry, and settles. The loaders
cannot forget an exit because they have none to forget, and a sixth variant
fails to compile until `settle` gains an arm for it.

Two pure functions feed the loaders. `nativeReply` classifies the native
result, including the JSON parse and the chain mismatch code, once for all
three loaders. `restoreInput` validates the key, the chain prefix and the
birthday, and returns the restore arguments or a rejection. Both are
tested with plain values.

Whether a settlement is a second failure travels with the retry, as a pair
of the `Retry` and its outcome, in place of a boolean read from state.

## Considered options

A union over the error state, with `retrying` folded into a variant of
`walletError`, was rejected. The loaders would still write state on exit,
so a forgotten exit remains representable.

A boolean `again` argument to `settle` was rejected for the retry pair.
Every first-attempt caller would pass a bare `false`, and nothing ties the
flag to the retry it describes.

## Consequences

A loader that needs the screen to change mid-flight, such as moving to the
progress screen, still does so, because that state is not part of the
outcome. Only the end of the attempt is reserved to `settle`.

The hold applies to `open` and `switch` retries alone. A create or restore
retry moves to the progress screen, where a hold would keep a failed screen
visible.

A rejected restore from the error screen now returns to the origin screen
that opened the form, where before it stayed on the error screen.
