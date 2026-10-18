# One route tree chosen by the session

Date: 2026-10-08

## Status

accepted

Ruled in a planning session on 2026-10-08. The session walked every
navigation call in the app and every surface that behaves as a screen.
Accepted on 2026-10-18, when zingo-mobile#1637 merged into `dev` as
`4ceb8f217`. The move to the static configuration API was dropped on the
way; the decision below records why.

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

Three routes receive functions as parameters: the scanner, the address
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
for the screen that opened it receives a request name in its params and
delivers the result on a channel keyed by that name; the opener
subscribes to it while mounted. An answer nobody listens for is dropped,
so it can never reach a later opener. Actions come from hooks that read
the wallet context today and may read a store later. Every parameter is
data.

A surface is a route when it survives a navigation, is reachable by link
or notification, or owns a back action. Every other surface is a sheet.
The options panel, the confirmation sheet, the add-tag sheet, the seed
sheet, and the delete sheet are sheets.

The tree:

```
Root  (native stack)
├─ Loading       session boot, locked or onboarding; one provider holds the boot state
│    ├─ Boot          session boot        start gate, then open the wallet
│    ├─ Lock          session locked      the declined gate, retry returns to boot
│    └─ Onboarding    session onboarding  custom navigator over StackRouter, OnboardingStage as its view
│         Welcome · ImportChooser · ImportWallet · Server · ServerList {chain}
│         Progress {kind} · OpenError
├─ Wallet        session wallet      native stack; the options panel fills Home
│    ├─ Home (bottom tabs)            History · Send (remote server, spendable wallet) · Receive
│    ├─ pushed group                  Settings · Server · About · MixnetDoctor · Insight
│    │                                SyncReport · Pools · AddressBook · AddressList {addressKind, request}
│    │                                ValueTransferDetail · Messages · WalletSeed {action}
│    │                                Confirm {calculatedFee, proposalPools, sendAllAmount, sendPageState}
│    │                                Computing {end?}
│    ├─ one-way group (no gesture)    MeetIronwood · MigrationStrategy · MigrationTransactions
│    │                                MigrationSending {transactions} · MigrationSplitPlan
│    │                                MigrationSplitting {plan?} · MigrationCadence
│    │                                MigrationSchedule {perBucket} · MigrationStatus
│    │                                MigrationBatchSending {denominations?}
│    └─ overlay group (transparent)   SeedBackup {entry} · ViewingKey {entry, switchTo?}
└─ Scanner {request, accepts, texts, origin?}   transparent modal, above every section's sheet portal
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
`Home`. The one scanner that reads addresses, seed phrases and viewing keys
is `Scanner`; its `accepts` function becomes a `kind` the scanner checks.
`NewSeed`, `ScreenEnum` and `RouteEnum` are deleted. `Route` is a const
object whose type is the literal union of its values. The header reads
its own route instead of a screen marker.

The tree stays a JSX declaration. The onboarding opens on `Welcome` or on
`OpenError` depending on the session, and the static configuration API
has no per-mount initial route, so that one navigator would have stayed
dynamic inside a static tree. The global `RootParamList` gives the typed
`useNavigation` the static API was wanted for.

The session owns the `zcash:` link. A link that arrives during boot or
lock is kept and delivered to `Send` when the wallet section mounts. A
link that arrives during onboarding is dropped. A view-only wallet, or
one on no server, has no `Send` tab, and says so instead.

Hardware back at the root consumes the press. The app closes when the
user swipes it away, as any other app. A screen that must block back
does it with its own listener, as the migration screens do.

The work landed on an integration branch through one pull request per
step, each green on its own: the session with `Boot` and `Lock`
(zingo-mobile#1602), the onboarding navigator (#1603), the wallet
navigator with component-only routes (#1626), serializable parameters
and the result channel (#1627), the options panel under the screens it
opens (#1628), and the loading section, `Route` and the typed hooks
(#1634). It reached `dev` as zingo-mobile#1637.

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

Moving the tree to the static configuration API was planned and then
dropped. It would have inferred the parameter types from the one
declaration, but the onboarding's start route depends on the session,
which static configuration cannot express per mount. The global
`RootParamList` already types the hooks.

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
