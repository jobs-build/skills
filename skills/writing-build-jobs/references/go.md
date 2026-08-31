# Go BUILD.jobs recipes (plugin-go + offline module cache)

Canonical pattern: `plugin-go` turns `go.sum` into one `gomod` import per
module; the recipe stages an offline `GOMODCACHE` and runs `go build` with
`GOPROXY=off CGO_ENABLED=0` — a fully static binary, `runtime_deps = []`.

## Canonical pins

```starlark
GOPLUGIN = fetcher(
    name   = "goplugin",
    url    = "https://github.com/jobs-build/plugin-go/archive/refs/tags/v0.1.2.tar.gz",
    sha256 = "29a3c0f5b5dd21128c2553e05423e906adbd345cb51de527dfdddd335ae71426",
)
GO_VERSION = "1.26.4"
# per-arch shas from https://go.dev/dl/?mode=json
go_sha = {
    "amd64": "1153d3d50e0ac764b447adfe05c2bcf08e889d42a02e0fe0259bd47f6733ad7f",
    "arm64": "ef758ae7c6cf9267c9c0ef080b8965f453d89ab2d25d9eb22de4405925238768",
}
```

Rules: keep `GO_VERSION` a module-scope constant reused in both the toolchain
URL and the `caches=` key; never pin a toolchain older than `go.mod`'s `go`
directive (`GOTOOLCHAIN=local` never auto-downloads); when bumping, fetch new
shas from go.dev — never invent a hash.

## Plugin contract (closure mode — pass all three kwargs)

```starlark
res = plugins["go"](go_sum = source.read("go.sum"),
                    go_mod = source.read("go.mod"),
                    go_closure = ["."])          # entry package dirs
mods = res["modules"]     # response: {modules, sources, closure}
```

Each `m` in `mods` has `path` and `input` (a `gomod` import rooted at
`cache/download/…`). `go_closure` lists entry packages (multi-binary:
`["./cmd/" + c for c in CMDS]`); `res["closure"]` is a complete cover of the
transitively imported local package dirs — the plugin always includes the
entry package dirs themselves, so forwarding it as `closure = res["closure"]`
satisfies the pin-time "closure must cover the build dir" check while letting
edits outside those dirs memo-hit. A module with no external deps still
calls the plugin with `go_sum = ""` (keeps the staging loop in place).

## The worked example (validator-watch `tally`, the current canonical form)

```starlark
# BUILD.jobs — hermetic JOBS build of tally.
#
# Pins to bump together:
#   - GOPLUGIN        → github.com/jobs-build/plugin-go releases
#   - GO_VERSION/shas → https://go.dev/dl/

GOPLUGIN = fetcher(
    name   = "goplugin",
    url    = "https://github.com/jobs-build/plugin-go/archive/refs/tags/v0.1.2.tar.gz",
    sha256 = "29a3c0f5b5dd21128c2553e05423e906adbd345cb51de527dfdddd335ae71426",
)

GO_VERSION = "1.26.4"

def plugins():
    return {"go": imp(fetcher = GOPLUGIN, params = {})}

def build():
    goarch = {"linux/amd64": "amd64", "linux/arm64": "arm64"}[platform]
    go_sha = {
        "amd64": "1153d3d50e0ac764b447adfe05c2bcf08e889d42a02e0fe0259bd47f6733ad7f",
        "arm64": "ef758ae7c6cf9267c9c0ef080b8965f453d89ab2d25d9eb22de4405925238768",
    }
    toolchain = imp(fetcher = "tarball+https", params = {
        "url": "https://go.dev/dl/go" + GO_VERSION + ".linux-" + goarch + ".tar.gz",
        "sha256": go_sha[goarch],
    })

    res = plugins["go"](go_sum = source.read("go.sum"),
                        go_mod = source.read("go.mod"),
                        go_closure = ["."])
    mods = res["modules"]

    inputs = {"toolchain": toolchain}
    copies = []
    for m in mods:
        name = "mod_" + m["path"].replace("/", "_").replace(".", "_").replace("-", "_")
        inputs[name] = m["input"]
        copies.append('cp -dpr "$(jq -r .%s <<<"$JOBS_DEPS")/." "$GOMODCACHE/"' % name)

    # Stage the toolchain + module cache from the read-only /jobs/store into
    # the writable build tree with sequential copies, then build from local
    # disk (the FUSE store does not serve go's high-concurrency stdlib reads
    # well).
    script = """
set -e
[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")
export GOROOT=/build/goroot
mkdir -p "$GOROOT" "$out/bin"
cp -dr "$(jq -r .toolchain <<<"$JOBS_DEPS")/go/." "$GOROOT/"
export PATH="$GOROOT/bin:$PATH"
export GOMODCACHE=/build/gomodcache GOCACHE=/go-build-cache GOPATH=/build/gopath
export GOPROXY=off GOSUMDB=off GOFLAGS=-mod=mod GOTOOLCHAIN=local CGO_ENABLED=0
mkdir -p "$GOMODCACHE/cache/download"
""" + "\n".join(copies) + """
chmod -R u+w "$GOROOT" "$GOMODCACHE" 2>/dev/null || true
cd "$SRC"
go build -o "$out/bin/tally" .
cat > "$out/JOBS.entrypoint" <<'EOF'
{"command": "bin/tally", "args": [], "env": {}}
EOF
"""

    return struct(
        name = "tally",
        inputs = inputs,
        env = {},
        script = script,
        runtime_deps = [],
        closure = res["closure"],
        # Large pure-Go translation units (e.g. modernc.org/sqlite) need more
        # than DefaultBuild's 1 GiB.
        resources = struct(cpu = "2", memory = "4Gi"),
        caches = {"/go-build-cache": "validator-watch-tally-go" + GO_VERSION},
    )
```

