# The app holds no consensus parameter and renders what zingolib reports

Date: 2026-10-20

## Status

proposed

Ruled in a grilling session on 2026-10-20 over the review of zingo-mobile
PR #1703 "fix: restore wallets with birthday 0 from Sapling activation".
The record moves to `accepted` when zingo-mobile PR #1704, which deletes
the activation tables, merges against a Binding Layer that reports
`birthday_adjusted_from`.

## Context

The Sapling activation height decides the lowest birthday a wallet can
have. zingolib holds it as a chain parameter and refuses a lower birthday
on restore. The app held two copies of the height, one in the import form
as a gate on the typed height, and one in the restore loader. The review
of PR #1703 found the two copies giving different answers for the same
height. The form refused 419199 on mainnet while the loader raised the
same height from a saved entry without a word. A third copy was on its way
in the stacked loader refactor, where `restoreInput` rejects a low height.

Both copies mapped regtest to 1 and the offline server to 1. The Binding
Layer parses the regtest activation schedule from the launched node and is
the only authority on it. Offline there is no chain, and a copy can name
no height for it.

The server info the app already receives carries
`sapling_activation_height`, and nothing in the app read it.

## Decision

zingolib owns the Sapling activation height, and no app layer copies it.
The app passes the birthday it was given to the Binding Layer, 0 when the
field is empty, and shows the birthday zingolib reports.

zingolib raises a birthday below activation on restore and on wallet file
read. It keeps the given height in memory for the process lifetime, beside
the height in use, and the seed and viewing-key reporters emit it as
`birthday_adjusted_from`. The saved wallet carries the raised height, and
the next open reports no adjustment.

The app reads the field in one place, the backend's birthday fetch that
runs after every wallet load. It shows one notice when the field is
present and above 0. A 0 is the empty birthday the app sent, and it gets
no notice.

The import form deletes its table, its gate, its range line and the
snackbar that named the range. The button enables on the key alone. The
restore loader deletes its table and its "Invalid Birthday" alert and
passes the height verbatim. The previous-install card shows the saved
birthday as saved.

## Considered options

One app module that serves the form, the loader and the card from one
table was rejected. It keeps the copy, and a copy cannot answer for
regtest or offline.

A Binding Layer call that returns the activation height for a chain hint,
with the app raising the height itself, was rejected. The app would still
hold the rule, and a wallet file raised on read would be invisible to it.

A notice computed in the loader from the height it sent and the height
the restore returned was rejected. It leaves a wallet file raised on open
silent and puts a second consumer beside the backend's birthday fetch.

Treating 0 as "no height" in zingolib, with no adjustment reported, was
rejected in favour of reporting it like any lower height. The app's one
consumer then applies one rule, present and above 0, with no memory of
what the loader sent.

## Consequences

The app cannot refuse a low height before the native call. The restore
runs, zingolib raises the height, and the user learns the height used from
the notice and from the backup screens.

The 2.1.0 users of zingo-mobile issue #1701, whose saved entry holds a
birthday of 0, restore from activation without a notice.

The change lands in two zingo-mobile pull requests. PR #1703 keeps a
birthday of 0 in `fetchWallet`, reads a missing saved birthday as 0, which
repairs the welcome screen crash, and raises a low height in the loader.
PR #1704 deletes both tables and the loader raise, and needs a Binding
Layer built from zingolib PR #3042, which implements zingolib issue
#3038.

The `BirthdayBelowSapling` check in pepper-sync stays as a guard that the
raise makes unreachable. The zingo-cli hint to pass `--birthday 0` becomes
true.
