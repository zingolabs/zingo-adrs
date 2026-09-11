# Decision records live in zingo-adrs and are mirrored into code repositories by subtree

## Status

superseded by [005](005-code-repositories-point-at-zingo-adrs-by-submodule.md)

Only the mirroring mechanism is superseded: code repositories hold a
submodule pointer, not a subtree copy. The scopes, numbering, citation
rule, status vocabulary, and sole-write-side rule all stand.

## Context and decision

Zingolabs decisions were recorded in two places that could not see each
other: this repository held three org-level records with no status and no
index, and zaino held sixteen numbered records under `docs/adr/` with its own
append-only rule. Numbers collided across the two (`003` and `0003` name different
decisions), a zaino record superseded an org record without the org record
saying so, and a code change deleted two zaino records instead of superseding
them. A reader could not tell which decisions were current, nor which
repositories a decision bound.

This repository, zingo-adrs, holds the records for every zingolabs code repository. It
has two kinds of record, told apart by path:

- An **org-scoped record** binds every code repository and lives at the top
  level.
- A **repo-scoped record** binds exactly one code repository and lives in
  that repository's subdirectory (`zaino/` for zaino).

Each scope numbers its records in its own sequence. A bare `ADR-NNNN` cites
a record in the citing repository's own scope; a citation into another scope
carries the path (`zaino/0016`, or the org-level `003`).

Every record states its standing on the first line under `## Status`, drawn
from a closed vocabulary: `proposed`, `accepted`, or `superseded by` followed
by a link to the successor. The root README carries a generated index per
scope, and a check in this repository's CI fails any pull request whose
records or index break these rules.

zingo-adrs is the sole write side. A record is proposed as a pull request
here, never in a code repository. Each code repository carries a read-only
copy of the whole of zingo-adrs, added with `git subtree add --squash` and refreshed
with `git subtree pull --squash` whenever a contributor chooses. A stale copy
is not a defect: a record documents a decision, not the code, so no code
repository gates on the copy's freshness or shape.

## Alternatives rejected

- **Per-repository record sets, cross-linked.** Every multi-repository decision
  would need a home, and the two record sets that existed already disagreed.
- **A git submodule or bare links** instead of a subtree. A submodule
  demands a pinned commit and a second clone step for every reader; links
  put the records a network hop away from the code that cites them.
- **Bidirectional subtree sync.** Editing under the copy and pushing upstream
  later forks the history of zingo-adrs across every consumer and depends on a
  step people forget.
- **One repository-wide numbering sequence.** It would renumber every zaino
  record and falsify every citation in published changelogs.
- **Splitting `current/` from `superseded/` directories.** Moving a file on
  supersession breaks every citation to it and contradicts the append-only
  rule.

## Consequences

- Records are copied into zingo-adrs without their prior git history; the
  originating repository keeps that history at the old paths.
- A code change and the record it implements land in two pull requests in
  two repositories, and the code may cite a record its own copy does not yet
  hold.
- Contributors who edit under a code repository's copy will conflict on
  merge; that conflict is the intended signal.
