# One route tree chosen by the session

Date: 2026-10-08

## Status

proposed

Ruled in a planning session on 2026-10-08, pending review. The session
walked every navigation call in the app and every surface that behaves as
a screen.

## Context

The app reaches its screens through four mechanisms at once.

- A JavaScript stack at the root holds `LoadingApp` and `LoadedApp`. Each
  one leaves the other with a `reset` that carries seven positional
  parameters. The biometric lock is a reset to `LoadingApp` with a flag.
- `LoadingApp` holds a `screen` field in its state and renders one of
  eight onboarding screens from a chain of conditions. These screens have
  no back action, no typed parameters, and a `serverReturn` field that
  remembers where the server screen came from.
- `LoadedApp` declares thirty routes on a native stack inside its
  `render`, each as a render callback that closes over the class. A
  `useEffect` inside one of these callbacks captures the navigation
  object into a class field, which is how the options panel opens a
  screen.
- The home tabs add and remove the `Send` tab from the tree as the
  wallet's mode and server change.

Five routes receive functions as parameters: the two scanners, the address
list, and the confirmation screen. A function parameter cannot be
serialized, which rules out a linking configuration, state persistence,
and the static configuration API. The scanners live at the root because
of it.

The `zcash:` link handler lives in `LoadedApp`. A link that arrives before
the wallet opens waits until that component mounts. Hardware back is
handled in seven places, one of them a root listener that consumes every
press. Seven migration screens each reset the stack to home on their own.
Two parallel enums name the routes. One screen, `NewSeed`, lost its mount
point when the onboarding motion landed.

Every screen under `screens/` and `ui/` is a function component.
`LoadedApp` and `LoadingApp` are classes, and a separate effort
(`ADR-0017`) plans to replace their state with a controller machine. The
route tree is independent of that effort and lands first.

## Decision

One route tree, declared in `app/navigation/`, holds every route of the
app. It is the only place that declares a route. A global `RootParamList`
types the tree.

A session value picks the section the root shows: `boot`, `locked`,
`onboarding`, or `wallet`. The root renders the section for the current
value. Changing the value unmounts the section left behind, as the
reset did. The session value is the only way across sections.

Route parameters are serializable data. A screen that produces a result
for the screen that opened it receives the opener's route name as
`returnTo` and returns with `popTo(returnTo, params, { merge: true })`.
Actions come from hooks that read the wallet context today and may read
a store later. Every parameter is data.

A surface is a route when it survives a navigation, is reachable by link
or notification, or owns a back action. Every other surface is a sheet.
The options panel, the confirmation sheet, the add-tag sheet, the seed
sheet, and the delete sheet are sheets.

The tree:

```
Root  (native stack)
├─ Boot          session boot        start gate, then open the wallet
├─ Lock          session locked      the declined gate, retry returns to boot
├─ Onboarding    session onboarding  custom navigator over StackRouter, OnboardingStage as its view
│    Welcome · ImportChooser · ImportWallet · Server · ServerList {chain}
│    Progress {kind} · OpenError {kind, details}
├─ Wallet        session wallet      native stack, the options panel as its layout
│    ├─ Home (bottom tabs)            History · Send (remote server, spendable wallet) · Receive
│    ├─ pushed group                  Settings · Server · About · MixnetDoctor · Rescan · Insight
│    │                                SyncReport · Pools · AddressBook · AddressList {addressKind, returnTo}
│    │                                ValueTransferDetail · Messages · Seed {action} · Ufvk {action}
│    │                                Confirm {calculatedFee, proposalPools, sendAllAmount} · Computing {end?}
│    ├─ one-way group (no gesture)    MeetIronwood · MigrationStrategy · MigrationTransactions
│    │                                MigrationSending {transactions} · MigrationSplitPlan
│    │                                MigrationSplitting {plan?} · MigrationCadence
│    │                                MigrationSchedule {perBucket} · MigrationStatus
│    │                                MigrationBatchSending {denominations?}
│    └─ overlay group (transparent)   SeedBackup {from}
└─ scanners group (transparent modal, above every section's sheet portal)
     ScannerAddress {returnTo, field?, raw?} · ScannerUfvk {returnTo}
```

Onboarding is a custom navigator built with `useNavigationBuilder` and
`StackRouter`. Its view is the existing `OnboardingStage`, which picks
each enter and exit animation from the pair of routes it moves between
and keeps the welcome branches mounted underneath. A native stack removes
a popped view before an exit animation can run. The custom navigator
keeps the leaving view until its animation ends.

Renames: `StartMenu` becomes `Welcome`, `ImportUfvk` becomes
`ImportWallet` (it restores from a seed or a viewing key), `WalletProgress`
becomes `Progress`, `WalletError` becomes `OpenError`, `HomeStack` becomes
`Home`. `NewSeed` and `ScreenEnum` are deleted. `RouteEnum` is retired
when the tree moves to the static configuration API, and literal unions
replace it.

A `linking` configuration owns the `zcash:` link. A link that arrives
during boot or lock is kept and delivered to `Send` when the wallet
section mounts. A link that arrives during onboarding is dropped.

Hardware back at the root consumes the press. The app closes when the
user swipes it away, as any other app. A screen that must block back
does it with its own listener, as the migration screens do.

The work lands in six pull requests, each one green on its own: the root
on a native stack (open as zingo-mobile #1572), the four-state session
with `Boot` and `Lock`, the onboarding navigator, the wallet navigator
with component-only routes, serializable parameters, and the static
configuration API.

## Considered options

A native stack for the onboarding section was rejected. The designer's
motion depends on exit animations that a native stack cuts short.

Scanners inside each section were rejected. The bottom-sheet portal of a
section renders above a scanner mounted inside it, and the onboarding
navigator can only push plain views.

Keeping the route tree inside `LoadedApp` and `LoadingApp` until the
controller machine of `ADR-0017` lands was rejected. The tree only needs
screens that read a context, which the classes already provide. Landing
it first leaves `ADR-0017` a thinner pair of components to replace.

Letting hardware back close the app from home was the agent's
recommendation. The maintainer rejected it.

Keeping the JSX declaration instead of the static configuration API was
rejected. The static API infers the parameter types and the linking
configuration from the one declaration, which is the point of a single
tree.

## Consequences

`LoadedApp` and `LoadingApp` become providers for their sections and mount
one navigator each. Everything they pass to screens through render
callbacks moves into the context they already provide. Screens read
actions through thin hooks, which is the one seam the later store
migration needs.

zingo-mobile #1280 rebases onto this tree. The controller machine of
`ADR-0017` replaces the providers' state and leaves the routes untouched.

Maestro flows select by test ID and are unaffected by the renames. The
navigation mocks under `__mocks__/@react-navigation/` shrink as screen
tests render inside a real container.
