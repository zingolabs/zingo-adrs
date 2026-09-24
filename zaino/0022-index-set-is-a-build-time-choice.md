# The index set is a build-time choice

## Status

proposed

Supersedes [zaino/0018](0018-served-block-representation-is-an-index-set-choice.md),
which framed the index set as a runtime configuration input and left the
choice open.

## Context and decision

zaino/0018 proposed that each deployment choose its index set in
configuration, that zaino answer reads it had not yet synced by passing them
through to the validator, and that a serviceability manifest advertise which
reads each configuration could answer. The finalised store carried a
capability bitmap for that purpose, and the same bitmap also let the store
report reduced capability during a migration and serve old data while a new
index was built.

Zaino has since dropped database migrations, and zainod no longer serves
indexes before its index is synced. Only one purpose of the capability
bitmap remains: gating the non-standard indexes, such as transparent address
history, that some deployments need and others do not.

We decide that the index set is fixed when zainod is built. Each
non-standard index is a cargo feature of zainod. The lean set, compact blocks
with their commitment tree sizes folded in, is the default build. The enabled
feature set feeds the database schema hash, so a zainod built with a
different index set finds a mismatched hash and rebuilds its index from
zebra. A build that omits an index also omits the reads that need it.

## Consequences

- The store's capability bitmap, its single-capability requests, and the
  version-to-capability mapping are deleted.
- No serviceability manifest is built; the binary's feature set is the whole
  answer to which reads a deployment serves.
- Changing a deployment's index set means installing a different build and
  paying one full rebuild of the index.
