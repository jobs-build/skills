# Ruby / Rails BUILD.jobs recipes (plugin-bundler + Alpine musl Ruby + zig cc)

Canonical pattern (examples/rails-build, 2026-08-19): a relocatable musl Ruby
assembled from Alpine v3.22 apks; `plugin-bundler` turns `Gemfile.lock` into
one `rubygems` import per gem; offline `bundle install --local` where
precompiled `-linux-musl` native gems unpack and self-contained C gems
(bcrypt, puma, json, prism, nio4r, msgpack, bootsnap) compile with **zig cc**;
a production Rails boot migrates SQLite and smoke-tests the compiled
extension; jsbundling assets build via musl Node + a yarn offline mirror.
Read `examples/rails-build/BUILD.jobs` in full — this file gives pins,
structure, and traps.

## Pins (canonical, tagged releases)

```starlark
BUNDLERPLUGIN = fetcher(name = "bundlerplugin",
    url = "https://github.com/jobs-build/plugin-bundler/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "2ad9805614f7eb2245e55dbbbe0bb37f63bfca23875c24d628dcbe9c6331ceab")
NODEPLUGIN = fetcher(name = "nodeplugin",
    url = "https://github.com/jobs-build/plugin-node/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "0fce6d96a9a6c0ed9e6b11840a00d7df18e85c66bac24541e6082345b9b1bbac")
ALPINEAPK = fetcher(name = "alpineapk",
    url = "https://github.com/jobs-build/fetcher-alpineapk/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "1f5a24368d977bce1c371ecc593d7b1fc23b62c51010a77c79dbede964e53cc9")
TARBALLXZ = fetcher(name = "tarballxz",     # zig tarballs (.tar.xz)
    url = "https://github.com/jobs-build/fetcher-tarballxz/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "1dd0a6d76aa3557a5dd089995999a600e15eb85be2fe75d7e3f00bcaa1281d9e")
```

Toolchain: Alpine v3.22 `ruby`/`ruby-libs`/`ruby-dev` 3.4.4-r0 +
`ruby-bundler` 2.6.8-r0 + musl, busybox, libgcc, gmp, libucontext(-dev),
zlib, libcrypto3/libssl3, libffi, yaml, make, tzdata (per-arch sha256s in the
example; `libstdc++` x86_64-only — the arm64 musl Node statically links it).
zig 0.16.0 via `TARBALLXZ` (`strip: 1`); musl Node v22.23.1 from
unofficial-builds + yarn classic 1.22.22 via seed `tarball+https`.

## Preparing the lockfile (host-side, once)

The committed `Gemfile.lock` must be musl-only **with checksums** — the
`CHECKSUMS` section's sha256s are the content addresses for the gem fetches:

```bash
bundle config set --local lockfile_checksums true
bundle lock
bundle lock --remove-platform <non-musl platforms>
bundle lock --add-platform x86_64-linux-musl aarch64-linux-musl
```

Both arch variants of native gems land in the lock; the `.gem` cache holds
both and bundler installs the matching one.

## Structure

`plugins()` returns `{"bundler": …, "node": …}` (plain dict — lock is
in-repo). In `build()`:

```starlark
gems = plugins["bundler"](gemfile_lock = source.read("Gemfile.lock"))
for i, g in enumerate(gems):
    inputs["gem_%d" % i] = g["input"]
    gem_copies.append('cp "$(jq -r .gem_%d <<<"$JOBS_DEPS")"/*.gem "$SRC/vendor/cache/" 2>/dev/null || true' % i)
npms = plugins["node"](yarn_lock = source.read("yarn.lock"))
# → *.tgz into /build/npm-mirror
```

Script phases (all in one string ending `.replace("x86_64", arch)`):

1. Stage apks into `/` (`cp -dr "$(jq -r .apk_… <<<"$JOBS_DEPS")/." /`).
2. Busybox applets → `/bin`; `ln -sf /bin/busybox /usr/bin/env` (rbconfig
   bakes `/bin/mkdir`, `/bin/sh`; npm shims need `/usr/bin/env`).
3. zig + `cc`/`c++` shims (below) + `export ZIG_GLOBAL_CACHE_DIR=/zig-cache
   ZIG_LOCAL_CACHE_DIR=/zig-cache`.
4. Stage Node + yarn into `/build`.
5. Gems: `mkdir -p "$SRC/vendor/cache"`, the gem copies, then
   `export BUNDLE_FROZEN=true BUNDLE_PATH=vendor/bundle BUNDLE_DISABLE_SHARED_GEMS=true`
   and `bundle install --local`.
6. JS: write `.yarnrc` pointing `yarn-offline-mirror` at `/build/npm-mirror`,
   `yarn install --offline --frozen-lockfile`.
