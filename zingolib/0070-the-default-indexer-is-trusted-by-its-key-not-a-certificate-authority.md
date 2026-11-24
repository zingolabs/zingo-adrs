# The Default Indexer is trusted by its key, and no certificate authority is trusted at all

## Status

proposed

Drafted 2026-10-21 after the eMudhra incident of 2026-10-20 and ruled in
session the same day. The rulings: the zec.rocks fleet is excluded from
the pinned host set from the beginning; a pin mismatch hard-fails the
connection; no backup key exists yet, so generating one is a precondition
of shipping the pin; and every certificate authority operator is assumed
malicious.

Amends [ADR-0021](0021-shim-tls-verifies-against-the-webpki-bundle.md),
which keeps its verifier for the hosts this record does not pin and gains
an implementation deadline for its update policy. Builds on
[ADR-0063](0063-a-session-attaches-to-one-indexer.md), which makes the
zingolabs endpoint the Default Indexer.

## Context

On 2026-10-20 eight intermediate certificates issued by eMudhra Limited were
found to carry private keys with between three and eight bits of entropy.
Anyone could derive those keys and issue browser-trusted certificates for
any host. Browsers blocked the eight intermediates within a day through
their revocation feeds, Firefox's OneCRL and Chrome's CRLSet. No root was
removed, because the four emSign roots the intermediates chain to are not
themselves compromised.

The wallet verifies every TLS connection against the whole Mozilla root
bundle and nothing else. The wallet-to-indexer channel calls
`ClientTlsConfig::new().with_webpki_roots()`, the SOCKS5 tunnel of mixnet
transmission does the same, and the nym stack verifies through the
ADR-0021 stand-in over `webpki-root-certs`. The bundle the lockfile pins
carries the four emSign roots. The wallet has no revocation feed, so it
keeps trusting the eight intermediates until Mozilla removes the roots and
the crate re-pins. ADR-0021 ratified a nightly staleness alarm and a
distrust alarm that would catch that re-pin; neither exists in the
workbench or the workflows.

The threat model this record adopts is the one the maintainer ruled: every
certificate authority operator is malicious. Under that model a
certificate chain proves nothing, an issuer restriction protects nothing,
and a revocation feed arrives from a party that is itself the adversary.
The only trust the wallet may place is in a public key it carries, whose
private half is held by a party the wallet already trusts for the
connection's purpose. Two such parties exist. zingolabs holds the private
key of the Default Indexer at `mainnet.zainod.zingolabs.dev`. Nym holds
the ed25519 signing keys of its API and the ed25519 identity keys of its
gateways, and the locked nym stack at 1.21.6 ships
`nym-directory-attestation`, whose `SignedDigestSnapshot::verify` checks a
directory snapshot against a caller-supplied set of trusted signer keys.

Two facts bound the implementation. `tonic` 0.14.6 exposes no hook for a
custom certificate verifier; its `ClientTlsConfig` accepts only root
certificates, so a verifier that inspects public keys must be installed
through a hyper-rustls connector handed to `connect_with_connector`. And
the nym gateway registration handshake already authenticates the gateway
by its ed25519 identity, so on that path only the directory fetch from
`validator.nymtech.net` rides bare HTTPS today.

## Decision

**The Default Indexer is authenticated by its public key.** The wallet
carries the SHA-256 of the Default Indexer's SubjectPublicKeyInfo and the
SHA-256 of a backup key. A connection to the Default Indexer succeeds only
when the leaf certificate's key matches one of the two hashes, the leaf
covers the host name and the current time, and the handshake signature
verifies under that key. No root store takes part: the pinned key is the
trust anchor, so a certificate from any authority, legitimate or
compromised, is accepted only if it carries our key and rejected
otherwise. The verifier is a `ServerCertVerifier` implementation installed
through a hyper-rustls connector, because tonic offers no other seam. The
pin table is compiled in beside the Default Indexer's declaration and keyed
by host name, and the test that pins the Default Indexer's URI also pins
its presence in the table.

