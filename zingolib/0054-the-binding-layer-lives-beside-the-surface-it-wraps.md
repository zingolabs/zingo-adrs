# The Binding Layer lives beside the surface it wraps

## Status

proposed

Ruled in a grilling session on 2026-09-23, pending review. Amended in a
grilling session on 2026-09-28, which replaced the move with a copy that
must pass equivalence gates before zingo-mobile repoints.

## Context

zingo-mobile carries two UniFFI crates that project zingolib into Swift
and Kotlin. The wallet crate (`rust/lib`, lib name `zingo`) wraps
zingolib through a UDL file. The proxy crate (`rust/nym-proxy-ffi`,
`zingo-nym-proxy-ffi`) hosts the Nym SOCKS5 proxy and resolves in its own
lockfile, because nym-sdk needs `crypto-common ^0.2` and the main graph
pins a release candidate. Record `zingo-mobile/0005` moved the proxy crate out
of this repository on the ground that a crate with one consumer belongs
beside that consumer.

A zingolib change that breaks either crate is discovered only when
zingo-mobile next moves its zingolib pin, often weeks later. A second
React Native consumer, Edge, would have to copy the crates or depend on
zingo-mobile. Dorian proposed the remedy: when the FFI lives in zingolib,
one pull request carries a surface change and its binding change, and a
change that forgets the binding fails at once.

The native code in zingo-mobile divides into two layers. The Binding
Layer is the UniFFI crates, the Swift and Kotlin generated from them,
thin idiomatic wrappers, and the platform packaging. The RN Bridge is the
React Native modules and the app shell (`RPCModule`,
`NymTransportModule`, `MainActivity`, `AppDelegate`, and their peers),
which import React Native and marshal calls between JavaScript and the
Binding Layer.

## Decision

The Binding Layer moves into zingolib. The RN Bridge stays in each React
Native consumer's own repository. This follows ADR 0024: consumers
converge on a surface zingolib owns, and the Binding Layer is that surface
projected into Swift and Kotlin, so zingolib owns it however many
consumers exist.

The layout is:

```
zingo-ffi/                    package zingo-ffi, lib name zingo (root member)
zingo-netutils/nym-proxy-ffi/ package zingo-nym-proxy-ffi (netutils workspace member)
bindings/android/             Gradle library; one AAR carrying both libraries
bindings/swift/               Package.swift; one XCFramework carrying both libraries
```

Once zingo-mobile has repointed, the wallet crate joins the root
workspace, and the proxy crate joins the standalone zingo-netutils
workspace, which already builds with `nym` on, so its patch onto
`webpki-verifier-shim` becomes a path patch. Until then each crate is a
standalone workspace, as the next section describes. Both lib
names stay unchanged, so the generated namespaces `uniffi.zingo` and
`uniffi.zingo_nym_proxy_ffi` survive the move.

This record does not overrule `zingo-mobile/0013`, which renames the proxy
crate to `mixnet-proxy` and retires the word "shim". The two decisions
apply in sequence. This record governs the relocation, which keeps every
name so that the move stays verifiable as a diff. The rename follows as a
separate change inside zingolib, like the reshapes below, and 0013 governs
the name the crate takes then.

Reshaping the surface is a separate, later concern: structured errors in
place of `ffi_error`'s string flattening, proc-macros in place of UDL,
and one UniFFI version in place of 0.28 and 0.29.

### Copy, then verify

zingolib receives a copy of the crates, and zingo-mobile's originals stay
in place and unchanged. zingo-mobile repoints only after the copy passes
the equivalence gates below.

Preserved function means two things. The Rust source is byte-identical to
the original. The packaging may be new Gradle and SwiftPM code, but what
it produces must match, at the boundary consumers see, what zingo-mobile's
builders produce.

Before the copy, one zingo-mobile pull request moves both crates' zingolib
pins to a single zingolib rev, R. That pull request's merge commit is the
source commit, S. zingo-mobile's `rust/` freezes at S and stays frozen
until the repoint lands. The import branches from R.

The crates arrive with their git history, filtered from S, in an import
commit that changes nothing. A separate placement commit makes the
manifest edits. Until the repoint, each crate is a standalone workspace,
excluded from the root and zingo-netutils workspaces, and carries its
lockfile from S verbatim. The only change between S and the copy is
where the crates live, and between the import and the repoint the copy
accepts only packaging changes.

Four manual gates, each run against S and R, prove preservation:

1. The tree hashes of the imported crate directories equal those of
   `S:rust/lib` and `S:rust/nym-proxy-ffi`.
2. The placement commit touches only manifests, and each lockfile diff is
   confined to the zingolib crates' source lines.
3. The generated Kotlin and Swift bindings match those generated at S byte
   for byte. The AAR and XCFramework carry the same ABI set, the same
   generated sources, and the same exported dynamic symbols. Their build
   inputs also match: the Cargo profile, the NDK version, the minimum SDK
   and iOS versions, the JNA version, and the package and module names.
