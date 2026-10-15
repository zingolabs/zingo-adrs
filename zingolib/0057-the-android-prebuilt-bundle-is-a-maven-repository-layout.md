# The Android Prebuilt Bundle is a Maven repository layout that carries its own POM

## Status

proposed

Accepted when the implementing pull request merges; from then on the layout
is the only shape published.

## Context

zingolib publishes the Android Prebuilt Bundle as one AAR in the OCI
registry, which Gradle added as a file dependency while every consumer
declared its runtime dependency JNA by hand, so a Bundle Pin move could leave
the app on a stale JNA silently. Three answers to that deferred item were
weighed, and the rulings of 2026-10-15 chose the third.

1. Annotate the OCI manifest.
2. Publish the AAR to a hosted Maven repository.
3. Keep the registry and change the layer: Gradle's `maven-publish` plugin
   writes the release component into a directory in canonical Maven layout,
   the AAR, its POM and `maven-metadata.xml`, and that directory is the
   bundle.

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
pin commands refuse the old shape by media type, so the one bare-AAR entry
cannot be pinned for Android again.

## Consequences

A Bundle Pin move carries the AAR's dependencies, and a new one reaches every
consumer through the POM with no change to publication, pin package or
consumer. The consumer's Gradle wiring changes once, from the file dependency
and the JNA line to a `maven` repository at the printed path and one
`implementation` line with the printed coordinate, which amends the earlier
consumption ruling and the artifact decision. kitchen-sync changes when its
pin first moves to the layout; zingo-mobile starts on it.

Hosted Maven publication, if ever taken up, is the same publication pointed
at another repository. The Android bundle's media type becomes gzip, and the
glossary's Prebuilt Bundle entry in `zingolib/CONTEXT.md` is restated. The
version embeds a commit prefix, so it does not sort as a release and no
consumer may resolve it dynamically, which is the intent.

## Appendix

### References

- zingolib/0054 and zingolib#2833, decisions 3 to 5, define the two Prebuilt
  Bundles per published commit; zingolib#2885 merged the publish workflow,
  and its merge commit ba7336710 is the one bare-AAR entry.
- zingolib#2833, decision 1, makes the GitHub Container Registry the store;
  decision 3, the Android artifact, is amended by this record.
- zingolib#2930, ruling 3, placed the AAR as a file dependency with
  app-declared JNA and is amended by this record; ruling 5 has every
  consuming job fetch by digest; its deferred item "JNA in the lock", raised
  in the review of 2026-10-12, is decided here before its step 4, in which
  zingo-mobile consumes the bundles.
- zingolib#2969 drafted the first answer, which the deferred item proposed:
  annotate the manifest with the JNA coordinate, copy it into the Bundle
  Pin, and declare JNA from the pin. It restates one dependency in three
  places and covers JNA alone, and it closed as superseded on 2026-10-15
  when zingolib#2979 opened.
- zingolib#2979 implements this record; the record is accepted when it
  merges.
- The deferral of hosted Maven publication, the second answer, was ruled in
  a design session on 2026-10-05 and is recorded in no issue: it needs a
  publisher account, credentials in the workflow and a release process
  beside the OCI one, and the registry is the store every consuming job
  fetches from by digest.

### Definitions

- An AAR (Android Archive) zips an Android library with one native library
  per ABI (Application Binary Interface, one CPU target).
- A Maven repository is a tree or server in the canonical layout Gradle
  resolves from, each artifact at `<group path>/<artifact>/<version>/` under
  a `group:artifact:version` coordinate.
- A POM (Project Object Model) is the XML beside an artifact that names its
  coordinate and dependencies; Gradle reads it for an artifact resolved by
  coordinate, and `files("x.aar")` reads none.
- `maven-metadata.xml` lists the versions a repository holds.
- JNA (Java Native Access) is the library through which the Kotlin Binding
  Set loads the native Binding Layer.
- The OCI (Open Container Initiative) registry, `ghcr.io`, holds each
  Prebuilt Bundle as one manifest with one layer, fetched by digest.