7. `export RAILS_ENV=production SECRET_KEY_BASE_DUMMY=1
   BOOTSNAP_CACHE_DIR=/bootsnap-cache`; `rails db:prepare`; a
   `rails runner` smoke test proving the zig-built bcrypt loads
   (`User.create! … .authenticate`); `rails assets:precompile`.
8. Assert `public/assets/.manifest.json` and `application-*.js` exist.
9. `cp -dr "$SRC"/. "$out"/` then
   `rm -rf "$out/vendor/cache" "$out/node_modules" "$out/tmp/cache" "$out/.yarnrc"`;
   write `bin/serve` (the `/jobs/store/*/` closure-staging preamble +
   `exec bundle exec rails server -b 0.0.0.0 -p "${PORT:-3000}"`) and
   `JOBS.entrypoint {"command": "bin/serve"}`.

Struct: `env = {}`; `runtime_deps` = the runtime apk subset (musl, busybox,
libgcc, gmp, libucontext, zlib, libcrypto3, libssl3, libffi, yaml, ruby,
ruby-libs, ruby-bundler, tzdata — headers/make/zig/node are build-only);
`resources = struct(cpu = "2", memory = "4Gi")` (default 1 Gi kills clang);
`caches = {"/zig-cache": "<project>-zig0.16.0",
"/bootsnap-cache": "<project>-bootsnap-ruby3.4.4"}`.

## The zig cc shim (retargets every mkmf compile)

rbconfig's `CC` is the relative string `cc`, so a `/build/bin/cc` first on
PATH retargets all native gem compiles:

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
exec /build/zig/zig cc -target x86_64-linux-musl -fno-direct-access-external-data "$@"
WRAP
```

(+ identical `c++` shim). When any gem needs staged Alpine `-dev`
headers/libs, append `-I/usr/include -L/usr/lib -L/lib` (zig only searches
its own sysroot); add `ar`/`ranlib` shims (`zig ar` / `zig ranlib`) if a gem
archives a static lib — busybox has neither (fizzy-build pattern).

Known zig/lld source pins: **json = 2.9.1** (≥2.10's
`__builtin_cpu_supports` SIMD probing fails in a `.so`); **trilogy = 2.12.6**
(2.11.1's extconf probes fail). Wrap `bundle install --local` in a handler
that tails `gem_make.out`/`mkmf.log` on failure — bundler hides the real
compiler error:

```sh
bundle install --local || {
  for f in $(grep -lEi 'fail|error' vendor/bundle/ruby/*/extensions/*/*/*/gem_make.out vendor/bundle/ruby/*/gems/*/ext/*/mkmf.log 2>/dev/null); do
    { echo "===== $f"; tail -n 80 "$f"; } >&2
  done
  exit 1
}
```

## Building an upstream Rails app (fizzy-build wrapper pattern)

When the recipe repo wraps an upstream ref (`basecamp/fizzy` style):

- One pin tuple is the whole bump surface:
  `FIZZY = ("basecamp", "fizzy", "<commit sha>")` →
  `imp(fetcher = "github", params = {"owner": …, "repo": …, "ref": …})`.
- Declare the tree as a resolution dep (`plugins()` struct form) **and** a
  build input with the identical definition — one graph node.
- Feed the plugin `deps["fizzy"].read("Gemfile.lock")` with
  `platforms = ["ruby", GEM_PLATFORM[platform]]` and a `transform =` dict
  (`gitgems_path`, `drop_gems`, `pin_versions`, `add_dependencies`); it
  returns `{packages, lock}` — inject the derived lock via heredoc +
  `.replace("LOCK_CONTENT_PLACEHOLDER", lock)`.
- Git-sourced gems (rails main, turbo-rails, …) come back as github imports
  staged into `vendor/gitgems/<name>` as bundler `path:` sources — git never
  runs in the sandbox.
- Assemble the app at `/build/app` from the import + a committed overlay
  `Gemfile`; `$SRC` contributes only the overlay.
- Add `BUNDLE_WITHOUT=development:test` and `BUNDLE_VERSION=system` (stops
  bundler self-updating to the lock's newer `BUNDLED WITH` offline).

Port gitlab-build's self-checking pin idiom: assert `.ruby-version` (or
`GITLAB_*_VERSION` files) against your pins at pin time and `fail()` with the
expected value — bumps become mechanical.

## Generic Ruby (non-Rails)

Keep: apk slice, zig shims, busybox/env symlinks, bundler plugin +
`vendor/cache` + `bundle install --local`, `/zig-cache`, runtime apk subset,
closure-staging entrypoint, json 2.9.1 pin. Drop: Node/yarn tier,
`db:prepare`, `assets:precompile`, bootsnap cache, `SECRET_KEY_BASE_DUMMY`.
Entrypoint becomes `bundle exec <bin>`.

Large gem sets: add the `JOBS_DEPS_FILE` fallback prologue (gitlab-build
needed it at ~950 imports).
