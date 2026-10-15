# The Android Prebuilt Bundle is a Maven repository layout that carries its own POM

## Status

proposed

## Context

zingolib publishes the Binding Layer as two Prebuilt Bundles per commit
(zingolib/0054, zingolib#2833). The Android Prebuilt Bundle has been one
AAR, pushed to the OCI registry as one layer, and a consumer's Gradle
build added it as a file dependency at the path the fetch command printed.

This record uses the following terms from the Android and Maven toolchains.

- An AAR (Android Archive) is the zip file that packages an Android
  library: its compiled classes, its manifest, its resources, and one copy
  of each native library per ABI (Application Binary Interface, one CPU
  target such as `arm64-v8a`).
- Maven is the JVM world's artifact and dependency convention, which
  Gradle follows when it resolves dependencies. A Maven repository is a
  directory tree or a server in Maven's canonical layout, where the
  artifact of a coordinate lives at `<group path>/<artifact>/<version>/`.
- A coordinate is the `group:artifact:version` triple by which Gradle
  names an artifact in a Maven repository, such as
  `org.zingolabs:zingo-binding-layer:6.0.0-ba7336710419c9ac`.
- A POM (Project Object Model) is the XML file that sits beside an
  artifact in a Maven repository and names its coordinate and its
  dependencies. Gradle reads the POM of every artifact it resolves by
  coordinate and resolves those dependencies in turn. A file dependency,
  `files("x.aar")`, reads no POM.
- `maven-metadata.xml` is the file Maven writes beside an artifact's
  version directories, listing the versions the repository holds.
- JNA (Java Native Access) is the Java library through which JVM code
  calls a native shared library without hand-written glue code. The
  Kotlin Binding Set loads the Binding Layer's native libraries through
  it.
- An OCI (Open Container Initiative) registry, here `ghcr.io`, is a
  content-addressed store of manifests and blobs. zingolib publishes each
  Prebuilt Bundle as one manifest whose single layer, one blob, is the
  bundle's file, and a consumer fetches it by digest.

An AAR carries no dependency metadata. The Android build declares JNA as
`api("net.java.dev.jna:jna:5.18.1@aar")`, and the AAR needs it at runtime
but does not contain it. Every consumer therefore declared JNA by hand at
the version it believed the bundle was built with, and nothing tied the
two: a Bundle Pin move to a bundle built against a newer JNA left the app
on the old one, silently. zingolib#2930 recorded this as a deferred
question to settle before zingo-mobile consumes the bundles.

Three answers were weighed on 2026-10-14 and 2026-10-15.

The first annotates the OCI manifest with the JNA coordinate, has the pin
command copy it into the Bundle Pin, and has Gradle declare JNA from the
pin. zingolib#2969 drafted it. It keeps the AAR as the bundle and the file
dependency in Gradle, but it restates one dependency of the AAR in three
places, a Gradle manifest, an annotation and a lock file, and it covers
JNA alone: a second runtime dependency of the AAR would need a second
annotation and a second key.

The second publishes the AAR to a hosted Maven repository. Gradle would
then resolve the AAR by coordinate and read its POM. zingolib#2833 ruled
on 2026-10-05 that Maven publication is deferred: it needs a publisher
account, credentials in the workflow and a release process beside the OCI
one, and ruling 1 of zingolib#2930 holds that a consumer's only reference
is the OCI registry.

The third keeps the OCI registry as the only reference and changes what
the Android layer holds. Gradle's `maven-publish` plugin, Gradle's own
writer of Maven repositories, publishes the library's release component,
the outputs of its release build, into a directory, which is a Maven
repository in canonical layout: the AAR, the POM that `maven-publish`
derives from the build's own dependency declarations, `maven-metadata.xml`
and the checksum files. That directory, archived unchanged, is the bundle. A
consumer adds the extracted directory as a Maven repository and declares
the AAR by coordinate, and Gradle resolves JNA from the POM the way it
resolves every other Maven dependency.

## Decision

The Android Prebuilt Bundle is a Maven repository directory in canonical
layout, written by Gradle's `maven-publish` plugin from the Android
library's release component and carried in the OCI registry as one gzip
tarball, as the iOS Prebuilt Bundle already carries the Swift package.
Gradle is the only writer of the layout. The POM declares every dependency
the AAR needs, JNA among them, and a consumer declares nothing by hand.

The coordinate is `org.zingolabs:zingo-binding-layer:<version>`, with the
group and the artifact the Android project already names. The version is
the zingolib crate version and the first sixteen characters of the
published commit, as in `6.0.0-ba7336710419c9ac`, so every published
commit has its own version and a hosted Maven repository could later hold
them all without collision. The workbench derives the version and passes
it to Gradle as a property; the build refuses to publish without it.

The package job runs Gradle's `publish` task into the repository
directory and uploads it, and `binding-publish` archives it, so both
platforms' archives come from one function in the workbench. Nothing
restates the coordinate: the fetch command reads it from the
`maven-metadata.xml` that Gradle wrote and prints it beside the repository
root, and the Bundle Pin keeps its shape. The fetch and the pin command
refuse an Android bundle in the old shape by its media type, so the one
entry published as a bare AAR cannot be pinned for Android again.

## Consequences

A Bundle Pin move carries the AAR's dependencies with it. Gradle resolves
JNA at the version the bundle's own build declared, and a new dependency
of the AAR reaches every consumer through the POM with no change to the
publication, the pin package or the consumer.

The consumer's Gradle wiring changes once: the file dependency and the
hand-written JNA line give way to a `maven` repository at the printed path
and one `implementation` line with the printed coordinate. kitchen-sync
makes that change in the pull request that moves its pin to the first
bundle in the new layout, and zingo-mobile starts on the new layout.

Publication to a hosted Maven repository, if it is ever taken up, is the
same `maven-publish` publication pointed at another repository. The
coordinates and the POM do not change, and a consumer moves only its
`repositories` block.

The Android bundle is no longer one file. A reader inspecting a published
layer sees a tarball of a directory tree rather than an AAR, the pin
package extracts it as it extracts the iOS package, and the workbench's
Android media type becomes gzip. The glossary's Prebuilt Bundle entry in
`zingolib/CONTEXT.md` is restated to the layout.

The version string embeds a commit prefix, so it does not sort as a
release version and no consumer may resolve it dynamically. A Bundle Pin
names the exact commit and digest, and the coordinate names the exact
version, which is the intent.
