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
| 0018 | [The served block representation is a choice of index set](zaino/0018-served-block-representation-is-an-index-set-choice.md) | proposed |
| 0019 | [Crate boundaries follow an independent-variation criterion](zaino/0019-crate-boundaries-follow-independent-variation.md) | proposed |
| 0020 | [Errors keep their cause typed; a variant exists because a caller branches on it](zaino/0020-errors-keep-their-cause-typed.md) | proposed |

### Repo-scoped records: zingolib

| Number | Record | Status |
| --- | --- | --- |
| 0001 | [LightClient starts offline by default](zingolib/0001-offline-by-default.md) | accepted |
| 0002 | [Regtest support is compiled out of production builds](zingolib/0002-regtest-compiled-out-of-production.md) | accepted |
| 0003 | [Test-owned chain caches snapshot completed setup and record returned outputs](zingolib/0003-test-owned-chain-caches.md) | accepted |
| 0005 | [Retire the darkside-tests crate and the orphaned zingo-testutils crate](zingolib/0005-retire-darkside-tests-and-zingo-testutils.md) | accepted |
| 0006 | [The pending proposal lives in the wallet; offline signing reads it](zingolib/0006-wallet-stored-proposal.md) | accepted |
| 0007 | [Library Birthday: a release-stamped floor for never-online wallet creation](zingolib/0007-library-birthday.md) | accepted |
| 0008 | [Offline-signed transactions get their expiry by proposal retarget, not a builder override](zingolib/0008-offline-expiry-by-retarget.md) | accepted |
| 0009 | [Tests run Ironwood-era by default; Orchard-era behavior is opt-in](zingolib/0009-tests-run-ironwood-era-by-default.md) | accepted |
| 0011 | [IP obfuscation for Transmission and price-fetch runs over the Nym mixnet](zingolib/0011-nym-mixnet-transmission.md) | accepted |
| 0014 | [Pool Activation is derived once, in pepper-sync, from zcash_protocol parameters](zingolib/0014-pool-activation-derived-in-pepper-sync.md) | accepted |
| 0015 | [ADR 0015: Landing in dev ships the wallet file format](zingolib/0015-landing-in-dev-ships-the-wallet-file-format.md) | accepted |
| 0016 | [Note-splitting execution is a stateless, send-shaped call](zingolib/0016-note-splitting-is-a-stateless-fused-call.md) | accepted |
| 0017 | [Phase 2 parts are due for the whole open window, and the random target is advisory](zingolib/0017-phase-2-parts-due-for-the-whole-open-window.md) | accepted |
| 0018 | [A part's anchor boundary is drawn at an age of one bucket or more, never at the window it broadcasts in](zingolib/0018-part-anchors-are-drawn-at-age-one-or-more.md) | accepted |
| 0019 | [Immediate migration (Drain) is exposed as a send-shaped call](zingolib/0019-immediate-migration-is-send-shaped.md) | accepted |
| 0020 | [Migration logic delegates to zcash_pool_migration](zingolib/0020-migration-logic-delegates-to-zcash-pool-migration.md) | accepted |
| 0021 | [The mixnet shim's TLS verifies against the compiled-in webpki bundle](zingolib/0021-shim-tls-verifies-against-the-webpki-bundle.md) | accepted |
| 0022 | [A Broadcast Witness is never the sync indexer](zingolib/0022-broadcast-witness-never-the-sync-indexer.md) | accepted |
| 0023 | [Top-window sync rides the mixnet](zingolib/0023-top-window-sync-rides-the-mixnet.md) | accepted |
| 0024 | [24. Consumers converge on a zingolib-owned mixnet surface](zingolib/0024-consumers-converge-on-a-zingolib-owned-mixnet-surface.md) | proposed |
| 0025 | [25. Going online requires Connectivity Consent](zingolib/0025-going-online-requires-connectivity-consent.md) | proposed |
| 0026 | [26. Mixnet capability compiles by default; activation stays a runtime consent](zingolib/0026-mixnet-capability-compiles-by-default.md) | proposed |
| 0028 | [28. The reference consumer lives in-repo in an excluded sub-workspace](zingolib/0028-the-reference-consumer-lives-in-repo-in-an-excluded-sub-workspace.md) | proposed |
| 0029 | [29. A maintained mixnet indexer pool replaces server selection](zingolib/0029-a-maintained-mixnet-indexer-pool-replaces-server-selection.md) | superseded by [zingolib/0038](zingolib/0038-an-exit-node-reservation-is-unique-to-its-holder.md) |
| 0030 | [The CLI crosses sync to async exactly once, dispatching from a static command table](zingolib/0030-the-cli-crosses-sync-to-async-exactly-once.md) | accepted |
| 0031 | [CLI stdout carries only the result; failures and narration travel on stderr, and a failed one-shot exits nonzero](zingolib/0031-cli-stdout-carries-only-the-result.md) | accepted |
| 0032 | [`network off` is zero-emission teardown, and `--offline` suppresses the network surface](zingolib/0032-network-off-is-zero-emission-teardown.md) | accepted |
| 0033 | [Network access demands a consent-tiered Network Observer](zingolib/0033-network-access-demands-a-consent-tiered-network-observer.md) | accepted |
| 0034 | [Server selection is a mixnet liveness sweep](zingolib/0034-server-selection-is-a-mixnet-liveness-sweep.md) | proposed |
| 0035 | [The acquisition race speaks the literature's vocabulary](zingolib/0035-the-acquisition-race-speaks-the-literatures-vocabulary.md) | proposed |
| 0036 | [A Destination, not a witness, receives a Transmission](zingolib/0036-a-destination-not-a-witness-receives-a-transmission.md) | proposed |
| 0037 | [Broadcast means many recipients](zingolib/0037-broadcast-means-many-recipients.md) | proposed |
| 0038 | [An Exit Node Reservation is unique to its holder](zingolib/0038-an-exit-node-reservation-is-unique-to-its-holder.md) | proposed |
| 0039 | [An Exit Node is Exclusive to one Destination or Shared across many](zingolib/0039-an-exit-node-is-exclusive-or-shared-across-destinations.md) | proposed |
| 0040 | [Send's escalation is a hedged race of full paths](zingolib/0040-sends-escalation-is-a-hedged-race-of-full-paths.md) | proposed |
| 0041 | [A platform-typed Mixnet Session acquires every transport](zingolib/0041-a-platform-typed-mixnet-session-acquires-every-transport.md) | proposed |
| 0042 | [A Destination Reservation levels concurrent Transmissions across operators](zingolib/0042-a-destination-reservation-levels-concurrent-transmissions.md) | proposed |
| 0043 | [A survey proves its exit with a Sentinel, and restarts when the proof fails](zingolib/0043-a-survey-proves-its-exit-with-a-sentinel.md) | proposed |
| 0044 | [A client proves its exit before its first role, and later clients trust fresh proof](zingolib/0044-a-client-proves-its-exit-before-its-first-role.md) | proposed |
| 0045 | [Boot proves four exits and assigns them by role](zingolib/0045-boot-proves-four-exits-and-assigns-them-by-role.md) | proposed |
| 0046 | [The wallet asks for a mixnet conduit by role](zingolib/0046-the-wallet-asks-for-a-mixnet-conduit-by-role.md) | proposed |
| 0047 | [Roles key conduits, and the wallet stops naming exits](zingolib/0047-roles-key-conduits-and-the-wallet-stops-naming-exits.md) | proposed |
| 0048 | [A mobile session rotates one client rather than separating roles](zingolib/0048-a-mobile-session-rotates-one-client.md) | proposed |
| 0049 | [Every transport comes from a long-lived host](zingolib/0049-every-transport-comes-from-a-long-lived-host.md) | proposed |
| 0050 | [Indexers are classified by role, trust, and location, and one broadcast rule draws from them](zingolib/0050-indexers-are-classified-by-role-trust-and-location.md) | proposed |
| 0051 | [Continuous sync keeps the wallet current as new blocks are mined](zingolib/0051-continuous-sync.md) | accepted |
| 0052 | [Indexerless operations are pure functions; effects live at the edges](zingolib/0052-pure-core-effects-at-edges.md) | superseded by [zingolib/0006](zingolib/0006-wallet-stored-proposal.md) |
| 0053 | [Destinations are drawn from the indexer registry per chain, on every transport](zingolib/0053-destinations-are-drawn-from-the-registry-per-chain-on-every-transport.md) | superseded by [zingolib/0050](zingolib/0050-indexers-are-classified-by-role-trust-and-location.md) |
| 0054 | [The Binding Layer lives beside the surface it wraps](zingolib/0054-the-binding-layer-lives-beside-the-surface-it-wraps.md) | proposed |
| 0055 | [zingolib admits Swift, Kotlin, and TypeScript with their native tooling](zingolib/0055-zingolib-admits-swift-kotlin-and-typescript-with-native-tooling.md) | proposed |

<!-- records-index:end -->
