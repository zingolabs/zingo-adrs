# The Regtest chain type ships unconditionally and carries the mock chain

## Status

accepted

## Context

zingolib/0002 gated `ChainType::Regtest` and the `ActivationHeights`
vocabulary behind a default-off `regtest` cargo feature, so that a shipped
wallet carried no regtest code path and no activation-schedule dependency.
Commit b9e98eae5 of zingolib ("remove zebra and clean up zingo config",
2026-02-17) deleted the feature from `zingolib/Cargo.toml`, and no manifest
in the workspace has declared it since. `ChainType::Regtest(ActivationHeights)`
compiles into every build, `ActivationHeights` comes from
`zingo_common_components`, the `cargo tree` tripwire that 0002 describes
exists in no workflow, and the record still read `accepted`.

The regtest harness left separately. zingolib#3000 ("test!: remove regtest
from zingolib", merged 2026-10-16 as 90e22c941) deleted `zcash_local_net`,
`zingolib_testutils` and every test that drove a regtest validator, under
zingolib/0058. The chain type did not leave with it: the mock chain of 0058
is `MockChain`, whose `chain_type` is `ChainType::Regtest(activation_heights)`,
because `Regtest` is the one `ChainType` whose `Parameters` implementation
reads a caller-supplied activation schedule. Mainnet and Testnet carry
zcash_protocol's fixed schedules, so a chain mined from height 0 on either
would reach Orchard after more than a million blocks, and no test could
defer NU6.3 or NU7. Twelve mock tests set a schedule of their own through
`MockChain::with_activation_heights`. The name follows upstream:
zcash_protocol's `NetworkType::Regtest` is its one network with free
activation heights, and the wallet file's chain tag `2`, the `uregtest`
address prefix of the golden fixtures and the FFI's `regtest:<schedule>`
hint all encode it.

## Decision

`ChainType::Regtest(ActivationHeights)` ships in every build of zingolib,
zingo-cli and the Binding Layer, ungated. It is the network of the mock
chain and of the offline synthetic wallets, and it remains a chain a
consumer may select by name. Its removal or renaming is deferred (ruled
2026-10-22); any such change gets a record of its own.

This record supersedes zingolib/0002, whose feature boundary no longer
exists and whose consequences (a build that refuses chain tag `2`, a
production address parser that never tries `Regtest`, a release tripwire)
no longer hold.

## Considered Options

Reinstating the `regtest` feature was rejected for now: the variant is a
test-time necessity for the mock chain, which runs on every pull request
under the default feature set, so a gate would have to be implied by
`testutils` again and would protect only the release artifacts, where the
cost of the variant is one enum arm and one schedule struct.

Renaming the variant, so that the word "regtest" names nothing in a
repository that has removed the regtest harness, was deferred: the wallet
file's chain tag, the address prefix of the golden fixtures, the CLI's
`--chain` vocabulary and the FFI hint all spell it, and each is a
compatibility surface.

## Consequences

- A build without the feature no longer exists, so a wallet file tagged
  `2` opens in every build, and every address parser may try `Regtest`.
- The mock chain, the synthetic-wallet builder and the golden fixtures
  depend on the variant, and a future removal must give them another chain
  type with a caller-supplied schedule.
- The "regtest" word in zingolib names the chain type alone; the harness,
  its caches and its launch block are gone, and the test glossary in
  zingolib's `docs/testing/CONTEXT.md` no longer defines them.
