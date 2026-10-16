---
status: proposed
---

# Chain tests run on the mock chain; live networks are testnet cells

zingolib's chain-bound tests ran against a regtest validator and indexer
that `zcash_local_net` launched, at minutes per test and a container stack
per run, and the harness's own behaviour (a launch block, a replay by
reorg competition, a convergence barrier) leaked into what the tests
pinned. We decided to delete regtest from zingolib entirely: the
`libtonode-tests` chain-bound tests, the `zingolib_testutils` crate, the
`zcash_local_net` dependency, the chain caches, the container job and the
`hierarchy-test` libtonode phase. Every guarantee those tests carried
takes exactly one of four dispositions. A guarantee the mock chain can
carry becomes a twin on `MockNet`, which runs in the ordinary unit suite
on every pull request. A guarantee that needs a real network becomes a
testnet cell behind the non-default `testnet` feature, compiled on every
pull request by the feature sweep and run only against a
zingolabs-operated zainod testnet node on a tagged release, serving TLS,
with NU6.3 active; until that node exists no CI profile enables the
feature. A guarantee about the indexer rather than the wallet is exported
to zaino, and its deletion here waits on its landing there. A guarantee
about the regtest harness itself, or about zebra's subsidy, is retired.

## Considered options

Keeping regtest as a nightly tier was rejected: a suite nothing runs per
pull request rots silently, as the gated `unit_test_twins` suite did
across the Ironwood era flip in 2026-07, and the harness behaviour it
pinned is not the wallet's.

Freezing the mock chain's validation at consensus bookkeeping (branch id,
expiry, anchors, nullifiers, inputs, maturity, fee) was rejected. It
would leave every send with a "the validator accepted these bytes"
residue and one testnet cell per transaction shape. Instead `validate`
grows to verify proofs, spend-authorization and binding signatures, and
null-data standardness, using the `orchard`, `sapling-crypto` and
`zcash_proofs` crates already in the lock file, so the residue the mock
cannot carry is network policy and a real indexer's behaviour, nothing
structural.

Staging the testnet cells in a file cargo does not see was rejected for
the same rot reason; a feature gate keeps them type-correct at zero run
cost.

## Consequences

A twin carries every guarantee of the chain-bound test it replaces and
may sharpen but never narrow; it lands in the commit that deletes the
test. The unit of the work is the guarantee, recorded in zingolib's
`docs/testing/regtest-removal-ledger.md`; the execution order is in
`docs/testing/regtest-port-plan.md`. The mock chain inherits the
pinned-binary blind spot ADR 0009 describes, so one testnet cell is the
ZIP 318 enforcement canary, and ADR 0009 is amended with it. ADR 0003
(test-owned chain caches) is superseded when the caches are deleted.
`zcash_local_net`, `zingo-consensus`, `toml_edit` and `toml_write` leave
the dependency graph; a workspace dependency diet is a separate follow-up.
The `*_mobileclient` scenarios zingo-mobile consumes and the
`ClientBuilder` zaino consumes go with `zingolib_testutils`, so the final
deletion waits until both repositories have moved off them. No new
dependency is added anywhere in this work.
