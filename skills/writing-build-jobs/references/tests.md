# TESTS.jobs: hermetic test recipes (jobs-iroh tests arc, 2026-11-07)

Needs a jobs-iroh newer than v0.35.1 on server, runners and client (the arc
bumped the runner ALPN to `jobs-runner-nats/4.0`). Design record:
`jobs-iroh/docs/design/2026-11-07-tests.md`.

## What a test is

A `TESTS.jobs` next to `BUILD.jobs` defines `test()` — the same struct as
`build()` **minus `runtime_deps`** (an error if present) — and optionally
`plugins()` exactly as for builds. The recipe is evaluated in the same two
stages with the same globals, and the script runs in the same sandbox
(`net=none`, `$SRC`, `/jobs/store/<key>` inputs, `$JOBS_DEPS`, `$out`,
caches). Three things differ:

- **No output.** `$out` exists and is discarded. Exit 0 = passed, anything
  else = failed.
- **`build` is predeclared**: an Input that is *this directory's own build*
  (same context, dir, platform, params; default `BUILD.jobs`) — the exact
  definition `jobs-client build` of that directory constructs, so the test
  joins that build instead of making a second one. It is explicit: a test
  that does not list it in `inputs` does not build it.
- **`timeout`** (test-only) bounds the run — **default 5 minutes**.

## Minimal skeleton

```starlark
def test():
    return struct(
        inputs  = {"app": build},
        env     = {},
        script  = '''
set -e
[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")
app="$(jq -r .app <<<"$JOBS_DEPS")"        # the build's artifact root
"$app/bin/tally" --version >/dev/null
''',
        closure = ["//tests"],                 # what the test reads from the tree
        timeout = "2m",
    )
```

`$(jq -r .app <<<"$JOBS_DEPS")` is the build's artifact root (what the build
left in `$out`); the build's runtime closure is mounted at the same
`/jobs/store/<key>` paths it had at build time, so a `JOBS.entrypoint`-style
relative command works unchanged.

## `test()` return struct

Everything `build()` accepts except `runtime_deps` (rejected), plus:

| Field | Required | Type | Semantics |
|---|---|---|---|
| `timeout` | no | str, Go duration (`"90s"`, `"10m"`, `"1h30m"`) | wall-clock limit of the run; whole seconds, at least `"1s"`; **default `"5m"`**. Past it the sandbox is killed — the whole process tree — and the test fails. `build()` rejects the field; builds are unbounded. |

`resources = struct(cpu = "2", memory = "4Gi")` works as for builds: memory
is a hard cgroup limit (OOM-kill = failure), CPU a scheduling reservation;
the default is 1 CPU / 1 GiB. Local `jobs-client test` applies no limits.

## Worked example: Go unit tests plus an artifact smoke test

Same pins and plugin contract as go.md. The test stages the toolchain and
module cache exactly like the build does (go.sum already covers test-only
modules), runs `go test`, then exercises the real artifact through `build`:

```starlark
# TESTS.jobs — hermetic tests of tally. Pins as in BUILD.jobs.

GOPLUGIN = fetcher(
    name   = "goplugin",
    url    = "https://github.com/jobs-build/plugin-go/archive/refs/tags/v0.1.2.tar.gz",
    sha256 = "29a3c0f5b5dd21128c2553e05423e906adbd345cb51de527dfdddd335ae71426",
)

GO_VERSION = "1.26.4"

def plugins():
    return {"go": imp(fetcher = GOPLUGIN, params = {})}

def test():
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

    inputs = {"toolchain": toolchain, "app": build}
    copies = []
    for m in res["modules"]:
        name = "mod_" + m["path"].replace("/", "_").replace(".", "_").replace("-", "_")
        inputs[name] = m["input"]
        copies.append('cp -dpr "$(jq -r .%s <<<"$JOBS_DEPS")/." "$GOMODCACHE/"' % name)

    script = """
set -e
[ -n "${JOBS_DEPS:-}" ] || JOBS_DEPS=$(cat "$JOBS_DEPS_FILE")
export GOROOT=/build/goroot
mkdir -p "$GOROOT"
cp -dr "$(jq -r .toolchain <<<"$JOBS_DEPS")/go/." "$GOROOT/"
export PATH="$GOROOT/bin:$PATH"
export GOMODCACHE=/build/gomodcache GOCACHE=/go-build-cache GOPATH=/build/gopath
export GOPROXY=off GOSUMDB=off GOFLAGS=-mod=mod GOTOOLCHAIN=local CGO_ENABLED=0
mkdir -p "$GOMODCACHE/cache/download"
""" + "\n".join(copies) + """
chmod -R u+w "$GOROOT" "$GOMODCACHE" 2>/dev/null || true
cd "$SRC"
go test ./...
"$(jq -r .app <<<"$JOBS_DEPS")/bin/tally" --version >/dev/null
"""

    return struct(
        name    = "tally tests",
        inputs  = inputs,
        env     = {},
        script  = script,
        timeout = "15m",
        caches  = {"/go-build-cache": "validator-watch-tally-go" + GO_VERSION},
    )
```

No `closure=` here on purpose: the whole service dir is the cover, so any
source edit re-runs the unit suite (what you want). A test that only
exercises the artifact should narrow to its own files
(`closure = ["//tests"]`) — the `build` input is keyed by the artifact's
*content*, so such a test then re-runs only when the binary or the test
files actually change.

## Identity and the pass record (why a docs commit re-tests nothing)

- A pass is recorded as `test-passed:<KT>`. **KT** = script + env + covered
  sources + platform + every input's *content* (the build's artifact bytes
  and runtime closure; an import's tree) — never an input's definition. A
  commit that touches nothing in the test's cover and leaves the build
  output byte-identical makes both the build and the test `(cached)`.
- Not part of KT: `resources`, `timeout`, `name`, cache contents. Raising a
  limit never re-runs a passed test.
- A failed test records nothing and re-runs on the next `test`/`remote-test`.
  Non-zero exit, timeout and OOM are hard failures — never retried by the
  scheduler on their own.
- Remote: a failed request stays inspectable on the server until deleted.
  Resubmitting the *unchanged* test while it is held reports the same
  failure at once — delete it first (`jobs-client tui --server …`, `d` on
  the request) or change the test/code, then `remote-test` again.

## CLI

```
jobs-client test        [--source d] [--dir d] [--test-file f] [--platform os/arch] [--param k=v]
jobs-client remote-test --server <endpoint-id> [--source d] [--test-file f] [--cpu …] [--memory …]
jobs-client status      --server <id>                 # KIND column: build | test
jobs-client diagnose    --server <id> --request <id> [--json]
```

`test` prints `test: <F>` / `passed: <KT>` and exits 1 on failure; an
unchanged rerun prints `(cached)`. `remote-test` pulls the pass record home,
so a later local `test` of the same tree is cached too. The context search
looks for `TESTS.jobs` the way `build` looks for `BUILD.jobs`.

## Common mistakes (tests)

| Mistake | Fix |
|---|---|
| `runtime_deps` in `test()` | Remove — a test ships nothing |
| `timeout` on `build()` | Hard error; only `test()` runs are time-limited |
| Suite needs more than 5 minutes | `timeout = "20m"` (Go duration, whole seconds) |
| Forgot `"app": build` in `inputs` | Nothing is built — the predeclared value is explicit |
| Expecting the app *sources* via `build` | `build` mounts the artifact; sources come from the cover (`closure=`, or the auto-seeded dir) |
| Test re-runs on every commit | Narrow `closure=` to the test files; `.amberignore` docs — the artifact input is keyed by content already |
| Writing to `$out` | Discarded; tests have no output — assert and exit |
