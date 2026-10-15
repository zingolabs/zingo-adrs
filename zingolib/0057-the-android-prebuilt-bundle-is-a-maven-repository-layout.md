# The Android Prebuilt Bundle is a Maven repository layout that carries its own POM

## Status

proposed

Accepted when zingolib#2979 merges; from then on the layout is the only shape
published.

## Context

zingolib publishes the Android Prebuilt Bundle as one AAR in the OCI registry
(zingolib/0054; zingolib#2833 decisions 3 to 5; zingolib#2885), which ruling 3
of zingolib#2930 had Gradle add as a file dependency while every consumer
declared its runtime dependency JNA by hand, so a Bundle Pin move could leave
the app on a stale JNA silently, the deferred item "JNA in the lock" of
zingolib#2930, raised 2026-10-12, to decide before its step 4. Three answers
were weighed, and the rulings of 2026-10-15 chose the third.

1. Annotate the OCI manifest with the JNA coordinate and pin it, as the
   deferred item proposed and zingolib#2969 drafted before closing as
   superseded by zingolib#2979; it restates one dependency in three places
   and covers JNA alone.
2. Publish the AAR to a hosted Maven repository, which a design session on
   2026-10-05, recorded in no issue, deferred for the account, credentials
   and release process it needs; decision 1 of zingolib#2833 makes the
   GitHub Container Registry the store that ruling 5 of zingolib#2930 has
   every consuming job fetch from by digest.
3. Keep the registry and change the layer: Gradle's `maven-publish` plugin
   writes the release component into a directory in canonical Maven layout,
   the AAR, its POM (Project Object Model, which lists its dependencies, JNA
   among them) and `maven-metadata.xml`, and that directory is the bundle.

## Decision

The Android Prebuilt Bundle is a Maven repository directory in canonical
layout, written only by `maven-publish` and carried in the OCI registry as
one gzip tarball, as the iOS bundle carries the Swift package. The POM
declares every dependency the AAR needs, JNA among them; a consumer declares
none by hand.

The coordinate is `org.zingolabs:zingo-binding-layer:<version>`, the version
being the zingolib crate version plus the first sixteen characters of the
published commit, as in `6.0.0-ba7336710419c9ac`. The workbench derives it
and passes it to Gradle as a property; the build refuses to publish without
it. The package job runs Gradle's `publish` task into the directory and
uploads it; `binding-publish` archives both platforms through one function.
The fetch command reads the coordinate from `maven-metadata.xml` and prints
it beside the repository root; the Bundle Pin keeps its shape. The fetch and
pin commands refuse the old shape by media type, so the one bare-AAR entry,
ba7336710 of zingolib#2885, cannot be pinned for Android again.

## Consequences

A Bundle Pin move carries the AAR's dependencies, and a new one reaches every
consumer through the POM with no change to publication, pin package or
consumer. The consumer's Gradle wiring changes once, from the file dependency
and the JNA line to a `maven` repository at the printed path and one
`implementation` line with the printed coordinate, which amends ruling 3 of
zingolib#2930 and decision 3 of zingolib#2833. kitchen-sync changes when its
pin first moves to the layout; zingo-mobile starts on it in step 4 of
zingolib#2930.

Hosted Maven publication, if ever taken up, is the same publication pointed
at another repository. The Android bundle's media type becomes gzip, and the
glossary's Prebuilt Bundle entry in `zingolib/CONTEXT.md` is restated. The
version embeds a commit prefix, so it does not sort as a release and no
consumer may resolve it dynamically, which is the intent.
