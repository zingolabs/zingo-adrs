# Releases are cut from dev, and the stable branch is retired

## Status

accepted

Closes the question zingolib/0062 left open under zingolib#2988. The
`stable` branch of this repository was its own decision, older than the
org-scoped record
[003](../003-zaino-branching-versioning-and-release-strategy.md), which
describes the same model for zaino. This record retires the branch here
and supersedes nothing.

## Context

Every crate release up to zingolib 7.0.0 was cut on `stable`, a branch
that took merges from `dev` alone and was meant to hold release-quality
code while `dev` moved. The model was this repository's own, in use since
at least zingolib#825 in March 2024, and record 003 of February 2026
describes the same model for zaino without being its source here. It
bought one thing: a fix for the last release could ship without the
half-finished work on `dev`.

It cost more than it bought. A consumer that waited on `stable` waited on
a back-port of every fix it needed, and by the time zingolib#2985 merged
the two branches they had diverged by 410 commits. zingolib/0062 found
that no product of the project depends on `stable`: zingo-mobile and
kitchen-sync take the Binding Layer by Published Commit of `dev`,
`zingo-cli` builds in the workspace, and crates.io has one external
dependent. zingolib 8.0.0 was cut from `dev` on 2026-10-28, the first
release made without the branch.

The release procedure itself lived nowhere in writing. Each release was a
pull request per crate group, a hand-ordered publish, and a verbal rule
about candidates.

## Decision

`dev` is the one long-lived branch, and every release is cut from it.

A release is prepared by one pull request into `dev` that bumps every
crate of the release set and cuts its changelogs, and the merge commit is
the first release candidate, tagged on the lead crate. When a candidate
needs a fix, a short-lived release line `release/<lead crate>_vX.Y.Z`
branches from the candidate's tag, holds the fixes and the later
candidates, and merges back into `dev` after each. The release is tagged
on the commit of the last candidate, so the bytes a consumer tested are
the bytes that ship. Every release, a patch included, has a line of its
own, and a line is deleted once its release is published.

The candidate becomes the release at zingolib's discretion, on the
evidence a tracking issue per cycle collects. Consumer reports are input
to that judgement and no consumer holds a veto.

The procedure, the tag and branch names, the checks, and the bar `dev` has
to hold are specified in
[docs/sop/SOP001-release.md](https://github.com/zingolabs/zingolib/blob/dev/docs/sop/SOP001-release.md)
of zingolib, which this record governs and does not repeat.

The per-crate semantic versioning, the pre-1.0 softening and the Keep a
Changelog practice of this repository continue unchanged. They match what
record 003 writes for zaino.

## Considered options

Keeping `stable` was rejected for the cost above: a second branch whose
only consumer was the back-port that fed it.

The changeset-derived pipeline zaino adopted in zaino/0016, where CI
derives versions, changelogs and tags from per-change files through four
gated branches, was not taken. This repository releases eight crates on an
irregular cadence by hand, and the pipeline's machinery is sized for a
problem it does not have yet. Nothing here forecloses it.

Publishing the merge commit of a release line into `dev`, instead of the
candidate's own commit, was rejected: the merge commit holds everything
`dev` gained since the candidate, which the line exists to exclude.

## Consequences

`dev` carries the stability promise `stable` carried. A pull request
merges into `dev` only when the work it describes is complete, and a
portion of a larger arc goes to an integration branch that merges when the
arc is whole. The specification states the rule.

The publication record of the Binding Layer assumed a linear history on
`dev` and refuses a release line candidate. zingolib#3113 amends it so an
entry may descend from any earlier entry. The procedure does not function
for a candidate that needed a fix until that lands. The amendment is owed
under either model: a `stable` hotfix back-ported into `dev` is the same
shape, a commit `dev` reaches that the last entry does not precede, so
keeping `stable` would not have spared it.

The push-triggered workflows run on `dev` and `release/**`, and the
Binding Layer crates carry `publish = false` so one workspace publish
covers the set.

The crates.io publish stays a maintainer's local step with signed tags as
its record, until zingolib#3115 moves it into a workflow under Trusted
Publishing.

## Appendix

### References

- zingolib#3114 lands the specification and the workflow changes.
- zingolib#2988 audits the crates on crates.io and held the question of
  whether `stable` continues.
- zingolib/0062 makes the Prebuilt Bundles the release channel of the
  Binding Layer and finds that no product depends on `stable`.
- zingolib/0015 ships a wallet file format the moment it lands in `dev`,
  which the new bar on `dev` makes harder, not easier.
- zaino/0016 is the pipeline this record considered and did not take.
