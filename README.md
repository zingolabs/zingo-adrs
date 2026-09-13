# Zingolabs decision records

zingo-adrs holds the architecture decision records for every zingolabs code
repository. "Architecture" reads broadly: any decision that shapes how a
repository is built, tested, or released belongs here, whether it concerns
system structure, process, or infrastructure.

## Scopes

A record's path says which repositories it binds.

- An **org-scoped record** binds every code repository. It lives at the top
  level, numbered `NNN`.
- A **repo-scoped record** binds exactly one code repository. It lives in
  that repository's subdirectory (`zaino/` for zaino), numbered `NNNN`.

Each scope numbers its records in its own sequence. Take the next free number
when proposing; concurrent proposals may collide, so renumber at merge, not
before. A gap in a sequence means a proposal was abandoned or renumbered,
nothing more.

## Citing a record

A bare `ADR-NNNN` cites a record in the citing repository's own scope. A
citation into another scope carries the path: `zaino/0016` names the zaino
record, and `003` names the org-scoped one.

## The append-only rule

Records are append-only. Supersede, never delete or rewrite: a reversed
decision gets a new record, and the old record's status names its successor.
A record documents the decision as made and does not track the code, so
"the code changed" is never a reason to remove one.

## Record shape

A record is a Markdown file named `NNN-kebab-title.md` (or `NNNN-` in a
repository scope) whose first line is a `# ` title. The first line under its
`## Status` heading is exactly one of:

- `proposed`, while the pull request is open;
- `accepted`;
- `superseded by [<citation>](<relative path to the successor>)`.

Prose after that line may narrow a partial supersession, but the line itself
is the whole record's standing. A record states one decision: context,
decision, consequences. Keep it short; full governed behaviour belongs in a
specification in the code repository, referenced from the record.

## Proposing a record

Open a pull request against `dev` in this repository. Records are never
proposed in a code repository: zingo-adrs is the sole write side, and code
repositories carry read-only copies.

## Pointing a code repository at zingo-adrs

A code repository holds zingo-adrs as a git submodule at its records path
(`docs/adr/` for zaino). Only the pointer, one commit hash, is checked in
there; the records never enter that repository's history. To add it:

```sh
git submodule add git@github.com:zingolabs/zingo-adrs.git docs/adr
```

To materialise the records after cloning:

```sh
git submodule update --init docs/adr
```

To advance the pointer to the current `dev` of zingo-adrs, then commit the change:

```sh
git submodule update --remote docs/adr
```

A stale pointer is not a defect, and no code repository gates on it. The
records are read-only from the code repository's side: propose changes here.

## Checking

`tools/workbench` holds `check-records`, which CI runs on every pull request.
It verifies file names, the status vocabulary, that every `superseded by`
link resolves, that numbers are unique within a scope, and that the index
below matches the records. Regenerate the index with:

```sh
cargo run --manifest-path tools/workbench/Cargo.toml -- --write .
```

## Index

<!-- records-index:begin -->
### Org-scoped records

| Number | Record | Status |
| --- | --- | --- |
| 001 | [No upstream types in public APIs](001-no-upstream-types-in-public-apis.md) | accepted |
| 002 | [`client_rpc_test_fixtures` moves to its own repository](002-client-rpc-test-fixtures-own-repo.md) | accepted |
| 003 | [ADR: Branching, Versioning, Documentation, Public Interfaces, and Release Strategy](003-zaino-branching-versioning-and-release-strategy.md) | superseded by [zaino/0016](zaino/0016-changeset-derived-release-pipeline.md) |
| 004 | [Decision records live in zingo-adrs and are mirrored into code repositories by subtree](004-decision-records-live-in-zingo-adrs.md) | superseded by [005](005-code-repositories-point-at-zingo-adrs-by-submodule.md) |
| 005 | [Code repositories point at zingo-adrs by submodule, and hold none of its content](005-code-repositories-point-at-zingo-adrs-by-submodule.md) | accepted |

### Repo-scoped records: zaino

| Number | Record | Status |
| --- | --- | --- |
| 0001 | [`zcashd_support` feature gate](zaino/0001-zcashd-support-feature-gate.md) | superseded by [zaino/0017](zaino/0017-zcashd-support-removed.md) |
| 0002 | [Live tests rejoin the root workspace under a single lock](zaino/0002-live-tests-rejoin-root-workspace.md) | accepted |
| 0003 | [Live-test taxonomy and two-crate split](zaino/0003-live-test-taxonomy-and-two-crate-split.md) | accepted |
| 0004 | [Rename the `integration` live-test partition to `clientless`; single `makers test` front door](zaino/0004-rename-integration-partition-to-clientless.md) | accepted |
| 0005 | [`zcashd_support` is opt-in, not a default feature](zaino/0005-zcashd-support-default-off.md) | superseded by [zaino/0017](zaino/0017-zcashd-support-removed.md) |
| 0006 | [aws-lc-rs is the preferred CryptoProvider; classical key exchange is deprecating](zaino/0006-aws-lc-rs-preferred-crypto-provider.md) | accepted |
| 0007 | [Block persistence is a row-set boundary; the domain block is not a storage value](zaino/0007-block-persistence-is-a-row-set-boundary.md) | accepted |
| 0008 | [Validator access is a set of single-question ports over domain primitives](zaino/0008-source-ports-and-domain-primitives.md) | accepted |
| 0009 | [The served JSON schema lives in `zaino-serve`, beside its only consumer](zaino/0009-served-json-schema-lives-in-zaino-serve.md) | accepted |
| 0010 | [ADR 0010: Mempool subsystem separated into `zaino-mempool` behind ports](zaino/0010-mempool-subsystem-separation.md) | accepted |
| 0011 | [The non-finalised chain head is a self-synchronising subsystem](zaino/0011-chain-head-subsystem-separation.md) | accepted |
| 0012 | [The finalised state is a subsystem behind ports, and its database is one implementation of them](zaino/0012-chain-store-subsystem-separation.md) | accepted |
| 0013 | [Domain quantity types carry invariants on results, not on operators](zaino/0013-quantity-arithmetic-result-types-carry-invariants.md) | proposed |
| 0014 | [Validator readiness is owned by the runtime, not by its source consumers](zaino/0014-validator-readiness-owned-by-runtime.md) | proposed |
| 0015 | [Zaino Release Flow Design](zaino/0015-periodic-release-flow.md) | superseded by [zaino/0016](zaino/0016-changeset-derived-release-pipeline.md) |
| 0016 | [Releases derive from changesets through a four-branch gated pipeline](zaino/0016-changeset-derived-release-pipeline.md) | accepted |
| 0017 | [zcashd support is removed; Zebra is the only backing validator](zaino/0017-zcashd-support-removed.md) | accepted |

<!-- records-index:end -->
