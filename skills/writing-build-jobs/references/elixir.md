# Elixir / Phoenix BUILD.jobs recipes (plugin-mix + Alpine Erlang/Elixir)

Canonical pattern: `plugin-mix` turns `mix.lock` into one content-addressed
`.tar` import per hex package (fetcher `hexpm`, pinned by the lock's **outer
checksum** — the second hash in each entry, which IS the sha256 of the
tarball on repo.hex.pm). The toolchain is a musl Erlang/Elixir from pinned
Alpine v3.22 apks; the deployable is a self-contained `mix release` with
bundled ERTS. Reference: `jobs-build/examples/phoenix-build/BUILD.jobs`
(app `fireside`) — read it in full before writing a new Elixir recipe; this
file gives the pins, the skeleton, and the traps.

## Pins (phoenix-build @ HEAD)

```starlark
MIXPLUGIN = fetcher(name = "mixplugin",
    url = "https://github.com/jobs-build/plugin-mix/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "84e06dae099fef3021d9548e58f4bcaf9ba12551e7df94da2f5118963a4449a8")
NODEPLUGIN = fetcher(name = "nodeplugin",          # Phoenix assets only
    url = "https://github.com/jobs-build/plugin-node/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "0fce6d96a9a6c0ed9e6b11840a00d7df18e85c66bac24541e6082345b9b1bbac")
HEXPM = fetcher(name = "hexpm",                    # also reused for pinned single files
    url = "https://github.com/jobs-build/fetcher-hexpm/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "938fac757eab6ff1c2ea5f0eaa0e4c3d68e2c75bd74c1cdd0b668baf6286e9ba")
ALPINEAPK = fetcher(name = "alpineapk",
    url = "https://github.com/jobs-build/fetcher-alpineapk/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "1f5a24368d977bce1c371ecc593d7b1fc23b62c51010a77c79dbede964e53cc9")
TARBALLXZ = fetcher(name = "tarballxz",            # zig only
    url = "https://github.com/jobs-build/fetcher-tarballxz/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "1dd0a6d76aa3557a5dd089995999a600e15eb85be2fe75d7e3f00bcaa1281d9e")
```

Toolchain apks (Alpine v3.22, per-arch sha256s in the example): `erlang27` +
`erlang27-dev` 27.3.4.6-r0 and `elixir` 1.18.3-r0 from **community** — import
the LITERAL names `erlang27`/`erlang27-dev`; plain `erlang` is a provides-only
virtual name. Supporting apks from main: musl, busybox, libgcc, libstdc++,
zlib, libncursesw, ncurses-terminfo-base, libcrypto3, libssl3, tzdata, make
(+ ccache, hiredis, libxxhash, zstd-libs when compiling NIFs).

Offline hex client + rebar3, imported as pinned single files via `HEXPM` with
`{"url", "sha256", "filename"}` params:

- hex 2.2.1: `https://repo.hex.pm/installs/1.18.0/hex-2.2.1.ez`,
  sha256 `a82791e28977f010a6eff5b2a35457448fe823f67e7390025b455f2fff0118a8`,
  `filename: "hex.ez"`.
- rebar3 3.25.1 from its **GitHub release asset** (repo.hex.pm/installs is
  NOT byte-stable): sha256
  `924576737f0d3098e1fa4691e6f542bf61036d234c61ebdc18aafef8e2bccf67`. The
  filename MUST be exactly `rebar3` — an escript derives its entry module
  from the file name.

## Skeleton

