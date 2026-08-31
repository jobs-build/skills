# Python BUILD.jobs recipes (uv + musl CPython)

Canonical pattern: `plugin-uv` turns `uv.lock` into one content-addressed
`pypi` import per dependency; the recipe stages a wheelhouse and installs the
locked set offline with `uv pip install --target` onto a relocatable musl
CPython. No venv, no `uv sync`, no network.

## Canonical pins (examples repo @ HEAD, commit `4dec841`)

| Thing | Pin |
|---|---|
| `uvplugin` | `https://github.com/jobs-build/plugin-uv/archive/refs/tags/v0.1.0.tar.gz` sha256 `1d29d05a3f5d0004dcfb9d93ee3e03c2d151bdf6761996dcf2a1532f6d1897d2` |
| `pybackendplugin` (sdists) | `https://github.com/jobs-build/plugin-pybackend/archive/refs/tags/v0.1.0.tar.gz` sha256 `87d77901f1950a58af3631afd3f8009406a1572039f41f46e632aa076943a58f` |
| `cargoplugin` (PyO3) | `https://github.com/jobs-build/plugin-cargo/archive/refs/tags/v0.1.0.tar.gz` sha256 `1f93fc9319613a677133e6b6f13948cd3c7a8e11c60f59770d893315eddf0818` |
| CPython (musl, relocatable) | `https://github.com/astral-sh/python-build-standalone/releases/download/20260623/cpython-3.12.13+20260623-<arch>-unknown-linux-musl-install_only.tar.gz` — x86_64 `1bd4bba28b410dc617dd4a386ce08bc988b87c4320d1532682f488cc5e16edc7`, aarch64 `b37a1dde4e7b8b2d2367218a3a955a4a08a89558fc7b43628dc05ec0c49e1ce9` |
| uv (static musl) | `https://github.com/astral-sh/uv/releases/download/0.11.24/uv-<arch>-unknown-linux-musl.tar.gz` — x86_64 `2f9eaa976cf8ad8574623dde6916fb87ec047524fe65b99dc11e44e33eae4bfb`, aarch64 `222fcd9878ed6f5a8565d121d1a10e1f958e3c7b42e4ec5cbc6927b2be20cc1a` |
| Rust musl (PyO3 only) | `https://static.rust-lang.org/dist/rust-1.96.0-<arch>-unknown-linux-musl.tar.gz` — x86_64 `545aff63f37dea2fcbd8037b877219fca6fbba97660bdcb8d3a0fc5df5fa9edf`, aarch64 `f55a1f02736d9bcd38969a30f2e6bde8184fd36f13e94f773cac068a56407f16` |
| maturin binary (own-extension slice only) | `https://github.com/PyO3/maturin/releases/download/v1.8.3/maturin-<arch>-unknown-linux-musl.tar.gz` — x86_64 `8cd87718295a87e47a9ba02db584e10afa67e1d95a141dc7a37846b2c72a3598`, aarch64 `eb7a07e375b6258f6c563a0a970c4a65f10cb3e4ad0af3ee64f58b6ce61c1b09` |
| musl loader | `imp("hostmusl", {}, platform = platform)` — no sha; host-provided, platform-pinned |

When starting a new project, verify current sha256s by downloading the URL and
running `sha256sum` — never invent a hash.

## Plugin contract

```starlark
pkgs = plugins["uv"](uv_lock = source.read("uv.lock"))
```

Returns a list of dicts: `input` (a `pypi` import), `name`, `version`, and
`kind` (`"wheel"` | `"sdist"`). Register each `input` under an index-derived
key (`whl_%d`) and resolve its path at runtime via
`$(jq -r .whl_0 <<<"$JOBS_DEPS")` — dep store paths are never known at
Starlark time.

## The worked example (examples/python-build/BUILD.jobs)

