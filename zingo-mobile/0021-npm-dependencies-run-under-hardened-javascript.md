# npm dependencies run under Hardened JavaScript, with pinned host globals and no install scripts

Date: 2026-10-16

## Status

accepted, amended 2026-11-08

Ruled in a session on 2026-10-15 and 2026-10-16. Implemented in
zingo-mobile pull requests #1642, #1646 and #1652, stacked on
`feat/lavamoat`, tracked in zingo-mobile issue #1641.

Amended in a session on 2026-11-08: decision 3 changes the pin from a
data property to an accessor, prototype hardening in both builds becomes
the fourth step of the arc, and strict mode for every module becomes the
fifth, tracked in zingo-mobile issue #1887.

## Context

The keys live in Rust. The JavaScript side still sees the destination
address, the amount, the clipboard, and the seed on the backup screen.
A compromised npm package anywhere in the tree can swap an address or
read the seed without touching zingolib. The `chalk` and `debug`
compromise of September 2025 did that in browsers by replacing `fetch`
and hooking wallet APIs on the global object.

On the date of this record `node_modules` holds 1,080 packages. Three
ship an install script, all of them tooling: `esbuild`, `unrs-resolver`,
`@parcel/watcher`. No application code mutates a builtin prototype or
calls `eval`.

The app runs on Hermes. Hermes has no direct `eval` and no `with`
statement, so it cannot host SES Compartments (facebook/hermes#957).
`lockdown()` works on Hermes through `ses/hermes`. MetaMask Mobile runs
it in production on the same stack: React Native, Hermes, Reanimated.

`lockdown()` freezes the language intrinsics. It does not freeze
`globalThis`, and `fetch`, `XMLHttpRequest`, `WebSocket` and the timers
are React Native's globals, installed by InitializeCore as lazy getters,
not the language's.

## Decision

Dependencies get the least trust the runtime can enforce. Three rules.

1. Dependency lifecycle scripts are off. `.yarnrc` carries
   `ignore-scripts true`; `yarn setup` installs with the frozen lockfile
   and runs `@lavamoat/allow-scripts`, which runs the packages allowed
   under `lavamoat.allowScripts` in `package.json` and the repository's
   own `postinstall`. The allowlist starts empty.
   `@lavamoat/preinstall-always-fail` fails the install if the `.yarnrc`
   entry is lost.

2. The bundle runs under Hardened JavaScript.
   `@lavamoat/react-native-lockdown` is the Metro serializer. The SES
   shim is the first polyfill; after InitializeCore and before the entry
   file, `hardenIntrinsics()` freezes the intrinsics. The lockdown
   options are the package's: error and console taming off, override
   taming severe. Babel leaves the shim untouched.

3. The host globals are pinned. The first import of `index.js` reads
   every host global that `setUpXHR` and `setUpTimers` install and
   redefines each as a non-configurable accessor on `globalThis`. The
   getter returns the value React Native installed. The setter throws a
   `TypeError` that names the global, in debug and release alike. The
   read is on the cold-boot path under a budget of 5 ms on a low-end
   Android device, measured once. Bindings only: the objects behind them
   stay writable until the fourth step.

4. The fourth step hardens `XMLHttpRequest.prototype`,
   `WebSocket.prototype` and the fetch classes with `harden`, in debug
   and release alike. The in-app Network Inspector of the dev menu,
   which rewrites `XMLHttpRequest.prototype` when toggled, is given up.

## Considered options

Endo used directly was rejected. LavaMoat is Endo's SES plus the Metro
and install integration we would otherwise write ourselves. The rest of
Endo serves code we do not run: third-party plugins and distributed
capabilities.

Per-package isolation with Compartments is not available. Hermes lacks
the language features, and the Hermes maintainers plan strict-mode local
`eval` only. The `policy.json` half of LavaMoat does not apply here.

Hardening the prototypes behind the host globals, in release only, was
rejected. The debug network inspector patches `XMLHttpRequest.prototype`,
so release would behave differently from debug with no device run to
back it. Hardening them in both builds was accepted on 2026-11-08, at the
cost of that inspector.

A data-property pin, non-writable and non-configurable, was the first
form of decision 3 and was rejected on 2026-11-08. The React Native
Babel preset emits sloppy-mode modules, where a write to a non-writable
data property completes as a silent no-op. SES removes the same no-op
for the intrinsics by turning their properties into throwing accessors
under `overrideTaming: 'severe'`, and the pin takes the same form.

Strict mode for every module, which makes every write to a non-writable
property throw, was deferred to zingo-mobile issue #1887 as the fifth
step. It changes the meaning of sloppy code across the dependency tree
and needs its own device run.

Freezing `globalThis` was rejected. React Native defines some globals
after the entry file.

Running `lockdown()` under Jest was rejected. Tests spy on globals such
as `Date.now`, which a frozen intrinsic refuses.

## Consequences

`yarn` alone leaves `patch-package` unapplied. CI and the quickstarts
use `yarn setup`. A new dependency with an install script fails
`allow-scripts` until someone allows it on purpose.

A dependency that patches an intrinsic fails at startup with
`TypeError: Cannot assign to read only property` naming the module. The
fix is to drop the package or to add its shim to `getPolyfills`, before
the freeze. A dependency that writes to a pinned global fails at the write with a
`TypeError` naming the global, in debug and release alike. A dependency
that adds a new name to a frozen intrinsic from sloppy-mode code still
completes as a silent no-op until the fifth step.

React Native bumps gain a gate. `react-native-lockdown` is untested on
Static Hermes, and React Native 0.82 ships Hermes V1. A bump may wait on
LavaMoat.

What a compromised dependency keeps: direct access to whatever it can
reach, including `NativeModules`, `AsyncStorage`, the clipboard and the
network. Lockdown removes the indirect path through the language
builtins, not the direct one.

Out of scope and open: code that runs at build time with full Node
access (Babel plugins, Metro transformers, `patch-package`, Storybook),
the native dependency graphs (Gradle, CocoaPods, Rust crates), and the
UI and worklet runtimes of Reanimated, which hold their own `globalThis`
and are not hardened.
zingolib/0060 takes the first build-time case: the Bundle Pin commands run
under a capability bound the consumer imposes through Node's permission
model.

The three pull requests passed the Android emulator integration tests
and the iOS build in CI. Debug and release on a device had not been
exercised on the date of this record.