```starlark
APK_ARCH = {"linux/amd64": "x86_64", "linux/arm64": "aarch64"}
APKS = [ (repo, name, version, {arch: sha256}), … ]
RUNTIME = ["musl", "busybox", "libgcc", "libstdc++", "zlib",
           "libncursesw", "ncurses-terminfo-base", "libcrypto3", "tzdata"]

def key_of(name):
    return "apk_" + name.replace("-", "_").replace("+", "_").replace(".", "_")

def plugins():
    return {"mix": imp(fetcher = MIXPLUGIN, params = {}),
            "node": imp(fetcher = NODEPLUGIN, params = {})}   # node: Phoenix only

def build():
    arch = APK_ARCH[platform]
    inputs, stage, runtime_deps = {}, [], []
    for (repo, name, ver, shas) in APKS:
        i = imp(fetcher = ALPINEAPK, params = {
            "branch": "v3.22", "repo": repo, "arch": arch,
            "name": name, "version": ver, "sha256": shas[arch]})
        inputs[key_of(name)] = i
        if name in RUNTIME:
            runtime_deps.append(i)
        stage.append('cp -dr "$(jq -r .%s <<<"$JOBS_DEPS")/." /' % key_of(name))

    pkgs = plugins["mix"](mix_lock = source.read("mix.lock"))
    hex_copies = []
    for i, p in enumerate(pkgs):
        k = "hex_%d" % i
        inputs[k] = p["input"]
        hex_copies.append('cp "$(jq -r .%s <<<"$JOBS_DEPS")"/*.tar "$HEX_HOME/packages/hexpm/"' % k)
    …
```

The script (all inside one `("""…""").replace("x86_64", arch)` string):

1. Stage every apk into `/` and expand busybox:
   `for a in $(/bin/busybox --list); do ln -sf /bin/busybox "/bin/$a"; done`
   plus `ln -sf /bin/busybox /usr/bin/env`.
2. (NIF projects) zig + cc wrappers — see below.
3. (Phoenix) stage musl Node.
4. Offline hex bootstrap:
   ```sh
   export HEX_HOME=/build/hex MIX_HOME=/build/mix HEX_OFFLINE=1
   mkdir -p "$HEX_HOME/packages/hexpm"
   cp "$SRC/hex-cache.ets" "$HEX_HOME/cache.ets"
   <hex_copies>
   mix archive.install "$(jq -r .hex_ez <<<"$JOBS_DEPS")/hex.ez" --force
   mix local.rebar rebar3 "$(jq -r .rebar3 <<<"$JOBS_DEPS")/rebar3" --force
   ```
5. Compile: `export MIX_ENV=prod MIX_BUILD_ROOT=/mix-build-cache LANG=C.UTF-8`,
   `ln -sfn /mix-deps-cache "$SRC/deps"`, deps-cache self-heal loop (below),
   `mix deps.get && mix compile`.
6. (Phoenix) assets + any in-build verification.
7. `mix release --overwrite`, atime touch-back (below), then
   `cp -dr "$MIX_BUILD_ROOT/prod/rel/<app>/." "$out/"` and write
   `$out/JOBS.entrypoint`.

Struct: `env = {}`, `runtime_deps = runtime_deps` (the RUNTIME apk slice —
never `[]`: a mix release with bundled ERTS always dynamically links the
nine-package slice; readelf-verified: libssl3 is build-only, libstdc++ IS
runtime on both arches because `beam.smp` NEEDs it), plus:

```starlark
caches = {
    "/mix-build-cache": "<project>-mix-elixir1.18-otp27",
    "/mix-deps-cache":  "<project>-mix-deps-elixir1.18-otp27",
    # NIF projects add:
    "/zig-cache": "<project>-zig0.16.0",
    "/ccache":    "<project>-ccache-zig0.16.0",
}
```

## The committed `hex-cache.ets` (required, easy to miss)

hex 2.x needs a registry snapshot at `$HEX_HOME/cache.ets` before it can
resolve/verify offline, generated by the SAME hex version the build pins:

```bash
export HEX_HOME=$(mktemp -d)     # fresh — don't reuse an existing one
mix archive.install <path-to-hex-2.2.1.ez> --force
mix deps.get
cp "$HEX_HOME/cache.ets" hex-cache.ets   # commit it; keep OUT of .amberignore
```

`HEX_OFFLINE=1` turns any cache miss into a loud failure; `deps.get`
re-verifies every tarball against mix.lock.

**Git deps in mix.lock are a hard error in plugin-mix** (no stable checksum).
Stock Phoenix pulls two (heroicons, daisyUI upstream) — drop them: render a
placeholder glyph in `core_components.ex`, take daisyUI as a normal npm
package.

## Three cache subtleties (all load-bearing)

