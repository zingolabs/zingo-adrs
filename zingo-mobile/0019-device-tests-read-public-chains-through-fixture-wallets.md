# Device tests read public chains through fixture wallets

Date: 2026-09-30

## Status

proposed

Ruled in a grilling session on 2026-09-30, pending review. The session
walked all 32 tests of the Rust harnesses one at a time.

## Context

Record `zingo-mobile/0015` keeps three Rust crates in zingo-mobile as
test harnesses: `rust/android`, `rust/ios`, and `rust/zingomobile_utils`.
Each harness test launches a regtest validator and an indexer through
`zingolib_testutils`, runs a funded scenario, and then spawns a script
that runs one test on a device.

The walk measured what each test reads from that chain.

- Seven of the ten Android integration tests, in whole or in part, read
  no chain. They check key derivation, address parsing, the version
  string, a price refusal, and wallet file repair. Each one pays for a
  validator, an indexer, a faucet, and a mined transaction.
- Four Android integration tests read a chain that exists before the
  test starts. None of the ten broadcasts a transaction. The
  transmission rule of `zingolib/0011` refuses a send from a wallet with
  no mixnet.
- The 13 Detox end-to-end specs have been disabled in CI since
  2025-02-27. Several name test IDs that the app no longer has. None
  uses a feature that only Detox provides.
- One spec, `reload_while_tx_pending`, needs a property that only
  regtest gives: a chain that mines no block while the test runs.
- The Send button opens only when the mixnet is ready. A mixnet exit is
  a host on the public internet, and it cannot reach a regtest indexer
  on a CI runner. No test that transmits can use regtest.

## Decision

Device tests read public chains. zingo-mobile launches no chain of its
own for a test.

A test that reads no chain is an offline device test. It runs with an
empty server URI.

The sync test runs on mainnet. Its birthday is the chain tip minus
10,000 blocks. It is a blocking check with one fallback server, and it
requires the wallet height to reach at least the height the server
reported before the sync.

Tests that need funds or a known history read a fixture wallet on
testnet. Three wallets exist.

- The shared fixture wallet has a published seed. Its history holds a
  receipt, a send, a memo-to-self, receipts into Sapling and
  transparent, three self-sends, and a shield. The refusal test, the
  value-transfer test, and the pool-balance test read it. The
  value-transfer test reads the oldest three entries. The pool-balance
  test keeps exact balances.
- The transparent fixture wallet has a published seed and one
  transparent note that nobody shields. The shield-offer flow reads it.
- The spending wallet has a secret seed and a large balance. The
  broadcast tests spend from it in the nightly.

A fixture wallet has a fixed birthday at its first funding height. When
the range from that birthday to the tip passes a budget, a maintainer
refreshes the wallet: a new seed, a new funding, and a new history,
published in one pull request. The provisional budget is 100,000 blocks,
which is about 87 days. A timing run on the emulator sets the final
number.

UI tests are Maestro flows. Detox leaves the repository with its specs,
its configuration, its package, and its two Gradle entries. A UI test
stops before the send gate, and the app carries no test-only switch
that opens the gate. The parts of the old specs that sit behind the
gate move to the broadcast tests, which run on testnet in the nightly
with a ready mixnet.

The three harness crates are deleted when the last test leaves regtest.
That test is the pool-balance test. It needs NU6.3 active on its chain,
and the testnet activation height is 4,134,000.

## Considered options

Regtest with a fresh chain in each CI run was the agent's
recommendation for the sync test and the refusal test. The maintainer
rejected it. It keeps a validator, an indexer, and a Rust harness on the
path of every pull request.

A prebuilt regtest chain, published as an artifact, was rejected. It
removes `cargo` from the test job and keeps the validator and the
indexer that serve the chain.

Moving the harnesses into zingolib was the agent's first recommendation,
made before it read `zingo-mobile/0015`, which had already rejected that
move. This record removes the need for it.

A fixture wallet on mainnet was rejected. It holds real coins behind a
published seed.

A scheduled refund from a faucet wallet was rejected. It adds a secret,
a job that can fail alone, and a transmission path to a blocking check.

A secret seed for the shared fixture wallet was rejected. Fork pull
requests receive no secrets, and the tests that read the wallet are
blocking checks.

Repairing Detox was rejected. Maestro already runs in the nightly, needs
no instrumented build, and covers every action the specs perform. One
spec needs `adb`, and the nightly workflow supplies it in two steps
around a tagged flow.

## Consequences

This reverses the part of `zingo-mobile/0015` that keeps the test
harnesses and `zingomobile_utils` in Rust. That record is still proposed,
and the rest of it stands. The workbench crate remains,
and zingo-mobile still needs a Rust toolchain to build the Binding Layer
and the workbench.

Public servers join the path of every pull request: one mainnet server
with a fallback for the sync test, and one testnet server for the three
fixture tests. An outage of either turns pull requests red.

A fresh regtest chain tracked the pinned zingolib. A change in fee or
pool policy broke the expected values in the bump that caused it. A
fixture wallet holds history from its last refresh, and that signal now
arrives at the next refresh.

A refresh costs about nine transmissions through the mixnet, once per
budget period. Until a tool exists for it, a maintainer performs it by
hand.

Anyone can send coins to a published fixture address or sweep the
wallet. The refusal test tolerates a deposit, because it checks a
minimum balance. The pool-balance test fails on a deposit until the next
refresh. A sweep fails all three fixture tests until the next refresh.

The harness chose the regtest height at which NU6.3 activates. A public
chain activates on its own schedule. The sync test scans
ironwood-activated blocks on mainnet only after height 3,428,143, and
the pool-balance test waits for testnet.

No test controls block production. The pending-transaction test checks
total balances, which a new block leaves unchanged. It detects a lost
pending transaction only in a run where no block arrives first.

Three facts are unmeasured on the date of this record: the duration of a
10,000-block mainnet sync on the emulator, the current testnet tip
against 4,134,000, and whether the mixnet bootstraps on the Android
emulator. The last one blocks every broadcast test.

Three specs are dropped with no replacement: the testnet server change,
the regtest server change, and the sync report's batch arithmetic. The
constant `GlobalConst.blocksPerBatch` goes with the last one.

`zingo-mobile/0015` plans a move of `RustFFITest.kt` into zingolib. This
record leaves that plan unchanged.
