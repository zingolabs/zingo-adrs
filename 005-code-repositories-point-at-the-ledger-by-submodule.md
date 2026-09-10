# Code repositories point at the ledger by submodule, and hold none of its content

## Status

accepted

Supersedes [004](004-decision-records-live-in-the-org-ledger.md) on the
mirroring mechanism only; every other decision in 004 stands.

## Context and decision

Record 004 chose `git subtree` to give each code repository a copy of this
ledger. A subtree's defining property is that it copies the remote content
into the host repository's tree and history, so every record, and the
ledger's own tooling, would be checked into every code repository and
re-copied on every refresh. That contradicts the ledger being the unique
home of decision state: the same text would exist in as many histories as
there are consumers, and a contributor could edit a copy and open a pull
request against it.

A code repository instead holds a **git submodule** at its records path
(`docs/adr/` for zaino). The only thing checked in is the gitlink, a commit
hash naming one ledger commit, plus the `.gitmodules` entry that says where
the ledger lives. The records themselves are materialised locally by
`git submodule update --init` and never enter the code repository's history.
Advancing the pointer is a deliberate commit that changes one hash.

## Alternatives rejected

- **`git subtree`**, as 004 chose. Rejected for copying content into every
  consumer's history.
- **A hand-rolled pointer file plus a fetch tool.** A metadata file holding
  the hash, a `.gitignore` line for the clone, and a workbench binary that
  materialises it. This reinvents the submodule and makes every reader
  depend on the tool instead of one git command.
- **Bare links to GitHub.** No pointer at all, so a code repository could
  not say which ledger state it was written against.

## Consequences

- A fresh clone of a code repository has an empty `docs/adr/` until the
  reader runs `git submodule update --init`; GitHub renders the directory as
  a link to the pinned ledger commit.
- CI in a code repository that does not check out submodules sees an empty
  directory, which is fine because nothing there gates on the records.
- The ledger's README replaces its subtree instructions with submodule ones.
