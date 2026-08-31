# Rust BUILD.jobs recipes (plugin-cargo + musl toolchain, no GCC)

Canonical pattern: `plugin-cargo` turns `Cargo.lock` into one `cargocrate`
import per crates.io crate; the recipe assembles a vendor directory + offline
cargo config and builds a **musl-DYNAMIC** binary linked by Rust's bundled
`rust-lld`. The output's ELF interpreter points at the `hostmusl` loader's
content-addressed store path, so `runtime_deps = [musl]`.

## Canonical pins

```starlark
CARGOPLUGIN = fetcher(
    name   = "cargoplugin",
    url    = "https://github.com/jobs-build/plugin-cargo/archive/refs/tags/v0.1.1.tar.gz",
    sha256 = "e9cf33c39662cd8f3facc976a49d9c5fc599df16bd90b424c44612973e7fc113",
)
RUST_VERSION = "1.96.0"
rust_sha = {
    "x86_64":  "545aff63f37dea2fcbd8037b877219fca6fbba97660bdcb8d3a0fc5df5fa9edf",
    "aarch64": "f55a1f02736d9bcd38969a30f2e6bde8184fd36f13e94f773cac068a56407f16",
}
```

Use **plugin-cargo v0.1.1**, not v0.1.0: its companion `fetcher-cargocrate`
v0.1.1 preserves crate file permission bits (v0.1.0 flattened modes to 0644,
which breaks crates whose build scripts exec helper files — e.g. jemalloc's
`config.status`). Keep `RUST_VERSION` a module-scope constant reused in the
URL and the cache key. Toolchain target is always
`<arch>-unknown-linux-musl` — musl host *and* target.

## Plugin contract

```starlark
crates = plugins["cargo"](cargo_lock = source.read("Cargo.lock"))
# each c: {"name", "version", "input"}  — input rooted at <name>-<version>/
```

Workspace mode (Cargo workspaces with path-dep siblings; see
`jobs-build/example-cargo-workspace`): pass `workspace = True` and the plugin
reads the mounted source tree, returning
`{"crates": […], "sources": […], "generated": {path: bytes}}` — a pruned
lockfile + reduced root `Cargo.toml` you forward as `generated =` and
`sources =` in the build struct.

**Git dependencies are a hard error** in plugin-cargo (no stable checksum).
Strip `[[package]]` blocks with `source = "git+…"` from the lock text before
the plugin call, import those repos with the `github` fetcher, and `sed` the
manifests to path deps (a `[patch]` section is NOT sufficient — cargo still
loads the patched-over git source, fatal offline). Then build with plain
`--offline` instead of `--frozen`, since the path rewiring legitimately
updates the lock.

## The worked example (validator-watch `barn`, the current canonical form)

