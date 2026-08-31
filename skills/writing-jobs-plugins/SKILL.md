---
name: writing-jobs-plugins
description: Use when creating or editing a JOBS build plugin (lockfile → content-addressed imports, e.g. plugin-go/plugin-uv/plugin-mix) or a JOBS fetcher (params → fetched tree, e.g. fetcher-gomod/fetcher-pypi), when a plugin call fails with a fetcher-pin or CBOR protocol error, or when adding a new language ecosystem to JOBS.
---

# Writing JOBS plugins and fetchers

## Overview

A **plugin** is a static Go binary that receives a recipe's kwargs (usually
lockfile bytes) as CBOR on stdin and returns import specs on stdout — it runs
sandboxed with **no network**. A **fetcher** is an executable `./fetch` that
downloads one artifact into `$JOBS_OUTPUT_DIR` — it is the only
network-capable step. A new ecosystem usually needs one of each, released as
`jobs-build/plugin-<eco>` + `jobs-build/fetcher-<eco>`, pinned together.
Only the **seed fetchers** (`tarball+https`, `github`, `hostmusl`) are
embedded in the engine and usable without a pin; every other fetcher — yours
included — must be pinned, either by a recipe's `fetcher(...)` or a plugin's
`fetchers.toml`.

Copy `jobs-build/plugin-mix` (5 source files — the minimal reference) and
`jobs-build/fetcher-gomod` (shell) or `fetcher-pypi` (Go, stdlib-only) as
templates. Full anatomy, protocol structs, and worked file listings:
[references/anatomy.md](references/anatomy.md).

## Plugin contract (the parts you must get right)

- Repo layout: `main.go` + one `<lockfile>.go` parser, `internal/importdef/`
  (**verbatim copy** from an existing plugin — byte-identical across all
  five), `fetch` wrapper, `fetchers.toml`, `BUILD.jobs` (boilerplate — copy
  it), vendored deps (`cbor/v2`, optionally `BurntSushi/toml`), AGPL LICENSE.
- Protocol: request `{call, source, deps, dir}`; decode kwargs defensively
  (accept both `[]byte` and `string`); use a `cbor.DecMode` with
  `DefaultMapType` of `map[string]any`. stdout is protocol only — all
  diagnostics to stderr. Exit 0 = ok, **75 = retryable**, else hard failure.
- Each dep becomes
  `inputSpec{Kind: "import", Definition: importdef.Definition{Fetcher: "<name>", Params: CanonicalParams(map[string]any{…})}.Canonical()}`.
  Response: bare array of `{name, version, input}` (plugin-go uses `path`;
  plugin-uv adds `kind: wheel|sdist`), or a map (`{modules, sources, closure}`
  / `{crates, sources, generated}`) for closure/workspace modes.
- **Determinism**: sort + dedup output — same lockfile in, byte-identical
  CBOR out, or pin-stage memoization breaks.
- **Backward compat**: the legacy response shape must stay byte-identical
  when new kwargs are absent. New capability = new optional kwarg + a map
  response, never a change to the old shape.
- Git deps / missing checksums / empty locks are **hard errors** — there is
  no stable content address for them.
- Conventions: kwarg names are snake_case lockfile names (`mix_lock`,
  `uv_lock`, `go_sum`, `cargo_lock` → yours: `<eco>_lock`); the fetcher name
  is the registry/ecosystem short name (`gomod`, `pypi`, `hexpm`,
  `cargocrate`); response elements use `name` except where the ecosystem's
  own identifier is a path (Go module `path`). Params carry whatever checksum
  the lockfile/registry provides (sha256 preferred; the fetcher verifies it —
  a weaker digest like Composer's SHA-1 shasum is the trust anchor only if
  the registry offers nothing stronger).
- `fetchers.toml` at the artifact root pins the companion fetcher
  (`[[fetcher]] name/url/sha256`); the engine injects these into every
  emitted import naming a non-seed fetcher, and hard-errors without a pin.
  **Bump it in lockstep with each fetcher release.**

## Fetcher contract

- Artifact root holds executable `./fetch`. In: `JOBS_FETCH_PARAMS` (params
  as JSON), `JOBS_OUTPUT_DIR` (only guaranteed-writable dir), optional
  `JOBS_SECRETS_FILE`. Runs **with network**; exit 0/75/other as above.
- The output tree is content-addressed: **scrub nondeterminism** (fetcher-gomod
  deletes `sumdb/`, `list`, `*.lock`) and preserve permission bits
  (fetcher-cargocrate v0.1.0's mode-flattening broke exec-ing build scripts —
  that class of bug).
- Needs a toolchain at run time? Declare it `runtime_deps` in the fetcher's
  BUILD.jobs and bake its store path into a generated `env.sh` that `fetch`
  sources (fetcher-gomod pattern).

## Testing (before tagging anything)

1. Parser unit test with a **verbatim real-world lockfile fixture** + negative
   tests (git dep, missing checksum, empty lock → error).
2. Protocol round-trip: `run(io.Reader, io.Writer)` (take streams as args!),
   feed a CBOR request, decode the response, assert `Kind == "import"` and
   `importdef.Decode(...).Fetcher` + params fields.
3. Fetchers: `httptest.NewServer` + injected `getenv` (fetcher-pypi pattern).
4. End-to-end: a consuming recipe via `jobs-client build`; rerun must print
   `(cached)`.
5. `gofmt -l .` prints nothing; `CGO_ENABLED=0` build works.

## Release

Tag `v0.x.y`; consumers pin
`https://github.com/jobs-build/<repo>/archive/refs/tags/v0.x.y.tar.gz` +
sha256 of that tarball (`curl -L <url> | sha256sum`). For a new ecosystem the
order is: tag the **fetcher** first, pin it in the plugin's `fetchers.toml`,
then tag the plugin. On later fetcher releases update the plugin's
`fetchers.toml` and re-tag the plugin; downstream recipes bump their
`fetcher(...)` pin to the new plugin tag.

## Common mistakes

| Mistake | Fix |
|---|---|
| Logging to stdout | stdout is CBOR protocol; stderr only |
| Editing the vendored `internal/importdef` | Keep it byte-identical to the other plugins |
| Params via ad-hoc CBOR encode | Always `importdef.CanonicalParams` — canonical bytes are the identity |
| Unsorted output | Sort by name/path then version |
| New kwarg changes old response shape | Old shape must stay byte-identical |
| Fetcher output varies per run | Scrub timestamps/locks/registry litter before exit |
| Forgot `fetchers.toml` bump after fetcher release | Recipes keep building with the old fetcher — bump together |
