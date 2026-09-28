# zingo-mobile keeps Rust only in its workbench

Date: 2026-09-28

## Status

proposed

Ruled in a grilling session on 2026-09-28, pending review. The same day,
before merge, the maintainer reversed the part that rewrote the workbench
in TypeScript.

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

The grilling session of 2026-09-23 kept the harnesses and
`zingomobile_utils` in zingo-mobile. The maintainer's standing rule sends
committed tooling to a Rust workbench crate, which would also keep
`rust/workbench` in zingo-mobile.

## Decision

After the repoint, zingo-mobile's only Rust source is its workbench
crate.

The harnesses and `zingomobile_utils` are rewritten in TypeScript, inside
zingo-mobile's existing Jest and Detox tooling. The workbench stays in
Rust, in keeping with the rule that committed tooling is Rust in a
workbench crate. Its logic is not rewritten, so the tooling carries no
churn. It stays where it is, as the only member of the `rust/`
workspace.

The removal belongs to the repoint, not to the copy. zingo-mobile's
`rust/` stays frozen and unchanged from TFC until the repoint lands, so
the copy's gates still compare against it.

## Considered options

Moving the harnesses into zingolib unchanged was rejected. They drive the
React Native app, so zingolib would take on the app's test machinery and
become tied to one consumer. Keeping the harnesses in Rust, as the
2026-09-23 session ruled, was rejected, because the maintainer requires
that zingo-mobile carry no product or test Rust. Rewriting the workbench
in TypeScript was ruled first and then reversed, because it would
redefine working tooling in a second language for no gain in function.

## Consequences

This supersedes the 2026-09-23 ruling that kept the harnesses and
`zingomobile_utils` in zingo-mobile.

zingo-mobile still needs a Rust toolchain to build. Under `zingolib/0054`
it builds the Binding Layer from source through Gradle `includeBuild` and
a SwiftPM local package path, until zingolib publishes a prebuilt bundle.

The CI cache key that hashes `rust/**` and the `rust:*` scripts in
`package.json` change with the removal. The workflows that run
`cargo run -p workbench` keep working unchanged.

## Open

`RustFFITest.kt` is copied to zingolib's `bindings/android`. Its driver in
`rust/android` could become TypeScript in zingo-mobile or move to zingolib
with the test. This record does not yet rule on it.
