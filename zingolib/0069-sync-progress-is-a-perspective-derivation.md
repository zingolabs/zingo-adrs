# Sync progress is a perspective derivation

## Status

proposed

Drafted on 2026-10-12 beside PR #2928, which implements it. The type names
`SyncProgress` and `SessionReport` were ratified by the user on the same day.

## Context

`pepper_sync::sync::sync_status` computed two percentages beside the counts
it reads from the wallet: the ratio on a percentage scale, clamped to the
scale, read as complete over an empty span once a sync had started, and held
one step short of complete while any scan range awaited nullifier retrieval.
`SyncStatus` and `SyncResult` carried those percentages, and both types
rendered a `Display` text and a `json::JsonValue` document, which bound the
engine to the `json` crate.

Nothing in the engine reads a percentage. Every consumer that reads one does
so to display it: the CLI prompt, the CLI `sync status` and `sync poll`
subcommands, the FFI `status_sync` and `poll_sync` functions, and through
them zingo-pc and zingo-mobile.

The review of PR #2643, which removed two session percentages that could
abort the process, asked that the session statistics stay until this logic
moved to a more suitable place. ADR 0024, decision 7, routes consumers
through zingolib rather than a second pin on pepper-sync. PR #2623 landed
`zingolib::perspective`, the editorial layer that derives presentation-facing
statements from the canonical record and never feeds wallet logic.

## Decision

The sync engine reports facts. `SyncStatus` carries the scan ranges, the
start height, the session and total counts per pool, the exact output ratio
as two integers, and the block span as two integers. `SyncResult` carries the
session's counts and the final output totals. Both answer whether nullifier
retrieval is pending, a fact about the scan ranges. Neither carries a
percentage, a `Display` text, or a JSON document, and pepper-sync does not
depend on the `json` crate.

`zingolib::perspective::sync_progress` derives presentation from those facts.
`SyncProgress` is built from a `SyncStatus` and `SessionReport` from a
`SyncResult`, each by a `From` conversion into a plain struct. They derive the
percentages by the rule the engine applied, render the text the engine
rendered, and render the JSON document with the keys the engine used. A
consumer that decoded the old document decodes the new one.

Any further reading derived from sync progress, a session percentage among
them, is added to this module and never to the engine.

## Consequences

pepper-sync's next release is breaking: three fields and four trait
implementations leave `SyncStatus` and `SyncResult`. A consumer that read a
percentage from the raw pepper-sync types builds a `SyncProgress` or a
`SessionReport` instead, or computes its own.

The JSON documents are unchanged in every key they carried. The status
document gains `total_blocks`, and the result document gains
`total_outputs_scanned` and `total_outputs`. zingo-pc and zingo-mobile, whose
types declare every progress field optional, decode both without change.

The session percentages that PR #2643 removed can be derived in this module
from the session counts the engine still reports, which answers the
condition that review set.
