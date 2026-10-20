# The Published Commit is the identity of a Prebuilt Bundle

## Status

proposed

Accepted when the implementing pull requests merge: the one that takes the
coordinate from the manifest annotation, and the one that gives the
descriptor the sixteen characters.

## Context

One published build of the Binding Layer was referenced by seven
identifiers on 2026-10-20: the zingolib commit, the registry tag spelled
from it, one manifest digest per platform, the descriptor that
`get_version` returns, the Maven coordinate of the Android bundle, the
entry in `bindings/published.toml`, and the consumer's Bundle Pin. Two of
them were human spellings of the same build in different forms. The
descriptor was `zl_<crate version>_<five commit characters>`, or the tag's
version alone on a release, minted by `zingolib/build.rs`. The Maven
version was `<crate version>-<sixteen commit characters>`, minted by the
workbench. A review of zingolib#3030 found the pin command downloading a
whole bundle to copy the coordinate out of it, and the word "manifest"
naming both the registry's OCI image manifest and the repository's record
of publications. A grilling session on 2026-10-20 ruled on each copy.

## Decision

The Published Commit is the one identity of a published build. Every other
reference derives from it or is recorded against it, and no record, pin,
check or message names a build by anything else.

The descriptor and the Maven version are derived spellings. Both carry the
zingolib crate version and the same sixteen-character prefix of the
Published Commit, produced by one source file that zingolib owns, compiled
into `zingolib/build.rs` and, by path, into the workbench. The descriptor
carries the prefix on a release as on any other commit. The release-tag
form `zl_<tag version>` is retired, and `build.rs` no longer reads the tag
at HEAD.

The registry is the one source a consumer reads. `binding-publish` reads
the Android bundle's coordinate from the Maven metadata inside the bundle
and writes it as the manifest annotation `org.zingolabs.zingolib.coordinate`
beside the revision. The descriptor annotation is retired.
`pin-binding-layer` records the commit, each platform's digest and the
coordinate from the manifests it resolves, downloads nothing, and prints
those three. A Maven bundle whose manifest names no coordinate is refused
as published before the annotation. The Bundle Pin refuses at parse a
coordinate whose version does not name the pin's commit.

The repository's record of publications is the Publication Record, keyed
by Published Commit and reconciled against the registry by a required
check. "Manifest" names the registry's OCI image manifest alone.

## Consequences

- The first entry a `zingolib-bundle-pin` of 0.3.0 can pin for Android is
  the next one published, since no earlier manifest carries the coordinate
  annotation. Step 4 of zingolib#2930 needs a new entry regardless.
- A consumer's About screen shows `zl_7.0.0_<sixteen characters>` where it
  showed `zl_7.0.0_<five>` or `zl_7.0.0`.
- The workbench tool `binding-manifest` and the check "Binding Layer
  manifest" carry a name the glossary now reserves for the registry's
  object. Their rename is a later change of its own, since branch
  protection names the check.
- Record zingolib/0054's statement that a checkout with tags at depth one
  names its release through the descriptor is superseded by this record:
  the descriptor names the commit, and the release stays readable in git.
- Record zingolib/0057's sentence on where the fetch reads the coordinate
  is amended: the pin holds it from the manifest, and the fetch verifies the
  extracted layout against the pin.

## Appendix

### References

- zingolib#3030 implements the coordinate in the Bundle Pin and, at
  6bdb2b9e1, the manifest annotation.
- zingolib#2930 holds the plan this record serves. Ruling 3 there is
  amended to the manifest annotation.
- The glossary entries Published Commit, Descriptor, Coordinate and
  Publication Record in `zingolib/CONTEXT.md` were ruled with this record.
- Mechanisms weighed for the one abbreviation rule: a shared source file
  (chosen), a new in-repo crate (rejected, since a path-only
  build-dependency would break `cargo publish` of zingolib or add a
  crates.io package), and two producers with a build-time cross-check
  (rejected as a second rule dressed as a check).

### Definitions

- The Published Commit is the zingolib commit a Prebuilt Bundle was built
  from.
- The descriptor is the display string `get_version` returns, which a
  consumer shows beside its own.
- The coordinate is the Maven `group:artifact:version` of the Android
  Prebuilt Bundle.
- The OCI (Open Container Initiative) image manifest is the registry's
  record of one bundle: its layer, its digest and its annotations.
- The Publication Record is `bindings/published.toml`.
