# How zingo-adrs relates to a code repository

The [README](../README.md#pointing-a-code-repository-at-zingo-adrs) is the
single statement of how a code repository such as zingolib holds zingo-adrs as
a submodule at `docs/adr/`, and it gives the commands to read the records and
to advance the pin. This howto only walks a reader from a code repository
through what that arrangement looks like from the other side.

A citation of `docs/adr/zingolib/0011-nym-mixnet-transmission.md` in zingolib
source names the file `zingolib/0011-nym-mixnet-transmission.md` in this
repository. The path resolves only in a checkout whose submodule is
initialised, so a fresh clone shows an empty `docs/adr/`. A GitHub diff that
adds the submodule shows `docs/adr` as a single link to the pinned commit, and
the files it replaces as deleted, although the records now live here.
