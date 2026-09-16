# Errors keep their cause typed; a variant exists because a caller branches on it

## Status

proposed

## Context

The workspace handles a failure in four different ways:

- `expect` or `unwrap` in production code, which panics;
- a wildcard match arm that drops the error;
- `map_err` into a `String` field, which loses the cause's type and its
  `source()` chain;
- a typed variant that holds the cause with `#[from]` or `#[source]`.

A rough count finds about 240 stringifying `map_err` sites and about 42
`#[from]` or `#[source]` sites (zingolabs/zaino#1575).

Only the last form keeps the information that a caller needs. ADR-0008 makes
retry policy a function of the error type. That works only while the type
reaches the caller. A cause that is flattened to a message cannot be matched,
cannot be walked with `source()`, and cannot be told apart from a different
failure that has the same text.

`#[from]` has a second problem. It maps a source error to one variant at every
call site. The same source error can mean different things at different
sites. A parse failure on user input is a rejected request. The same parse
failure on data from a validator that runs in the same process is corrupt
source data, not a transport fault. A blanket conversion gives both the same
name.

## Decision

Every failure is classified by these four rules, in order.

1. **A failure that no real input can cause is not an error.** Make it
   unrepresentable in the types. Do not give it a variant or a panic. This is
   the rule of ADR-0013 ("no unchecked door"), applied to failure paths.

2. **A failure that a caller branches on gets its own variant.** The caller
   matches the variant, never the message. A failure that no caller branches
   on may go into a category variant. That variant holds the cause as
   `#[source]`.

3. **`From` and `#[from]` are for context-free conversions only.** A
   conversion is context-free when the source error means the same thing at
   every call site. When the meaning depends on the call site, use `map_err`
   to a named variant. The named variant still holds the cause.

4. **Never:**
   - `expect` or `unwrap` outside tests;
   - a wildcard match arm that drops an error;
   - a cause converted to a string (`e.to_string()` into a message field).

A message field is correct when there is no underlying error value, for
example when a validator answers with a refusal. Rule 4 applies to a cause
that is an error value.

## Considered options

- **Stringify at crate boundaries, keep types inside a crate.** Rejected: a
  crate boundary is where a caller in another crate needs to branch. A string
  there removes the information at the point where it is used.

- **One `#[from]` for each source error type.** Rejected: it is short, but it
  gives one name to failures that mean different things (rule 3). The
  classification then moves into prose, or into matches on message text.

- **Opaque, `anyhow`-style errors.** Rejected for library crates: they keep
  the `source()` chain but remove the matchable variant that rule 2 requires.
  They stay acceptable at a binary's top level, where no caller branches.

## Consequences

- Error enums grow where callers branch and stay small where callers do not.
  Each new variant needs a caller that matches it.
- `Error::source()` returns the real cause at every level, so logs and
  diagnostics show the full chain.
- The existing sites migrate crate by crate. The first slice is the
  `zaino-source` transport error, which holds a failure mode and a string. It
  gets a typed cause, and a failure mode for invalid data from an in-process
  source. The read-state adapter reports that data as a parse failure, which
  is a transport category.
- A wildcard arm on an error enum needs a justified exception. A crate can
  enforce this with `clippy::wildcard_enum_match_arm`.

## Related

- ADR-0008: source ports separate a domain answer from a transport failure,
  and retry policy follows the failure type.
- ADR-0013: a result type holds its invariant, and no constructor skips the
  check.
- ADR-0014: validator readiness is a lifecycle concern of the runtime, kept
  apart from transient transport failures.
- zingolabs/zaino#1489: the chain-head advance error keeps the transport
  cause as `#[source]`, an early application of rules 2 and 4.