```starlark
# BUILD.jobs — build a pure-Python app with JOBS + UV, fully offline.

UVPLUGIN = fetcher(
    name   = "uvplugin",
    url    = "https://github.com/jobs-build/plugin-uv/archive/refs/tags/v0.1.0.tar.gz",
    sha256 = "1d29d05a3f5d0004dcfb9d93ee3e03c2d151bdf6761996dcf2a1532f6d1897d2",
)

def plugins():
    return {"uv": imp(fetcher = UVPLUGIN, params = {})}

def build():
    # Every "x86_64" in the script is retargeted by one .replace() at the end;
    # only the toolchain URLs + sha256 are selected per-arch.
    arch = {"linux/amd64": "x86_64", "linux/arm64": "aarch64"}[platform]
    uv_sha = {
        "x86_64": "2f9eaa976cf8ad8574623dde6916fb87ec047524fe65b99dc11e44e33eae4bfb",
        "aarch64": "222fcd9878ed6f5a8565d121d1a10e1f958e3c7b42e4ec5cbc6927b2be20cc1a",
    }
    py_sha = {
        "x86_64": "1bd4bba28b410dc617dd4a386ce08bc988b87c4320d1532682f488cc5e16edc7",
        "aarch64": "b37a1dde4e7b8b2d2367218a3a955a4a08a89558fc7b43628dc05ec0c49e1ce9",
    }
    uv = imp("tarball+https", {
        "url": "https://github.com/astral-sh/uv/releases/download/0.11.24/uv-" + arch + "-unknown-linux-musl.tar.gz",
        "sha256": uv_sha[arch],
    })
    python = imp("tarball+https", {
        "url": "https://github.com/astral-sh/python-build-standalone/releases/download/20260623/cpython-3.12.13+20260623-" + arch + "-unknown-linux-musl-install_only.tar.gz",
        "sha256": py_sha[arch],
    })
    musl = imp("hostmusl", {}, platform = platform)

    pkgs = plugins["uv"](uv_lock = source.read("uv.lock"))

    inputs = {"uv": uv, "python": python, "musl": musl}
    copies = []
    specs = []
    for i, p in enumerate(pkgs):
        name = "whl_%d" % i
        inputs[name] = p["input"]
        copies.append('cp -dr "$(jq -r .%s <<<"$JOBS_DEPS")/." /build/wheelhouse/' % name)
        specs.append("%s==%s" % (p["name"], p["version"]))

    script = ("""
musl="$(jq -r .musl <<<"$JOBS_DEPS")"
uvdir="$(jq -r .uv <<<"$JOBS_DEPS")/uv-x86_64-unknown-linux-musl"
pydir="$(jq -r .python <<<"$JOBS_DEPS")/python"

# 1. musl loader for the dynamically-linked interpreter at BUILD time.
mkdir -p /lib
ln -sf "$musl/lib/ld-musl-x86_64.so.1" /lib/ld-musl-x86_64.so.1
ln -sf "$musl/lib/libgcc_s.so.1"       /lib/libgcc_s.so.1
export LD_LIBRARY_PATH="$musl/lib"

# 2. Stage interpreter + uv into the writable /build tree (sequential cp; amberfuse).
mkdir -p /build/python /build/uv /build/wheelhouse /build/uvcache "$out/bin"
cp -dr "$pydir/." /build/python/
cp -dr "$uvdir/." /build/uv/
chmod -R u+w /build/python /build/uv
export PATH="/build/uv:$PATH" HOME=/build UV_CACHE_DIR=/build/uvcache

# 3. Build the wheelhouse from the fetched .whl inputs.
""" + "\n".join(copies) + """

# 4. Offline install of the LOCKED dependency set into $out/site-packages.
uv pip install \\
    --python /build/python/bin/python3 \\
    --no-index --offline --find-links /build/wheelhouse \\
    --target "$out/site-packages" \\
    """ + " ".join(specs) + """

# 5. App package onto site-packages (pure Python; no PEP 517 build here).
cp -dr "$SRC/src/myapp" "$out/site-packages/myapp"

# 6. Launcher: build-time paths baked via an EXPANDING heredoc; run-time logic
#    via a LITERAL heredoc (so $here/$@ stay literal).
cat > "$out/bin/myapp" <<EOF
#!/jobs/shell/bin/bash
LDSO="$musl/lib/ld-musl-x86_64.so.1"
PYBIN="$pydir/bin/python3"
MUSLLIB="$musl/lib"
EOF
cat >> "$out/bin/myapp" <<'EOF'
here="$(cd "${0%/*}/.." && pwd)"
export PYTHONPATH="$here/site-packages"
export LD_LIBRARY_PATH="$MUSLLIB"
exec "$LDSO" "$PYBIN" -m myapp "$@"
EOF
chmod +x "$out/bin/myapp"

cat > "$out/JOBS.entrypoint" <<'EOF'
{"command": "bin/myapp", "args": [], "env": {}}
EOF
""").replace("x86_64", arch)

    return struct(name = "python-build example", inputs = inputs, env = {}, script = script, runtime_deps = [python, musl])
```

Output layout: `$out/bin/myapp` (launcher), `$out/site-packages/` (flat
`--target` install + the app package), `$out/JOBS.entrypoint`. The launcher
invokes the musl loader explicitly on the interpreter (`exec "$LDSO" "$PYBIN"`),
so no `/lib` symlink is needed in the run sandbox — which is exactly why
`runtime_deps = [python, musl]`.

## Key mechanics