4. zingo-mobile's full suite, `RustFFITest.kt`, `ZingoTest.swift`, and the
   Detox end-to-end tests, runs on a discarded zingo-mobile branch that
   consumes the copy through the new packaging. Each test's outcome must
   match a baseline recorded from three runs at S. A test that fails at S
   and fails on the copy counts as preserved function.

The copied crates' own Rust tests run in per-pull-request CI rather than
as a manual gate. There, `live_mixnet.rs` may fail on the network without
failing the check.

Each gate has a fixed remedy for failure:

1. If gate 1 fails, the import is discarded and redone from S, never
   corrected by hand.
2. If gate 2 fails, the placement commit is rewritten.
3. If gate 3 or gate 4 fails because of the packaging, the packaging is
   fixed and both gates run again.
4. If gates 1 and 2 pass but gate 3 or gate 4 fails for any other reason,
   the plan stops. Identical source and identical lockfiles should not
   produce different behavior, so such a failure means an assumption is
   wrong, and the remedy is another grilling session, not a code change.

The repoint moves zingo-mobile to a zingolib rev later than R, because
zingolib's `dev` keeps moving after the import merges. The repoint pull
request therefore passes gate 4 again before it merges.

zingo-mobile builds the Binding Layer from source at one pinned zingolib
rev, through Gradle `includeBuild` and a SwiftPM local package path. The
consumer chooses the version. zingolib publishes no prebuilt bundle until
a consumer needs one.

Tests split along the same line. `RustFFITest.kt` calls the generated
bindings directly, so it is copied to `bindings/android` with its Rust
driver, and zingo-mobile's original stays until the repoint.
`ZingoTest.swift` drives `RPCModule` and the Detox suites drive the app,
so they stay in zingo-mobile.

CI enforces the rationale in two tiers. Every pull request checks
`zingo-ffi` with the `nym` and `perspective` features, generates the
Kotlin bindings, and builds the AAR for x86_64. A nightly run builds the
AAR for all four ABIs, builds the XCFramework on macOS, and runs
`RustFFITest.kt` on an emulator. These nightly jobs replace the calls to
zingo-mobile's reusable workflows.

## Considered options

A separate `zingolib-ffi` repository was rejected. It restores the
cross-repository delay that motivates the move, and nobody identified a
benefit beyond separation. Leaving the crates in zingo-mobile was rejected
for the same reason, and because Edge would then depend on another app's
repository. Moving the RN Bridge as well was rejected, because it would
bring React Native into zingolib and tie Edge to Zingo's bridge.

The 2026-09-28 amendment rejected four alternatives. A move that deleted
the originals at import was rejected, because it leaves nothing to check
the copy against. A plain snapshot without history was rejected, because
it loses `git blame` and turns gate 1 into a directory diff. Joining the
shared workspaces at import was rejected, because the check would then
have to prove a resolved-graph diff harmless. A fully green suite as
gate 4 was rejected, because pre-existing failures at S would block a
copy that preserves them, and fixing them would break the freeze.

## Consequences

This supersedes `zingo-mobile/0005` and reverses ADR 0011's 2026-08-10
relocation amendment. When this record is accepted, the status line of
`zingo-mobile/0005` changes from `accepted` to `superseded by` a link to
this record.

The RN Bridge still breaks in zingo-mobile's CI, not zingolib's, when
zingo-mobile moves its pin. That break is deliberate and bounded to the
bridge.

zingo-mobile's `rust/` freezes from S until the repoint lands. Open pull
requests that touch it are merged before S, closed, or replayed into
zingolib with `git am --directory` after the repoint, since a replay
before then would break the copy's byte identity.

## Pull request dispositions

The grilling session ruled on each open zingo-mobile pull request that
touched `rust/` on 2026-09-23:

- zingo-mobile#1418 merges before the freeze.
- zingo-mobile#1212 and zingo-mobile#1297 close, because the repoint
  supersedes the zingolib pins they move.
- zingo-mobile#1197, which excises the regchest backend, rebases and
  merges before the import.
- zingo-mobile#1233, which removes duplication from the accessors, rebases
  and merges before the import.
- zingo-mobile#1242 and zingo-mobile#1243, which reshape the migration
  cadence surface, close and return as reshapes inside zingolib after the
  move.
- zingo-mobile#1293, which renames the proxy namespace, closes and returns
  after the move as the rename that `zingo-mobile/0013` governs.
- zingo-mobile#1323 replays into zingo-netutils after the repoint.
- Dorian splits draft zingo-mobile#1365 along the line between the
  Binding Layer and the RN Bridge.

By 2026-09-25, #1418 and #1197 had merged, and #1212, #1297, #1242, #1243,
#1293, and #1323 had closed. #1233 closed by mistake without merging.
Whether it is replayed into zingolib or dropped is still open. #1365 is
still open.
