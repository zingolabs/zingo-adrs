# zingo-mobile pins zingolib by submodule

Date: 2026-09-28

## Status

accepted

Ruled in a grilling session on 2026-09-28. Accepted on 2026-10-01, when
zingo-mobile#1467 merged into `dev` as `b3354c4c0` with zingolib as a
shallow submodule at `zingolib/`. The submodule holds no tags at depth
one, and the Binding Layer's `zl_` descriptor therefore names the release
tag that points at HEAD after a tags-only fetch, which zingolib#2812
settled. The CI jobs that build native code fetch the submodule's tags.

## Context

Record `zingolib/0054` moves the Binding Layer into zingolib, and
zingo-mobile builds it from source at one pinned zingolib rev, through
Gradle `includeBuild` and a SwiftPM local package path. Both mechanisms
need a zingolib checkout on disk. Before the repoint, zingo-mobile's Rust
harnesses pin zingolib separately, by git revision in `rust/Cargo.toml`.

## Decision

zingo-mobile holds zingolib as a git submodule at `zingolib/`. The
submodule's commit is zingo-mobile's one zingolib pin. A bump is a
one-line change to that commit.

Gradle includes `zingolib/bindings/android`, and Xcode references the
local package `zingolib/bindings/swift`. The Rust harnesses in `rust/`
take zingolib through path dependencies into the submodule, not through
git revisions, so the app and its tests always use the same zingolib.

A checkout needs `--recurse-submodules`, or `git submodule update --init`
after cloning. A shallow, blob-filtered submodule clone keeps the cost of
zingolib's size small.

## Considered options

A git subtree was rejected. It copies zingolib's files into zingo-mobile's
tree, which returns zingolib's Rust to zingo-mobile against
`zingo-mobile/0015`. It also restores a second, editable copy of the
Binding Layer, which `zingolib/0054` exists to remove. A subtree bump is a
large file diff, and it records the rev only in a merge message.

A rev recorded in a file, which scripts use to clone zingolib, was
rejected. Every script must agree on how to fetch, and local builds drift
easily.

An environment variable that names a zingolib checkout, as gate 4 uses,
was rejected as the lasting mechanism, because it pins nothing.

Separate git revisions for the harnesses were rejected, because they can
drift from the app's pin, and a harness would then judge the app against a
different zingolib.

## Consequences

zingo-mobile already runs this workflow for `docs/adr`, which
`zingo-adrs/005` points at zingo-adrs by submodule. The same costs apply:
a submodule update step, worktrees that need `--force` to remove, and pin
changes that a careless commit can carry.

`rust/Cargo.lock` stops carrying a separate zingolib source.
