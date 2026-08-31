# BUILD.jobs Starlark API (authoritative, from jobs-iroh v0.31.0)

The engine (`jobs-build/jobs-iroh`, packages `recipe/` + `builddef/`) is the
contract; this file condenses it for recipe authors.

## Evaluation model

- `BUILD.jobs` at the build dir (override: `--build-file`, or per-input
  `build_file=` / inline `buildJobs=`).
- The file is evaluated **twice**: the `pluginresolve` stage calls
  `plugins()` (no plugins callable), the `pin` stage calls `build()` (live
  `plugins` mapping; the `def plugins():` is stripped from the AST first).
  Both are called with zero arguments. `build()` is required; `plugins()`
  optional.
- Stock Starlark, hermetic: no `load()`, no `print` output, no `while`, no
  recursion, no top-level `if`/`for` (use comprehensions and module-level
  constants), no `set()`. `fail("msg")` aborts with a message.

## Globals

| Name | What |
|---|---|
| `platform` | target platform string, e.g. `"linux/amd64"`, `"linux/arm64"` |
| `params` | decoded `--param k=v` values (all strings); `None` when absent |
| `source.read(path)` | **bytes** of a source file (errors if missing); wrap in `str()` before `.strip()`/`.splitlines()` |
| `source.exists(path)` | bool |
| `imp(fetcher, params, requiredTags=[], platform=None)` | declare an import; `fetcher` = seed-fetcher name string or a `fetcher(...)` value |
| `fetcher(name, url="", sha256="", build=None)` | declare a fetcher JOBS builds itself; exactly one of url+sha256 (a GitHub tarball, `strip:1`) or `build=` |
| `bld(source, dir="", platform=platform, params=None, buildJobs="", build_file="")` | build-of-a-build; `source` is an Input; `buildJobs` (camelCase!) = inline recipe text |
| `subbuild(dir, platform=platform, params=None, build_jobs="", build_file="")` | build a descendant dir (`"sub/app"`) or, with `"//path"`, any dir in the widened context (siblings; note snake_case `build_jobs`) |
| `struct(**kwargs)` | starlarkstruct |
| `plugins[name](**kwargs)` | call a declared plugin — **build() only, keyword args only** |
| `deps[name]` | resolution dep declared by `plugins()` — **build() only**: `.read(path)`, `.exists(path)`, `.path(rel="")` → `/jobs/deps/<name>/…` |

Seed fetchers usable by name string: `tarball+https`
(`{"url", "sha256", "strip"}`), `github` (`{"owner", "repo", "ref"}` — use a
commit SHA, a branch name caches as one import), `hostmusl` (no params).
**Every other fetcher must be declared** with `fetcher(...)` or arrive
pre-pinned inside a plugin's bundled `fetchers.toml` (`gomod`, `cargocrate`,
`pypi`, `hexpm`, `npm`, `rubygems`, `alpineapk`, `tarballxz`, `gitlab`, …).

## `plugins()` return shapes

```starlark
def plugins():
    return {"go": imp(fetcher = GOPLUGIN, params = {})}
# or, with resolution deps readable at pin time:
    return struct(
        plugins = {"bundler": imp(fetcher = BUNDLERPLUGIN, params = {})},
        deps    = {"gitlab": imp(fetcher = GITLABFETCHER, params = {…})},
    )
```

Only `plugins=`/`deps=` fields are accepted (typos hard-error). Values must
be Inputs. Dep names: ≤128 bytes, no slashes.

## `build()` return struct

| Field | Required | Type | Semantics |
|---|---|---|---|
| `inputs` | yes | `{str: Input}` | keys become `$JOBS_DEPS` lookup names |
| `env` | yes | `{str: str}` | merged first; `SRC`/`out`/`PATH`/etc. overwrite it — convention is `{}` and `export` in the script |
| `script` | yes | str | run as `bash -ex` from a file |
| `runtime_deps` | yes | `[Input]` | must be a subset of `inputs` values; ships with the artifact — every entry's transitive closure is mounted at the same `/jobs/store/<key>` paths at run/image time |
| `name` | no | str | display label only, never affects identity |
| `closure` | no | `[str]` | COMPLETE cover of the source tree (build dir NOT auto-seeded; must cover it or pin fails). Narrows the rebuild key: edits outside it memo-hit |
| `sources` | no | `[str]` | additive covered paths (widened context only; mutually exclusive with `closure`) |
| `generated` | no | `{path: str\|bytes}` | pin-synthesized files overlaid on the covered tree (≤1 MiB total) — pruned lockfiles, version stamps |
| `sources_allow_escaping` | no | `[str]` | symlinks allowed to point outside the context |
| `caches` | no | `{path: id}` | persistent build caches, see below |
| `resources` | no | `struct(cpu="2", memory="4Gi")` | scheduling requirement; default cgroup is 1 CPU / **1 GiB memory.max** — big compiles get OOM-killed without this |

