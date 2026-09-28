# How zingo-adrs relates to a code repository

zingo-adrs is the only place where zingolabs decision records are written. A
code repository such as zingolib holds this repository as a git submodule at
`docs/adr/`. The submodule pins one zingo-adrs commit and copies no records
into the code repository's history, as [005](../005-code-repositories-point-at-zingo-adrs-by-submodule.md) records.

Inside a zingolib checkout, `docs/adr/` is therefore a view of this
repository. The records in this repository's `zingolib/` directory appear at
`docs/adr/zingolib/`, and the org-scoped records appear directly under
`docs/adr/`. A citation of `docs/adr/zingolib/0011-nym-mixnet-transmission.md`
in zingolib source names the file `zingolib/0011-nym-mixnet-transmission.md`
here. The directory stays empty until the reader initialises the submodule,
and a GitHub diff shows it as a single link to the pinned commit.

Run the commands below from the code repository's root to read the records or
to advance the pin. Propose or amend a record by a pull request against `dev`
in this repository, never by an edit inside the submodule.

zingolib: `.gitmodules`
```ini
[submodule "docs/adr"]
	path = docs/adr
	url = https://github.com/zingolabs/zingo-adrs.git
	branch = dev
```

```sh
# Materialise the records after cloning the code repository
git submodule update --init docs/adr

# Advance the pin to the current dev of zingo-adrs, then commit the new hash
git submodule update --remote docs/adr
```