**A mismatch hard-fails.** A connection whose leaf matches neither pin is
a typed connection error the session reports as such, and the wallet never
speaks to the endpoint. There is no report-only mode and no override.

**The backup key is generated before the pin ships.** No backup key exists
today. Shipping a single pin would lock every released wallet out of the
Default Indexer the moment its key is lost or rotated, until a release
carries a new pin. The operator therefore generates the backup key pair
offline, stores the private half offline, and hands only the public hash to
the release. Rotation is a two-release procedure: a release adds the next
key as the backup, the server key rotates to the former backup, and the
following release retires the old key. The pin values are confirmed by the
operator from the server's private key, never taken from a network
observation, which the adversary could have forged.

**The nym directory leaves TLS out of its trust path.** The wallet
carries the ed25519 keys of the trusted nym API signers, obtained from Nym
out of band. Exit-node discovery fetches the described nodes as it does
today, then fetches the signed digest snapshot, verifies it under the
carried signer keys, and accepts a node only when its identity is bound in
the verified snapshot. The HTTPS to `validator.nymtech.net` is transport
only; a forged certificate for that host yields a topology the wallet
rejects. The gateway handshake then authenticates the node by the same
identity. Until this step lands the directory fetch stays exposed, and
this record names that exposure rather than hiding it.

**The zec.rocks fleet is excluded from the beginning**, and so is every
other host zingolabs does not operate. Their operators may rotate keys or
issuers without notice, and a pin on such a host breaks users at the
operator's convenience, not ours. A user who chooses an indexer chooses
its operator, and the wallet verifies that host as it does today, against
the ADR-0021 bundle. Any later widening of the pinned set is an amendment
to this record.

**ADR-0021's update policy is implemented, not merely ratified**, because
it is the only defense the unpinned hosts have. The nightly workbench job
gains the staleness alarm and the distrust alarm that record describes,
and a compiled-in list of distrusted intermediates, by SHA-256 of the
certificate, that the bundle verifier rejects, with the nightly job diffing
the list against Mozilla's OneCRL and opening an issue on a new entry. The
eight eMudhra intermediates are the list's first entries. This is
defense in depth for hosts the user chose; it is no defense under the
adopted threat model, and the record does not claim otherwise.

## Consequences

No certificate authority can impersonate the Default Indexer to this
wallet. Impersonation needs the Default Indexer's private key or the
backup's, both held by zingolabs. The Default Indexer cannot rotate its
key without a release that carries the next pin first, and the two-release
procedure above is the operating rule for whoever holds the key. Loss of
both private keys locks every released wallet out of the Default Indexer
until a release carries new pins, which is why the backup is generated
offline before the first pin ships.

No certificate authority can feed this wallet a forged mixnet topology
once the signed snapshot lands. Nym's signer keys are a second compiled-in
pin with a second rotation dependency, on Nym rather than on zingolabs,
and the record accepts that dependency because the alternative is to trust
a certificate authority.

A user who chooses a third-party indexer is protected only as well as the
Mozilla bundle protects them, and the wallet says so nowhere today. A
follow-up may surface the distinction in the session's indexer report.

## Considered options

**Restrict the pinned hosts to Let's Encrypt's roots.** Rejected by the
threat-model ruling. Let's Encrypt is a certificate authority operator.

**Pin the TLS key of `validator.nymtech.net`.** Rejected. Nym holds that
key and renews its certificate on its own schedule; a pin would break the
mixnet at every rotation that changes the key. The signed snapshot pins a
key Nym publishes for exactly this purpose.

**Pin the whole maintained pool, zec.rocks included.** Rejected by ruling
on 2026-10-21.

**Certificate Transparency verification in the client.** Rejected. rustls
has no SCT verification, and a log is operated by parties the threat model
does not trust.

**Platform trust stores with their revocation feeds.** Rejected for the
reasons ADR-0021 and zingo-mobile/0006 record, and because a platform
store is a larger set of the authorities this record distrusts.
