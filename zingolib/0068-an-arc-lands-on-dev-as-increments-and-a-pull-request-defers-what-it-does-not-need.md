# An arc lands on dev as increments, and a pull request defers what it does not need

## Status

accepted

Ruled in session 2026-11-24. The work is tracked in zingolib#3222.

This record amends one consequence of
[ADR-0064](0064-releases-are-cut-from-dev.md): the sentence that sends a
portion of a larger arc to an integration branch. The rest of that record
stands, and this record supersedes nothing.

## Context

ADR-0064 gave `dev` the promise that `stable` had carried: `dev` can be
released on any day. Its consequence was that a pull request merges only
when the work it describes is complete, and that a portion of a larger
arc goes to an integration branch. The release procedure lists what an
incomplete portion looks like: a dead code path, a half-migrated type, a
disabled test, or a feature behind an unfinished flag.

Read broadly, "half-migrated type" covers any migration of a family of
items that is not finished. Under that reading almost no part of a large
change can land alone, and the author has two choices. The first is one
large pull request. zingolib#3212 took it: 2,422 added lines across 61
files, typed as a refactor, carrying three changes of behaviour that a
review found only after the whole was written. The second is an
integration branch, which is itself a divergence. Work collects beside
`dev`, other contributors build on a `dev` that lacks it, and the merge
at the end is a diff that nobody reads line by line.

A second rule pushed the same way. The guidance for agents, ruled
2026-10-02, said to collapse any duplication found in code a change
already touches. An agent that fixed a bug in a function therefore also
extracted the helper beside it and changed every caller, in the pull
request that held the fix.

Of the last 60 merges into `dev` before this record, ten added more than
450 lines of code. Most contributors now work through agents, which
write a large diff as readily as a small one, so the size of a pull
request is set by its instructions.

## Decision

A commit of `dev` is releasable when every public item works completely
and no item exists in two forms. A struct that carries an old field
beside its replacement fails, as does a function kept beside its
successor with no caller, a method that is declared and unimplemented, a
disabled test, or a flag that gates unfinished code. Sibling items in
different styles pass: one method of a trait may return a domain type
while the others still return generated types.

An arc is a change too large for one pull request. It lands on `dev` as
increments. An increment is one pull request that makes one semantic
change and leaves `dev` releasable in the sense above, and it merges as
soon as it is ready. A tracking issue, opened before the first increment,
states the end state of the arc and lists its increments, and every
increment names it. The issue is what makes an uneven public surface on
`dev` acceptable: it says which increment makes the surface uniform.

An integration branch remains for the arc that has no increment able to
hold the guarantee. A person opens it, never an agent on its own
judgement, and it merges `dev` into itself daily.

A pull request defers what it does not need. It fixes what a CI gate
rejects and what sits on a line the task must rewrite. Every other
defect its author finds becomes a follow-up, an issue that names the item,
states the defect and the condition that closes it, and links the pull
request that found it.

A follow-up describes the default branch. It names the item as `dev`
holds it, so the issue reads true to someone who never saw the pull
request, and it is still true after the pull request merges or closes. A
defect that exists only on the branch of an open pull request is never a
follow-up.

Feedback that blocks a pull request is never filed as an issue. A
reviewer who finds a defect the pull request introduces, or any reason it
should not merge as it is, writes a review comment on the pull request,
and the author resolves it there. An issue is for work that can wait
until after the merge.

The rule to collapse duplication in touched code
is withdrawn for duplication that existed before the change. The
duplication gate still rejects a new group.

The tests that decide whether a pull request makes one semantic change,
the count of added lines above which an author must state why it cannot
be split, and the check that enforces the count are specified in
`AGENTS.md` and the workbench of zingolib. This record governs them and
does not repeat them.

## Considered options

A limit on the size of a pull request was rejected. zingolib#3212 was
hard to review because it mixed a refactor with changes of behaviour, and
a rename across 332 files is large and easy to review. Size is a signal
that asks for a reason, and coherence is the rule.

The broad reading of "half-migrated type" was rejected for the cost
above. It makes the integration branch the ordinary route for a large
change, and the stated purpose of many small pull requests into `dev` is
that the work of independent contributors does not diverge.

Keeping the rule to collapse duplication on contact was rejected. It
makes the size of a fix depend on the debt of the file it lands in, and
it hides a refactor inside a pull request typed as a fix. A filed
follow-up keeps the debt visible and gives its removal a review of its
own.

Filing every review finding as an issue was rejected. An issue that
holds a blocking finding lets the pull request merge with the finding
open, and an issue written against a branch describes code the default
branch may never hold. The review comment keeps a blocking finding beside
the lines it concerns.

Letting an agent file follow-ups without asking was rejected. An issue
publishes under the developer's account, so the agent drafts and the
developer grants.

## Consequences

`dev` can show a public surface in two styles while an arc is in
progress. A reader who finds one should look for the tracking issue
before concluding that the bar on `dev` was missed.

Review moves to the increment. The reviewer of an increment reads the
tracking issue to learn where the arc is going, and judges the increment
on whether it is whole.

Debt that a change once removed in passing now waits in the tracker
under the `follow-up` label. The count of open follow-ups measures
whether deferral is paying its way, and a queue that only grows is the
signal to schedule the work.

The release procedure of zingolib,
[docs/sop/SOP001-release.md](https://github.com/zingolabs/zingolib/blob/dev/docs/sop/SOP001-release.md),
states the earlier rule in its bar for `dev`. The pull request that
writes the guidance for agents amends it.

## Appendix

### References

- zingolib#3222 tracks the arc that puts this record into effect.
- zingolib#3212 is the pull request whose review prompted the rule, and
  zingolib#3219 is the first increment split out of it.
- zingolib/0064 gives `dev` the promise this record interprets.