```starlark
# BUILD.jobs — hermetic JOBS build of barn, fully offline & C-toolchain-free.
#
# Pins to bump together:
#   - CARGOPLUGIN url/sha256 → github.com/jobs-build/plugin-cargo release
#   - RUST_VERSION/rust_sha  → https://static.rust-lang.org/dist/ (musl tarballs)

CARGOPLUGIN = fetcher(
    name   = "cargoplugin",
    url    = "https://github.com/jobs-build/plugin-cargo/archive/refs/tags/v0.1.1.tar.gz",
    sha256 = "e9cf33c39662cd8f3facc976a49d9c5fc599df16bd90b424c44612973e7fc113",
)

RUST_VERSION = "1.96.0"

def plugins():
    return {"cargo": imp(fetcher = CARGOPLUGIN, params = {})}

def build():
    # Every "x86_64" in the script below (target triple, ld-musl loader,
    # rust-std/rustlib dirs, [target.*]) is the same token, so a single
    # .replace() at the end retargets the whole script.
    arch = {"linux/amd64": "x86_64", "linux/arm64": "aarch64"}[platform]
    rust_sha = {
        "x86_64": "545aff63f37dea2fcbd8037b877219fca6fbba97660bdcb8d3a0fc5df5fa9edf",
        "aarch64": "f55a1f02736d9bcd38969a30f2e6bde8184fd36f13e94f773cac068a56407f16",
    }
    toolchain = imp(fetcher = "tarball+https", params = {
        "url": "https://static.rust-lang.org/dist/rust-" + RUST_VERSION + "-" + arch + "-unknown-linux-musl.tar.gz",
        "sha256": rust_sha[arch],
    })
    musl = imp(fetcher = "hostmusl", params = {}, platform = platform)

    crates = plugins["cargo"](cargo_lock = source.read("Cargo.lock"))

    inputs = {"toolchain": toolchain, "musl": musl}
    copies = []
    for i, c in enumerate(crates):
        name = "crate_%d" % i
        inputs[name] = c["input"]
        copies.append('cp -dpr "$(jq -r .%s <<<"$JOBS_DEPS")/." /build/vendor/' % name)

    script = ("""
musl="$(jq -r .musl <<<"$JOBS_DEPS")"
toolchain="$(jq -r .toolchain <<<"$JOBS_DEPS")"

# 1. musl loader + libgcc_s for the PREBUILT toolchain (its PT_INTERP is
#    /lib/ld-musl-x86_64.so.1; rustc/cargo also NEED-link libgcc_s.so.1).
mkdir -p /lib
ln -sf "$musl/lib/ld-musl-x86_64.so.1" /lib/ld-musl-x86_64.so.1
ln -sf "$musl/lib/libgcc_s.so.1" /lib/libgcc_s.so.1
export LD_LIBRARY_PATH="$musl/lib"

# 2. Stage the toolchain into the writable /build tree (sequential cp avoids
#    the FUSE store's concurrent-read EIO). install.sh is never run.
mkdir -p /build/rust "$out/bin"
cp -dr "$toolchain"/*/rustc/. /build/rust/
cp -dr "$toolchain"/*/cargo/. /build/rust/
cp -dr "$toolchain"/*/rust-std-x86_64-unknown-linux-musl/. /build/rust/
chmod -R u+w /build/rust
export PATH="/build/rust/bin:$PATH"
export CARGO_HOME=/build/cargo

# 3. Vendor every crate (each input rooted at <name>-<version>/). cp -p
#    preserves the store's stable mtimes — cargo re-runs any build script
#    without rerun-if directives whenever ANY file of its package looks newer
#    than the cached script output, so fresh copy mtimes would rerun every
#    vendored build script and cascade into a full recompile despite the warm
#    target cache.
mkdir -p /build/vendor
""" + "\n".join(copies) + """

# 4. Offline config: source replacement + dynamic musl linked by Rust's
#    BUNDLED rust-lld (no GCC); --dynamic-linker/-rpath point the output's
#    interpreter/RUNPATH into the content-addressed store.
RUSTLLD="/build/rust/lib/rustlib/x86_64-unknown-linux-musl/bin/rust-lld"
mkdir -p "$SRC/.cargo"
cat > "$SRC/.cargo/config.toml" <<EOF
[source.crates-io]
replace-with = "vendored-sources"
[source.vendored-sources]
directory = "/build/vendor"
[target.x86_64-unknown-linux-musl]
linker = "$RUSTLLD"
[build]
rustflags = ["-C", "target-feature=-crt-static", "-C", "link-self-contained=yes", "-L", "native=$musl/lib", "-C", "link-arg=--dynamic-linker=$musl/lib/ld-musl-x86_64.so.1", "-C", "link-arg=-rpath", "-C", "link-arg=$musl/lib"]
EOF

# 5. Build offline; install the binary + entrypoint. --frozen = --locked
#    --offline. CARGO_TARGET_DIR points into the declared build cache, so
#    warm builds recompile only the app crate.
export CARGO_TARGET_DIR=/cargo-target-cache
cd "$SRC"
cargo build --release --frozen
cp "$CARGO_TARGET_DIR/release/barn" "$out/bin/barn"
cat > "$out/JOBS.entrypoint" <<'EOF'
{"command": "bin/barn", "args": [], "env": {}}
EOF
""").replace("x86_64", arch)

    return struct(
        name = "barn",
        inputs = inputs,
        env = {},
        script = script,
        runtime_deps = [musl],
        caches = {"/cargo-target-cache": "validator-watch-barn-rust" + RUST_VERSION},
    )
```

## Why each linking flag is there

- `rust-lld` is invoked **directly** (not via a `cc` driver), so link args are
  **bare** — `--dynamic-linker=…`, `-rpath` — never `-Wl,`-prefixed.
- `target-feature=-crt-static`: with a musl host, host == target, and a
  `+crt-static` final binary collides with proc-macros (which need dynamic
  dylibs). The output is musl-dynamic by design.
- `link-self-contained=yes` + `-L native=$musl/lib` resolve `-lc`/`-lgcc_s`
  from the hostmusl artifact plus the toolchain's self-contained crt.
- `--dynamic-linker` + `-rpath` bake the loader's `/jobs/store/<key>` path
  into PT_INTERP/RUNPATH — `jobs-client run`/`image` mount the materialized
  runtime closure at the same paths.

## Non-negotiables

- **`cp -dpr` for crate vendoring** (mtime preservation; see step 3 comment).
- Caches: `{"/cargo-target-cache": "<project>-<component>-rust" + RUST_VERSION}`
  — keyed to the toolchain so a Rust bump gets a fresh cache.
- Big builds need `resources = struct(cpu = "4", memory = "16Gi")` — the
  runner enforces the default 1 GiB as a cgroup `memory.max`; cargo `-j` is
  the real parallelism/memory control (the cgroup caps memory, not CPU).
- Many-input builds (hundreds of crates): start the script with
  `[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")`.

## Native-code deltas (C/C++ deps — lighthouse-harvester pattern)

When crates compile C/C++ (openssl-sys style), the "no GCC" premise inverts:
stage an Alpine v3.22 apk toolchain (gcc, g++, make, cmake, musl-dev,
linux-headers, libclang, …) via the `alpineapk` fetcher, symlink busybox
applets into `/bin`, and switch the cargo config to
`linker = "/usr/bin/gcc"`, `link-self-contained=no`, `-L native=/usr/lib`,
with `-Wl,`-wrapped link args (`link-arg=-Wl,--dynamic-linker=…`,
`link-arg=-Wl,-rpath,$musl/lib`) because gcc drives the link. Apks stay
build-only (`runtime_deps` unchanged); `rm -f /usr/lib/libstdc++.so` keeps
`-lstdc++` static. Add `export TMPDIR=/build/tmp` (sandbox `/tmp` is tmpfs —
unreclaimable cgroup memory) and `export CONFIG_SHELL=/bin/sh SHELL=/bin/sh`
(autoconf must not record the FUSE-store bash). Append (`>>`) to
`.cargo/config.toml` if the repo checks one in.

For PyO3/maturin extension wheels, see the Rust sections of
[python.md](python.md).