1. **Cache `deps/` too, not just `_build`**: a fresh `mix deps.get` into an
   empty `deps/` re-extracts every tarball with new mtimes and mix recompiles
   the whole closure even with a warm `_build`. Hence the
   `ln -sfn /mix-deps-cache "$SRC/deps"` symlink.
2. **Self-heal a partially-pruned deps cache** — degrade to cold, never break:
   ```sh
   for d in /mix-deps-cache/*/; do
     [ -e "${d}.hex" ] && [ -z "$(ls -A "$d" | grep -Fvx .hex)" ] && rm -rf "$d"
   done 2>/dev/null || true
   ```
3. **Atime touch-back after the release**: `tar -cf /dev/null -C /mix-deps-cache .`
   — warm rebuilds never re-read unchanged deps, so without this the
   atime-pruner evicts their sources after the first warm build.

## C NIFs with zig cc (exqlite/SQLite pattern — orthogonal to Phoenix)

zig 0.16.0 via `TARBALLXZ` (`strip: 1`), fronted by ccache — zig's own cache
does NOT cache user C objects (measured: two back-to-back `sqlite3.c`
compiles, ~30 s each; ccache warm: 6 s):

```sh
cat > /build/bin/cc <<'WRAP'
#!/bin/sh
for a in "$@"; do
  shift
  case "$a" in
    -Wl,-z,pack-relative-relocs|-Wl,--compress-debug-sections=zlib) ;;
    *) set -- "$@" "$a" ;;
  esac
done
exec /usr/bin/ccache /build/zig/zig cc -target x86_64-linux-musl -fno-direct-access-external-data "$@"
WRAP
sed 's/zig cc/zig c++/' /build/bin/cc > /build/bin/c++
chmod +x /build/bin/cc /build/bin/c++
export ZIG_GLOBAL_CACHE_DIR=/zig-cache ZIG_LOCAL_CACHE_DIR=/zig-cache
export CCACHE_DIR=/ccache CCACHE_COMPILERCHECK=string:zig-0.16.0
export CCACHE_SLOPPINESS=time_macros,include_file_mtime,include_file_ctime
export CC=/build/bin/cc CXX=/build/bin/c++
```

`CCACHE_COMPILERCHECK=string:…` because the zig import's sha256 already pins
the compiler bytes and zig is re-staged fresh each build (mtime detection
would always miss). The two filtered `-Wl` flags are ones zig's linker
rejects. NIF compiles need `resources = struct(cpu = "2", memory = "4Gi")` —
the sqlite amalgamation + a running beam exceed the default 1 Gi cgroup.

## Phoenix-only parts (drop for a plain mix project)

- `NODEPLUGIN` + musl Node (v22.23.1 from unofficial-builds, `strip: 1`) +
  `plugins["node"](package_lock = source.read("assets/package-lock.json"))`
  seeded into npm's cache (`npm cache add …/*.tgz` per package), then
  `npm ci --offline` with
  `npm_config_cache=/build/npmcache npm_config_offline=true`.
- `config/config.exs` must defuse binary downloads:
  `config :esbuild, path: System.get_env("ESBUILD_PATH")` and
  `config :tailwind, path: System.get_env("TAILWIND_PATH")`; the build exports
  both pointing at `assets/node_modules/.bin/…`, then `mix assets.deploy` and
  `ls priv/static/assets >/dev/null`.
- `mix ecto.create && mix ecto.migrate` plus an in-build roundtrip
  (`mix run -e '…'` under `set -e`) prove the NIF end-to-end; export
  `DATABASE_PATH` and `SECRET_KEY_BASE` (required by `config/runtime.exs`).
- `$out/bin/serve` launcher (`#!/jobs/shell/bin/bash`): stages the runtime
  closure from `/jobs/store/*/` into `/`, relinks busybox applets, exports
  `PHX_SERVER=true PHX_HOST=… PORT=${PORT:-4000}`, and
  `exec bin/<app> start`; entrypoint `{"command": "bin/serve"}`. A plain mix
  release points the entrypoint at `bin/<app> start` behind a similar
  closure-staging preamble.

`.amberignore`: exclude `/_build/`, `/deps/`, `*.ez`, `*.db` — but KEEP
`hex-cache.ets`.
