# zingo-mobile takes the Binding Layer from published Prebuilt Bundles

Date: 2026-10-12

## Status

proposed

Ruled in a grilling session on 2026-10-12, recorded in zingolib#2930. The
record moves to `accepted`, and `zingo-mobile/0016` to `superseded by` it,
in the pull request that deletes the submodule, which is step 6 of that
plan. Until then the submodule still builds the app, and `zingo-mobile/0016`
still binds.

## Context

Record `zingo-mobile/0016` holds zingolib as a git submodule at `zingolib/`
and builds the Binding Layer from it, through a Gradle `includeBuild` and a
local Swift package. zingolib#2833 moved that build into zingolib, which now
publishes each pinned commit as two Prebuilt Bundles on the GitHub Container
Registry: an AAR for Android and a Swift package for iOS, each tagged with
the full commit and immutable. kitchen-sync, a second consumer, proved on
2026-10-12 that an app can fetch both bundles by digest and call
`get_version` through them (kitchen-sync#1).

The submodule feeds more than the build. Fourteen workflows check it out.
Two required checks run zingolib's crate tests from it, the Android unit
tests compile a wire-contract test out of it, and one Jest test reads a
Rust source file for an error prefix. The Rust harness crates that
`zingo-mobile/0016` also served are gone since 2026-10-02.

## Decision

zingo-mobile deletes the zingolib submodule and takes the Binding Layer
only from the Prebuilt Bundles that zingolib publishes.

1. One lock file holds the Bundle Pin: the zingolib commit and the digest
   of each platform's bundle. It is the one statement of the pinned
   zingolib. No build of the Binding Layer remains in zingo-mobile, and no
   override points the build at a local one.
2. zingo-mobile manages its bundles as kitchen-sync manages its own, and
   only a significant trade-off may separate the two. This record binds
   zingo-mobile alone. The rule that every consumer manages the bundles
   identically is zingolib#2930 ruling 2, and no org-scoped record holds
   it yet.
3. The AAR enters Gradle as a file dependency at the path the fetch
   prints, and the app declares JNA at the version the bundle's build
   declares. The fetch extracts the Swift package to a fixed path that
   Xcode references as a local package, guarded by a digest marker.
4. One required CI job per platform fetches the bundle and checks that
   the registry's tag still resolves to the pinned digest. Every other job
   fetches for itself by an anonymous pull verified by digest, with no
   cache in front of it until a measurement calls for one.
5. The tests that read zingolib's source are zingolib's tests, and they
   run there: the crate tests, the live mixnet start, and the Kotlin and
   Swift wire-contract tests. The mixnet error prefix is pinned where it
   is minted until the RN Bridge classifies by the typed exception.
6. The release takes the pinned AAR as its only native input, and the R8
   keep-verifier reads its class list from the AAR's `classes.jar`.
7. A developer who changes the Binding Layer publishes the commit through
   zingolib's publish request and pins it.

## Considered options

Keeping the submodule beside the lock as a lasting arrangement, with a
check that the two name one commit, was rejected. The check guards a
second statement of one fact, and the submodule's only remaining readers
are tests that belong to zingolib. The two do stand side by side through
steps 4 and 5 of zingolib#2930, where ruling 10 holds them to one commit.

A local input override, a Gradle property and an Xcode setting that take
a Binding Layer built in a zingolib checkout, was rejected. It is a second
code path that CI never exercises, and it keeps alive the mechanism this
record removes.

Copying the tests that read the submodule into zingo-mobile was rejected.
Each checks something zingolib mints.

## Consequences

The Binding Layer build leaves zingo-mobile's CI and its release, and the
cache keys, the build workflows, and the actions that served it go with
it. A pull request of zingo-mobile downloads two bundles instead of
building native code.

Decision 7 has a price. Each iteration on the Binding Layer is a zingolib
pull request, a `/publish` comment by someone with write permission, about
21 minutes of build, a manifest entry and a changelog section that never
leave, and 615 MiB that the registry keeps forever. The publish command
refuses a pull request from a fork, so a fork has no path to a Binding
Layer change. The rejected override would have spared a local change all
of that, at the cost of a second code path.

zingo-mobile's `CONTEXT.md` takes Prebuilt Bundle, Bundle Pin, Binding Set
and Binding Layer Endpoint from zingolib's glossary by reference.

A pin bump is one change to the lock file. The lock does not yet carry the
JNA coordinate that the AAR needs, so a bump can desync the two; zingolib#2930
holds that question.

zingolib#2930 sequences the work so that each new piece runs before the
old piece it replaces is deleted.

## Amendment (2026-10-21): the publish command is a workflow dispatch

The Consequences section prices an iteration on the Binding Layer at a
zingolib pull request and a `/publish` comment. zingolib#3046 removed
both. `gh workflow run binding-layer-publish.yaml --ref dev --field
commit=<sha>` publishes any commit of `dev`, and the workflow opens the
record pull request itself. The build time, the entry, the changelog
section and the registry's retention stand. A fork still has no path to
a Binding Layer change, because the plan refuses a commit that is not on
`dev`.
