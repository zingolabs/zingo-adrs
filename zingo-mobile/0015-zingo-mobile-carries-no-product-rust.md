# zingo-mobile carries no product Rust

Date: 2026-09-28

## Status

proposed

Ruled in a grilling session on 2026-09-28, pending review. The same day,
before merge, the maintainer reversed the part that rewrote the workbench
in TypeScript. A later session that day deferred the rewrite of the test
harnesses, possibly forever, and settled where `RustFFITest.kt` goes.

## Context

At the Freeze Commit (TFC), zingo-mobile `f3d1ac9d4`, the `rust/`
workspace holds seven crates. Record `zingolib/0054` copies three of them
into zingolib as the Binding Layer: `rust/lib`, `rust/uniffi-bindgen`,
and `rust/nym-proxy-ffi`. Four crates would remain after the repoint:

- `rust/android` is the Rust harness for the Android integration and
  end-to-end suites.
- `rust/ios` is the Rust harness for the iOS integration suite.
- `rust/zingomobile_utils` is a helper library for both harnesses.
- `rust/workbench` holds zingo-mobile's CI tooling, including the
  `ci-gate` binary that the integration workflow runs.

## Decision

After the repoint, zingo-mobile carries no product Rust. The Binding
Layer's crates leave zingo-mobile, and zingo-mobile builds them from its
pinned zingolib, as `zingo-mobile/0016` records.

The workbench stays in Rust, in keeping with the rule that committed
tooling is Rust in a workbench crate.

The test harnesses and `zingomobile_utils` also stay in Rust. Rewriting
them in TypeScript is deferred, possibly forever: the rewrite is churn
that may bring no net benefit, and the harnesses are the machinery that
judges the Binding Layer. After the repoint, they take zingolib through
path dependencies into zingo-mobile's zingolib submodule, so one pin
serves the app and its tests.

`RustFFITest.kt` stays in zingo-mobile until the repoint lands. A
zingo-mobile pull request then deletes its one call into the RN Bridge,
which no assertion depends on, and a follow-up moves the file into
zingolib, as `zingolib/0054` describes.

## Considered options

Moving the harnesses into zingolib unchanged was rejected. They drive the
React Native app, so zingolib would take on the app's test machinery and
become tied to one consumer. Rewriting the harnesses in TypeScript was
ruled first and then deferred, because it redefines working test
machinery in a second language for no clear gain. Moving the harnesses
into the workbench was rejected, because it would weigh a light tooling
crate down with zingolib's test utilities. Rewriting the workbench in
TypeScript was ruled first and then reversed, for the same reason as the
harnesses.

## Consequences

This supersedes the 2026-09-23 ruling that kept the harnesses and
`zingomobile_utils` in zingo-mobile for a later rewrite.

zingo-mobile still needs a Rust toolchain, both to build the Binding Layer
from source and to run its Rust harnesses and workbench.

The `rust/` workspace keeps `android`, `ios`, `zingomobile_utils`, and
`workbench` as members. The repoint removes `lib` and `uniffi-bindgen`
from it, and deletes `rust/nym-proxy-ffi`.
