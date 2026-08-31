---
name: writing-build-jobs
description: Use when creating or editing a BUILD.jobs file — hermetic JOBS build recipes (Starlark) for Go, Rust, Python, Elixir/Phoenix, Ruby/Rails, or multi-language projects; also when a JOBS build fails offline, a plugin call errors, caches don't warm, or you need pins for toolchains/plugins/fetchers.
---

# Writing BUILD.jobs recipes

## Overview

JOBS is a Nix/Bazel-inspired distributed build system: every build runs
`net=none` in a rootless sandbox; every input (toolchain, module, wheel, gem,
apk) is a pinned, content-addressed import. A `BUILD.jobs` file is a Starlark
recipe defining `plugins()` (lockfile-expanding plugins + resolution deps)
and `build()` (inputs, a bash script, runtime deps, caches).

**Core principle: nothing is fetched at build time.** If the script would
touch the network, the recipe is wrong — turn the fetch into an `imp()` with
a sha256, a plugin call over the lockfile, or a pinned apk.

## Workflow

1. Read [references/recipe-api.md](references/recipe-api.md) — the evaluation
   model, globals, `build()` struct fields, sandbox contract, caches.
2. Read the language reference and copy its worked example as the skeleton:
   [go.md](references/go.md) · [rust.md](references/rust.md) ·
   [python.md](references/python.md) · [elixir.md](references/elixir.md) ·
   [ruby-rails.md](references/ruby-rails.md). Multi-component, upstream-source,
   or monorepo builds: also [advanced.md](references/advanced.md).
3. Reuse the reference pins verbatim (verified against the org repos as of
   2026-08-31); for anything newer, download the artifact and `sha256sum`
   it — **never invent a hash**.
4. Verify: `jobs-client build --source .` (or `jb run`), then rerun — the
   rerun must print `(cached)`. Debug failing scripts inside the sandbox
   with `jobs-client develop`.

## Quick reference

| Ecosystem | Plugin call | Per-dep fetcher |
|---|---|---|
| Go | `plugins["go"](go_sum=…, go_mod=…, go_closure=["."])` | `gomod` |
| Rust | `plugins["cargo"](cargo_lock=…)` | `cargocrate` |
| Python | `plugins["uv"](uv_lock=…)` | `pypi` |
| Elixir | `plugins["mix"](mix_lock=…)` | `hexpm` |
| Ruby | `plugins["bundler"](gemfile_lock=…)` | `rubygems` |
| Node assets | `plugins["node"](package_lock=…)` | `npm` |

Universal script prologue:

```sh
set -e
[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")
# resolve inputs: "$(jq -r .name <<<"$JOBS_DEPS")"
```

Universal idioms: per-arch sha dicts + one `.replace("x86_64", arch)` on the
script; stage toolchains into `/build` with sequential `cp -dr` +
`chmod -R u+w` (never run from the FUSE store); dependency trees (go module
cache, cargo vendor, gems) take `cp -dpr` — the `-p` preserves store mtimes
that gate rebuilds, while toolchains stay `cp -dr`; `env = {}` with exports
in the script; a one-line smoke test (`"$out/bin/x" --version >/dev/null`)
before the `JOBS.entrypoint` heredoc that ends the script; cache ids keyed to
the toolchain version — `<project>-<component>-go1.26.4`, or
`<project>-go1.26.4` when the repo IS the component.

## Common mistakes

| Mistake | Fix |
|---|---|
| Invented/guessed sha256 | Download + `sha256sum`; copy reference pins verbatim |
| Git deps in the lockfile | Hard error in every plugin — strip them, import via `github` fetcher + path-dep rewiring (rust.md, elixir.md) |
| `runtime_deps` wrong | Static Go binary → `[]`; musl-dynamic Rust/Python → `[musl]`(+interpreter); mix release → the RUNTIME apk slice |
| Missing `resources` on a heavy compile | Default cgroup is 1 GiB `memory.max` — OOM-SIGKILL. Add `struct(cpu="2", memory="4Gi")` |
| No `(cached)` on rerun | Something nondeterministic entered the source cover — check `.amberignore` and `closure=` |
| Script uses a tool not in the seed shell | Only 22 binaries on PATH — use `busybox <applet>` or stage apks |
| `plugins["x"](…)` outside `build()` / positional args | Plugins are callable only in `build()`, kwargs only |
| Assuming CWD | Repo-root builds start at `/` — always `cd "$SRC"` |

## Creating a new plugin or fetcher instead

If the ecosystem has no plugin yet, see the sibling skill
**writing-jobs-plugins**.
