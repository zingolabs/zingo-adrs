# zingod is the only consumer of zaino's library crates; in-process embedding is unsupported

## Status

proposed

## Context and decision

zallet embeds zaino in its own process. It constructs `NodeBackedChainIndex`
directly, runs it in ephemeral mode so that no persistent index is opened, and
reads finalised data through zebra's `ReadStateService` or through the
JSON-RPC interface. Several parts of zaino exist only to serve that embedder:
the split of the chain-index interface into `ChainIndex` and
`ChainIndexRpcExt`, the public re-exports of `zaino-state`, the ephemeral
finalised-state backend, and the rule that zaino defers to a TLS
cryptography provider an embedder installed first.

zaino's core product is a lightwalletd server, zingod, which builds its
indexes from zebra over zebra's JSON-RPC interface. Its secondary product is
the set of non-standard indexes, such as block-explorer indexes, that zingod
also serves. Neither product needs an embedder, and the embedding surface
costs a second contract on every change to the chain index.

We decide that zingod is the only consumer of zaino's library crates. Zaino
no longer supports in-process embedding, and zallet support ends. A program
that needs zaino's indexes runs zingod and talks to it over gRPC or JSON-RPC.

## Consequences

- Every workspace crate except zingod is marked `publish = false`, and the
  publishable set shrinks to zingod.
- Library items narrow to the smallest visibility that zingod's build
  accepts; `pub` survives only where zingod crosses a crate boundary.
- `ChainIndexRpcExt` merges back into one chain-index trait, and the
  `zaino-state` re-exports that existed for embedders are deleted.
- The ephemeral finalised-state backend is deleted. zingod does not bind its
  client listeners until its index is synced, so nothing needs a passthrough.
- zallet must either run zingod as a separate service or pin a zaino release
  that predates this record. The zallet maintainers learn of this decision
  before the change that removes the embedding surface merges.