Omit `resources` for a small service (frontend-api does — the default
1 CPU / 1 GiB fits ordinary Go compiles); add it back the moment a build gets
OOM-SIGKILLed (`… Killed`) or pulls in a heavyweight pure-Go translation unit.

## Non-negotiable details

- **`cp -dpr` for module staging** — `-p` preserves the store's stable
  content-addressed mtimes for cache friendliness. (Older examples use
  `cp -dr`; the `-p` form is current.)
- **`set -e` and the `JOBS_DEPS_FILE` fallback** — inline `$JOBS_DEPS` is
  capped at 64 KiB; builds with many inputs get the deps JSON in a file:
  `[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")`.
- **Env line, verbatim**: `GOPROXY=off GOSUMDB=off GOFLAGS=-mod=mod
  GOTOOLCHAIN=local CGO_ENABLED=0`.
- **Caches key convention**: `<project>-<service>-go` + `GO_VERSION` mounted
  at `/go-build-cache` (`GOCACHE` points there). Platform is auto-appended by
  the runner; caches are advisory, atime-pruned, uploaded on success only.
- **`.amberignore`** (gitignore semantics) per service: exclude `README.md`,
  local dev binaries, `node_modules/`, `dist/`, `*.db` so doc edits don't
  re-key the build.
- Prefer pure-Go deps; keep CGO off; never `gogo/protobuf` (unpruned
  pre-1.17 module graph breaks offline builds).

## Variations

**Multiple binaries** (jobs-iroh, gitlab-build):

```starlark
CMDS = ["jobs", "runnerd", "registryd", "gcsweep"]
res = plugins["go"](…, go_closure = ["./cmd/" + c for c in CMDS])
# script: go build -trimpath -o "$out/bin/" ./cmd/jobs ./cmd/runnerd …
```

Add an in-sandbox smoke test before the entrypoint:
`"$out/bin/jobs" --version >/dev/null`.

**Monorepo / go.work** (example-go-monorepo): the source context widens
automatically to the git repo root (`$SRC` is the service dir,
`$SRC_ROOT` the repo root), so `replace example.com/lib/common => ../../lib/common`
resolves as-is and the plugin's closure covers reached sibling package dirs.
Add `GOWORK=off` to the env line — `go.work` is editor sugar; the build pins
to the consumer module. Alternative (validator-watch): keep each service a
self-contained module with `go_closure = ["."]` and duplicate shared files
per service, because the build context then covers only the service dir.

**CGO + static musl link** (mev-research): `CGO_ENABLED=1 CC=gcc` with an
Alpine v3.22 apk toolchain (musl, busybox, and a ~14-package gcc closure via
the `alpineapk` fetcher; symlink busybox applets into `/bin` and
`ln -sf /bin/busybox /usr/bin/env`), then
`go build -ldflags '-linkmode external -extldflags "-static"'`. Apks are
build-only — `runtime_deps` stays `[]`.

**Embedded frontend** (mev-research): `plugins()` returns
`{"node": …, "go": …}`; `plugins["node"](package_lock = source.read("ui/package-lock.json"))`
feeds an offline `npm ci` + `npm run build` into `ui/dist` (verify with
`test -f dist/index.html`) before the Go build `go:embed`s it. Two caches:
`{"/go-build-cache": …, "/npm-cache": …}`.
