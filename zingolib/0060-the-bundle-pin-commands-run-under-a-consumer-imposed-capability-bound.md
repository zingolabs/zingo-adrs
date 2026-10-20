# The Bundle Pin commands run under a capability bound the consumer imposes

## Status

proposed

Accepted when the zingolib pull request that releases `zingolib-bundle-pin`
0.4.0 merges; from then on the package extracts in process and the publisher
packs every Prebuilt Bundle in the pax tar format.

## Context

zingo-mobile/0021 runs the app's npm dependencies under Hardened JavaScript
and names the code it leaves open: code that runs at build time with full
Node access. Step 4 of zingolib#2930 adds the first zingolabs-authored
build-time Node package to zingo-mobile, `zingolib-bundle-pin`, whose
`fetch-binding-layer` runs inside Gradle, inside an Xcode build phase and in
every consuming CI job, and whose `pin-binding-layer` rewrites the Bundle
Pin. kitchen-sync already runs both. A compromised release of the package
would run with the same access as the build that invokes it: the signing
keystore beside the Android project, and the token files a CI job holds.

Hardened JavaScript cannot reach this code. SES lockdown freezes the
intrinsics of the app bundle on Hermes, and the install-script allowlist
governs lifecycle scripts, of which the package has none. Node 22.13 made
its permission model stable: `--permission` denies the file system, child
processes, worker threads and native addons unless an `--allow-*` flag opens
each one. It does not bound the network.

The package spawned `tar` to extract both bundles. Under the permission
model a child inherits no bound, so the one flag that admits `tar` admits
every process a compromised package would start.

## Decision

The consumer bounds the commands; the package never bounds itself. Each
consumer holds two yarn scripts in its `package.json`, one per command,
whose text the package's README gives verbatim, and Gradle, Xcode and CI
run the commands only through those scripts. Each script runs Node with
`--permission` and opens exactly what the command needs: the package's own
directory, the pin file, the fetch store of each platform and the iOS
extraction root, and for the pin command the pin file and its partial
sibling, the partial for reading as well, since a rename reads its source. Nothing is opened with a wildcard,
and no script opens child processes, workers or addons. The scripts invoke
the command's script file, not the `node_modules/.bin` symlink, since the
permission model follows symlinks. Two rules of the model, found by probe on Node 22.18, shape the lines. An
opened directory covers its children and never itself, so everything the
package removes, renames or writes sits one level under an opened
directory: the iOS root holds the package, its partial sibling during a
swap, and the digest marker, and the Xcode path does not move. The model
resolves every grant when Node starts, and a path that does not exist yet
is granted as one file, so the fetch script creates the three directories
it opens before Node starts.

The package extracts in process. `node:zlib` inflates the archive and a
tar reader of the package's own, since the package has no dependency, writes
regular files and directories and refuses every other entry: symbolic and
hard links, absolute paths, and paths that leave the extraction root. The
reader understands the ustar name split and PAX extended headers, and the
publisher packs both bundles with `--format=pax`, the spelling GNU tar and
bsdtar share, so GNU long-name entries are a refusal and not a code path.

The package pins the claim. A test in `bindings/pin` reads the two script
lines out of the README and runs each real command under them against a
fixture root whose store already holds a bundle, so no network is touched,
and asserts success; a second run drops one opened path and asserts the
refusal. The package's engines floor is Node 22.13.0, the first release with
the stable flag, and every consumer pins one Node version in `.nvmrc`.

## Considered options

Granting `--allow-child-process` and keeping the `tar` spawn was rejected:
it leaves the bound with a hole the size of the threat the bound addresses.

Extraction by the caller, Gradle's `tarTree` and a shell `tar` in Xcode and
CI, was rejected: the marker and coordinate checks would split between the
package and three call sites, against the ruling of zingolib#2930 that the
scripts live in one place.

Spelling the flags at every call site was rejected: six copies per
consumer drift independently, and one widened copy drops the bound
silently.

Letting the package publish its own flag list was rejected: the package is
the threat.

Running the commands under SES `lockdown()` or a LavaMoat policy was
rejected: lockdown hardens intrinsics against other packages, and this
package has none; a policy needs Compartments, which bound a package's
reach only through the module graph, while the permission model bounds the
process.

## Consequences

The bound covers what a compromised release can read and write on the build
machine. It does not cover the network: the package fetches from one
constant host, and a compromised release could send what it can read, which
is the pin file and the bundle. It does not cover anything a consumer runs
outside the two scripts.

`zingolib-bundle-pin` 0.4.0 fetches no entry published before the format
pin, as 0.2.0 fetched no bare-AAR entry: entries packed by GNU tar in its
default format hold long-name entries for the Maven paths that exceed 100 characters. The first
`/publish` after the release yields the first entry every consumer can
fetch at 0.4.0. kitchen-sync moves to 0.4.0 directly and skips the 0.3.0
follow-up; zingo-mobile starts on 0.4.0 with the bound in its first commit.

The deferred decision of zingolib#2930 that Gradle, Xcode and CI call the
commands from `node_modules` is amended: they call the consumer's two yarn
scripts. zingo-mobile/0020 commits zingo-mobile to manage its bundles as
kitchen-sync does, so the bound reaches zingo-mobile through that record;
zingo-mobile/0021 names this record as its first build-time case.

A release that reaches one directory further fails the package's own test
before it publishes, and fails a consumer's build with Node's
`ERR_ACCESS_DENIED` naming the path if the README and the consumer drift.

## Appendix

### References

- zingo-mobile/0021 records Hardened JavaScript for the app bundle and lists
  build-time Node code as out of scope and open.
- zingolib#2930 holds the submodule-deletion plan; its deferred decision on
  sharing the fetch scripts and its step 4 are amended by this record.
- zingolib#3030 releases `zingolib-bundle-pin` 0.3.0; zingolib#3035 implements
  this record as 0.4.0, and the record is accepted when it merges.
- zingolib/0057 defines the Maven repository layout whose paths exceed the
  ustar name field.
- Node's permission model: https://nodejs.org/docs/latest-v22.x/api/permissions.html

### Definitions

- The permission model is Node's process-wide allowlist of file-system
  paths, child processes, workers and addons, enabled by `--permission`.
- ustar is the POSIX tar header, with a 100-character name field and a
  155-character prefix field; PAX extended headers carry longer paths as
  `path=` records; GNU tar's default format carries them instead in a
  long-name entry of type `L`.
- A partial sibling is the `<path>.partial` the package writes and renames
  into place so that a reader never sees a half-written path.
