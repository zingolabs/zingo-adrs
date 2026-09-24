# Releases senesce by chain height instead of a remote kill switch

## Status

proposed

Drafted as 0004 in a maintainer's worktree and never committed. It took
0011 on import because 0004 was already taken.

The Least Authority audit (Suggestion 1) recommended a forced-update gate: a remotely controlled
minimum supported version that blocks usage of deprecated builds. We adopted the threat but rejected
the remote control. Each release bakes an estimated release height at prep time, and the app measures
its own age against the Zcash chain tip using the existing block-spacing math. At 54 days past the
release height the app shows a warning banner, at 67 days a launch interstitial, and at 81 days the
build is senescent: it refuses to attach to any server and launches into the existing Offline mode,
with the wallet readable, keys exportable, and a store link presented. Recovery is an in-place store
update; key export is the fallback, never the primary path. This is the zebrad end-of-support model
(warn at 91 days, refuse to run at 105) adapted to a wallet: a wallet must degrade rather than die,
so senescence removes network attach instead of halting the process.

The chain is the clock. The gate compares the wallet's last-known synced height against the baked
height, reads no device clock, and performs no runtime fetch of any policy value. A hard gate applies
at launch when the last-known height is already past drop-dead; a session whose sync crosses the
threshold finishes with an interstitial and gates at next launch, which grants a long-absent user one
grace session. Mainnet only; testnet and regtest never gate; `__DEV__` builds skip the gate so old
checkouts do not senesce on emulators. `release-prep.mjs` stamps the height (fetched from the hosh
registry at prep time, human-run and committed) into a shared const; the app must never fetch a
minimum version at runtime, because that would recreate the remote kill switch through the back door
and make the registry operator a party who can gate the fleet.

## Considered Options

- A signed remote manifest (minimum version, not-after timestamp, pinned Ed25519 key): rejected as
  the primary mechanism. It can accelerate a deadline in an emergency, but it requires a signing-key
  ceremony, a perpetual hosting commitment, and a trusted holder of a fleet-wide kill switch, and its
  fail-open/fail-closed dilemma has no clean answer. It remains compatible as a later addition on top
  of senescence.
- Wall-clock age measurement: rejected; device clocks are user-settable and the chain height is
  already synced, already proof-of-work-weighted, and suppressing it also visibly stalls sync.
- Hard lockout on senescence: rejected; a self-custody wallet must never stand between a user and
  their keys, so the gate removes network attach and nothing else.

## Consequences

- The 81-day lifetime binds the release cadence: the longest observed production gap is 52 days, so
  a slip approaching the lifetime pushes users of the previous release toward the gate. A release-age
  tripwire (alarm when the latest production tag passes 30 days) is recommended alongside.
- Betas ship far inside the window and effectively never gate; production carries the policy.
- The senescence module exposes state only (`fresh | warning | final | senescent` plus the estimated
  drop-dead date); all copy is translation keys rendered at the display edge, per ADR-0002.
