# The app offers only the drain migration

Date: 2026-10-25

## Status

accepted

Ruled on 2026-10-25, when the team chose to stop developing the private
migration. zingo-mobile PR #1762 removes it from the app, and zingolib
PR #3089 drops its driving functions from the Binding Layer.

## Context

The app offered two ways to move Orchard funds into Ironwood. The drain
([zingolib/0019](../zingolib/0019-immediate-migration-is-send-shaped.md))
sends every Orchard note in one session. The private migration splits the
notes first
([zingolib/0016](../zingolib/0016-note-splitting-is-a-stateless-fused-call.md))
and then sends parts inside scheduled windows
([zingolib/0017](../zingolib/0017-phase-2-parts-due-for-the-whole-open-window.md)).
Beta builds 328 to 332 offered the private path. Every build after them
showed it as a disabled card, while its six screens, its bridge methods and
its batch reminders still shipped, and a reconcile of it ran on every
launch.

## Decision

The app offers the drain as its only migration. The private path's
screens, routes, bridge methods and reminders leave the app.

A private migration that an older build left in flight is cancelled at
launch without a word to the user: the app reconciles it, so the parts
that confirmed stand, then cancels the rest, so the drain can move the
notes they held. The app keeps `migration_status`, `reconcile_migration`
and `cancel_ironwood_migration` for that alone.

## Consequences

The one-way group of the route tree in
[zingo-mobile/0018](0018-one-route-tree-chosen-by-the-session.md) holds
MeetIronwood, MigrationStrategy, MigrationTransactions and MigrationSending.
The six private routes it lists no longer exist.

Whether Orchard holds only a residual, which decides the Home notice's
dismiss and the Pools note, comes from the drain plan: no transaction
planned and a residual above zero.

zingolib keeps the private migration's logic. Only the Binding Layer
functions the app no longer calls leave. zingo-pc still offers the private
migration through its own native module, which links the zingolib crate
directly, and this record does not cover it.
