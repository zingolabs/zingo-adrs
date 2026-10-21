# The Prebuilt Bundles are the release channel of the Binding Layer

## Status

proposed

Accepted when the zingo-mobile pull request of zingolib#2930 step 4 merges
with its Bundle Pin on a Published Commit of `dev` that is not a `stable`
release; that merge retires ruling 10 of zingolib#2930.

## Context

The zingolib workspace publishes seven crates to crates.io: `zingolib`,
`pepper-sync`, `zingo-netutils`, `zingo-net-diag`, `zingo-memo`,
`zingo-status` and `zingo-price`. Each release is cut on `stable` and
published from there; the newest is zingolib 7.0.0, cut on 2026-10-15 at
bfb0a2156. zingolib#2988 audited the seven crates on that day. One crate
outside zingolabs depends on `zingolib`, `zexcavator` 0.0.3, which requires
`zingolib ^0.0.1`; no crate on crates.io depends on 7.0.0 or on any 6.x
release, and the downloads of the seven crates are the project's own CI and
developer builds. The audit ruled that the project minimizes its investment
in crates.io in every way possible.

The Binding Layer crates are not published to crates.io. Since the 2026-10-12
amendment of zingolib/0054, zingolib builds them and publishes each commit a
consumer pins as two Prebuilt Bundles in the GitHub Container Registry,
tagged by the commit, recorded in the Publication Record
`bindings/published.toml`, and kept forever. zingolib/0059 makes the
Published Commit the one identity of a Prebuilt Bundle: the registry tag,
the Maven coordinate and the descriptor all name the commit, and the version
in the coordinate is the crate version at the commit, not a version anyone
published. zingolib#3046 publishes a commit of `dev` by dispatch at the
commit's own tag.

The Publication Record holds three entries, ba7336710, 502a453fe and
d4ac5772d. Every one is a commit of `dev`, and none is a release. kitchen-sync
pins the second and moves to the third. zingo-mobile still pins zingolib as a
git submodule at bfb0a2156, the 7.0.0 release on `stable`, and ruling 10 of
zingolib#2930 holds its step 4 until the lock and the submodule name one
commit, which the ruling expected to be a `stable` release. The second
2026-10-15 amendment of zingolib#2833 found that wait tied to the release
cadence of crates.io and asked for it to be re-examined before step 4. The
`stable` branch and `dev` diverged by 410 commits before zingolib#2985 merged
them, so a consumer that waited on `stable` waited on a back-port of every
fix it needed.

## Decision

The Prebuilt Bundles are the release channel of the Binding Layer. A
Published Commit is a release of the Binding Layer, and a consumer takes the
Binding Layer only by pinning a Published Commit in its Bundle Pin. A
consumer never pins a crate version, a `stable` commit as such, a git tag or
a crates.io release.

A commit is eligible for publication because it is a commit of `dev`, and
for no other reason. Membership of `stable` and publication to crates.io
confer nothing on a commit. The version in a Maven coordinate is the crate
version at the commit and carries no release meaning; the sixteen commit
characters beside it are the release.

No step of a consumer's plan waits on a crates.io publish or on a `stable`
release. Ruling 10 of zingolib#2930 is amended: its wait on a `stable`
release is retired, and step 4 sets zingo-mobile's lock and its submodule to
one Published Commit of `dev`, the newest entry at the time of the change.

A Published Commit stays a release for as long as the registry holds it,
which under zingolib/0054 is forever, so a consumer's pin never dangles and a
build of any past commit of a consumer reproduces its Binding Layer. The
changelog section of each entry in `bindings/CHANGELOG.md`, generated from
the audited crates' changelogs since the previous entry, is the release note
of the channel.

This record rules nothing on the seven crates' own publication to crates.io
or on the `stable` branch. It rules that the Binding Layer, and every
consumer of it, depends on neither.

## Considered options

Publishing only `stable` releases, as ruling 10 of zingolib#2930 assumed,
was rejected: it ties every mobile fix to the cadence of a channel with one
external consumer, and it makes each fix a back-port before it is a
publication.

Giving the Binding Layer versions of its own, by semantic version tags on
the bundles, was rejected: zingolib/0059 made the Published Commit the one
identity, a version would be a second name for the same bytes, and a
consumer that pulls by digest reads no version.

Publishing every push to `dev` was rejected by decision 4 of zingolib#2833:
zingolib publishes only the commits a consumer pins, since each publication
costs a build of both platforms and a permanent registry entry.

Publishing the Binding Layer crates to crates.io and building them in the
consumer was rejected by the 2026-10-12 amendment of zingolib/0054: a
consumer builds no Rust.

## Consequences

A consumer's zingolib is named by a commit and nowhere by a version. The
descriptor, the coordinate and the lock all read the commit, and a release
version of zingolib appears in no consumer.

Ruling 10 of zingolib#2930 no longer waits. Step 4 may start on the third
entry, d4ac5772d, and sets the lock and the submodule to it in one change.

A Binding Layer change reaches a consumer by publishing its commit and
moving the pin. The third publication took thirteen minutes from its request
to the recording commit.

The `stable` branch and the seven crates.io releases serve no product of the
project. Whether they continue is the question of zingolib#2988, which this
record leaves open; it removes the Binding Layer from the reasons to answer
yes.

Every Published Commit is kept forever, so the registry grows with each
publication. Decision 4 of zingolib#2833 bounds that growth to the commits
consumers pin.

## Appendix

### References

- zingolib#2988 audits the seven crates on crates.io and rules that the
  project minimizes its investment there.
- zingolib#2833 holds the publication plan; its decision 4 bounds the
  publications, and its 2026-10-15 amendments asked for this re-examination.
- zingolib#2930 holds the submodule-deletion plan; its ruling 10 is amended
  by this record.
- zingolib/0054, amended 2026-10-12, moves the Binding Layer's build into
  zingolib and keeps every published bundle.
- zingolib/0059 makes the Published Commit the identity of a Prebuilt Bundle.
- zingolib#3046 publishes a commit of `dev` by dispatch at the commit's own
  tag.
- zingo-mobile/0016 pins zingolib as a submodule; zingo-mobile/0020 commits
  zingo-mobile to the Prebuilt Bundles.

### Definitions

- A release channel is the path by which a consumer takes a product: here
  the Prebuilt Bundles, pulled by digest from the registry at a Published
  Commit.
- The Published Commit is the zingolib commit a Prebuilt Bundle was built
  from, as zingolib/0059 defines it.
- The Publication Record is `bindings/published.toml`, one entry per
  Published Commit, append-only.