- **Arch retargeting**: write the whole script against literal `x86_64` and
  finish with `.replace("x86_64", arch)`. Per-arch URLs/shas are computed
  *before* the string so they escape the replace.
- **Staging over FUSE**: `cp -dr` sequentially from the store into `/build`,
  then `chmod -R u+w`. The on-demand FUSE store misbehaves under concurrent
  reads; always stage toolchains locally.
- **Two-heredoc launcher idiom**: expanding heredoc bakes absolute store paths;
  `<<'EOF'` literal heredoc carries runtime shell logic untouched by Starlark
  or build-time expansion. Shebang is `#!/jobs/shell/bin/bash`.
- **`env` struct field stays `{}`**; exports live in the script.
- **No closure/caches in the Python examples** — a gap, not a rule. For heavy
  builds (e.g. Rust sdists) add `caches = {"/cargo-target-cache": "<proj>-rust<ver>"}`
  like the Rust recipe does.

## Sdist dependencies: the sub-build pattern (python-sdist-build)

The uv plugin tags deps `kind = "wheel" | "sdist"`. Route sdists through a
sub-build that produces a wheel:

```starlark
if p["kind"] == "wheel":
    inputs[name] = p["input"]
else:
    inputs[name] = bld(source = p["input"], buildJobs = GENERIC)
```

`bld(source=, buildJobs=)` runs an inline recipe *string* (`GENERIC`) against
the fetched sdist. Both branches yield a directory containing a `.whl`, so the
wheelhouse copies and `specs` stay unconditional.

`GENERIC` is a triple-quoted Starlark string that is itself a full recipe:

- Declares `PYBACKENDPLUGIN`; calls
  `plugins["pybackend"](pyproject = pp, platform = platform)` where
  `pp = source.read("pyproject.toml") if source.exists("pyproject.toml") else b""`.
  The plugin resolves the PEP-517 backend against a curated build-system
  wheelhouse (setuptools+wheel for legacy sdists, maturin for Rust extensions)
  and returns `{input, name, version}` records.
- Installs the backend wheels offline, then builds with
  `UV_PYTHON_DOWNLOADS=never uv build --wheel --no-build-isolation --offline
  --no-index --find-links /build/backends --python /build/python/bin/python3
  --out-dir "$out" "$SRC"`.
- Returns `struct(inputs=…, env={}, script=…, runtime_deps=[])` — no `name`;
  the output is just a wheel.

**Quoting discipline**: inside `GENERIC`'s `"""…"""` you cannot use heredocs
(collide with the quotes); build the inner script by concatenation with `\\n`
and escaped `\"`. If you must emit a file containing double quotes (e.g.
`.cargo/config.toml`), use `printf` with `Q=$(printf '\x22')`.

## PyO3 / maturin extension (python-rust-build)

Building your *own* extension wheel — deltas from the pure-Python recipe:

- Two plugins: `{"cargo": …, "uv": …}`;
  `crates = plugins["cargo"](cargo_lock = source.read("Cargo.lock"))`.
- Extra imports: musl Rust 1.96.0 and the standalone musl maturin v1.8.3
  binary; extra loader symlinks `libc.so` and `libgcc_s.so` (link-time names).
- Stage rust: merge `"$rustdir"/*/rustc/.`, `*/cargo/.`, and
  `*/rust-std-x86_64-unknown-linux-musl/.` into `/build/rust/`; export
  `CARGO_HOME=/build/cargo`, `PYO3_PYTHON=/build/python/bin/python3`,
  `LIBRARY_PATH="$musl/lib${LIBRARY_PATH:+:$LIBRARY_PATH}"`.
- Vendor crates into `/build/vendor` and write `$SRC/.cargo/config.toml` with
  source replacement + `rust-lld` linking (see the Rust reference for the exact
  config).
- Build: `maturin build --release --offline --interpreter /build/python/bin/python3
  --compatibility linux --out /build/wheels`, then
  `uv pip install … --find-links /build/wheelhouse --target "$out/site-packages" /build/wheels/*.whl`.
- With `abi3-py312` the `.so` doesn't link libpython, so Rust stays build-only:
  `runtime_deps = [python, musl]` unchanged.

Third-party Rust sdists (python-rust-sdist-build, e.g. `nh3`): keep the
uniform GENERIC PEP-517 path instead — pybackendplugin serves the
arch-matching `musllinux_1_1_<arch>` maturin *wheel* as just another curated
backend, gate the Rust toolchain/vendor block on
`if source.exists("Cargo.lock")` (evaluated at Starlark time), and export
`CARGO_NET_OFFLINE=true MATURIN_NO_INSTALL_RUST=1`.
