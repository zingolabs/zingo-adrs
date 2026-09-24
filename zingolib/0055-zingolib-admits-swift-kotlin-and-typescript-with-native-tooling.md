# zingolib admits Swift, Kotlin, and TypeScript with their native tooling

## Status

proposed

Ruled in a grilling session on 2026-09-23, pending review.

## Context

This repository has been Rust only. Committed tooling was Rust in the
workbench crate, and a new language needed explicit consent. ADR 0028
admitted TypeScript, but confined it to `zingo-tauri/`, and a Python
binding test was removed in favour of a Rust round trip. ADR 0054 moves the Binding Layer
here, and that layer is Swift and Kotlin by nature, with packaging that
only Gradle and SwiftPM produce idiomatically. zingo-mobile's Node build
scripts cannot move unchanged under the old rule.

## Decision

zingolib admits Swift, Kotlin, and TypeScript. Rust code and Rust builds
stay in the Rust workbench. Swift, Kotlin, and TypeScript use their own
modern tooling: SwiftPM for Swift, Gradle with the Kotlin DSL for Kotlin
and Android packaging, and the ecosystem's standard toolchain for
TypeScript. zingo-mobile's `.mjs` build scripts do not move; the workbench
and the native tools replace them. Python and Bash remain barred as
committed files, apart from small task glue.

This widens ADR 0028. That record confined TypeScript to `zingo-tauri/`,
and this record admits it anywhere in the repository, with Swift and
Kotlin beside it.

## Consequences

When this record is accepted, ADR 0028's status gains a sentence saying
that this record lifts its confinement of TypeScript.

Contributors to the Binding Layer need the Android SDK and NDK, and a
macOS host for the XCFramework. The Rust-only workspace remains buildable
without either.
