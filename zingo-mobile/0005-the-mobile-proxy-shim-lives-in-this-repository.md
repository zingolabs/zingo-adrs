# 5. The mobile proxy shim lives in this repository

Date: 2026-08-10

## Status

accepted

Ratified by the maintainer in the 2026-08-10 design session
that superseded PR #1251. Mirrored on the zingolib side by the ADR 0011
amendment in zingolib PR #2666, which removes the shim from that
workspace.

Record `zingolib/0054`, proposed, would reverse this decision and return
the shim to zingolib.

## Context

The mobile app hosts the Nym mixnet SOCKS5 proxy in-process through a
UniFFI shim, the crate `zingo-nym-proxy-ffi`. The shim wraps `NymProxy`
from zingolib's `zingo-netutils`, and the mobile host is its only
consumer. It began life inside zingolib, which left this repository
consuming pre-built bundles from a producer that had no other reason to
build them. The superseded migrate-nym-bindgen branch (PR #1251) pulled
in the opposite direction: it vendored all of `zingo-netutils` into
this repository as `rust/nym-host`, forking a crate zingolib continues
to maintain, and it pinned zingolib to a branch that has since been
deleted.

## Decision

Only the shim moves. `zingo-nym-proxy-ffi` relocates to
`rust/nym-proxy-ffi/` under its original crate name, which UniFFI turns
into the binding namespace the Kotlin and Swift hosts import. Every
other crate remains in zingolib, and every zingolib reference here
names `branch = "dev"`. The shim's former `path` dependency on
`zingo-netutils` becomes a git dependency on zingolib dev with the
`nym` feature.

The crate sits outside the main cargo workspace (`exclude`), with its
own committed `Cargo.lock`, because nym-sdk's transitive graph needs
`crypto-common ^0.2` and the main workspace pins a release candidate
that cannot unify with it. The build tooling follows the crate:
`bundle-android-shim` ports from zingolib's workbench into this
repository's, `build_ios.mjs` builds the shim from its new path, and
the iOS artifact is renamed from `NymHost.xcframework` to
`ZingoNymProxyFFI.xcframework` so no artifact carries the name of a
directory that never existed on dev.

## Considered options

Leaving the shim in zingolib was rejected because a crate with one
consumer belongs beside that consumer, and cross-repository bundle
handoff made every shim change a two-repo dance. Vendoring all of
`zingo-netutils` (the nym-host approach of PR #1251) was rejected
because it forks a maintained crate, and the fork's divergence from
zingolib dev would be invisible until something broke at runtime.

## Consequences

The shim's lockfile and the main workspace's lockfile pin different
zingolib revisions over time. The wallet reaches the shim only through
its SOCKS5 listener and the UniFFI surface, never through shared Rust
types, so the skew is accepted. A revisit of the floating `dev` pin is
scheduled rather than left open-ended.

CI in this repository owns the crate's verification: a host-side job
runs check, clippy, fmt, and nextest inside the excluded workspace.
Per-ABI cross-builds are deferred to the release path. The
produce/consume split (`bundle-android-shim` stages a tree that
`consume-android-shim` ingests) exists because the artifacts once
crossed a repository boundary, and a follow-up may collapse the two
tools into one.