Cover paths: `"//lib/x"` = context-root-relative, `"proto"` = build-dir
relative. The context root widens automatically to the git repo root when you
build a subdirectory (`$SRC` = the dir, `$SRC_ROOT` = the root).

## The sandbox the script runs in

- Rootless namespaces incl. **net — no network**. Imports/fetchers are the
  only network-capable step.
- `$SRC` = writable source (build dir); `$out` = `/build/out` (lowercase!);
  `$HOME` = `/build`; `/tmp` tmpfs; deps read-only under `/jobs/store/<key>`.
- `$JOBS_DEPS` = JSON `{inputName: "/jobs/store/<key>"}` — **dropped from the
  env when >64 KiB**; always also at `$JOBS_DEPS_FILE`. Robust recipes start:
  ```sh
  set -e
  [ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")
  ```
  and resolve paths with `jq -r .name <<<"$JOBS_DEPS"`.
- `PATH` = the static shell artifact's `bin/` ONLY: bash, sh, jq, busybox,
  cat, cp, mv, rm, ln, ls, mkdir, chmod, tar, grep, sed, env, printf,
  basename, dirname, test, true, false. Anything else:
  `busybox <applet>` — or stage an Alpine userland.
- CWD is `$SRC` for subdirectory builds but `/` for repo-root builds —
  always `cd "$SRC"` explicitly.
- **Stage toolchains out of the store before use** (`cp -dr` sequentially
  into `/build`, then `chmod -R u+w`): the FUSE store misbehaves under
  concurrent reads. Use `cp -dpr` for dependency trees whose mtimes feed a
  build system's staleness checks (go modules, cargo vendor).
- Everything left in `$out` is the artifact. Optional
  `$out/JOBS.entrypoint`: `{"command": "bin/app", "args": [], "env": {}}` —
  relative command resolves against the output root. `#!/jobs/shell/bin/bash`
  shebangs work at run time (`/jobs/shell` symlink).

## Caches

`caches = {"/go-build-cache": "<project>-<component>-<toolchain-version>"}`.

- Path: absolute; not `/`, `/build`, or under `/jobs`, `/build/src`,
  `/build/out`, `/dev`, `/proc`, `/tmp`.
- Id: `[a-zA-Z0-9][a-zA-Z0-9._-]*`, ≤128 bytes, no `:`; the runner appends
  the platform.
- Advisory: seeded rw, atime-pre-aged; after a **successful** build only,
  pruned to what the build touched and republished. Key the id to the
  toolchain version so a toolchain bump starts a fresh cache.
- Cache contents may vanish or be partially pruned — scripts must degrade to
  cold, never break (see the deps-cache self-heal in elixir.md).

## Plugin calls

`plugins["name"](kwarg = value, …)` — kwargs only; bytes/strings/dicts/lists
travel as CBOR to the plugin binary. Responses come back as Starlark; any
`{kind, definition}` pair rehydrates into an Input. Plugin-emitted imports
naming non-seed fetchers get their pins injected from the plugin's bundled
`fetchers.toml` — or hard-error.

## CLI (jobs-iroh's `jobs-client`; older docs say `jb`/`jobs`)

```
jobs-client build   [--source d] [--dir d] [--platform os/arch] [--param k=v]
jobs-client run     [-- args…]      # build + exec JOBS.entrypoint
jobs-client develop                 # interactive bash INSIDE the build sandbox;
                                    # script saved at /build/build.sh, not run
jobs-client image -o app.tar        # OCI tarball from artifact + runtime closure
jobs-client remote-build --server <endpoint-id>
jobs-client diagnose --server <id> --request <id> [--json]
```

A rerun of an unchanged build must print `(cached)`. `develop` is the
debugging tool for a failing script.

## Identity intuitions (why recipes look the way they do)

- Pinning is memoized on source content: same bytes → same work, across
  definitions and machines.
- `name` and cache *contents* never affect identity; `caches` declarations,
  `resources`, `env`, `script`, `inputs`, and the covered source tree all do.
- `closure=` is the lever that stops unrelated edits (docs, sibling
  services) from rebuilding a component. `.amberignore` (gitignore syntax)
  trims what enters the source context at all.
