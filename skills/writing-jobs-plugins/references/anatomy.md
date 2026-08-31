# Plugin & fetcher anatomy (worked files)

Everything here is quoted from `jobs-build/plugin-mix` (the minimal plugin)
and `jobs-build/fetcher-gomod` unless noted.

## Plugin repo layout

```
BUILD.jobs                       # builds the artifact (boilerplate, below)
fetch                            # bash wrapper so the artifact satisfies the fetcher contract too
fetchers.toml                    # companion fetcher pins
main.go                          # CBOR stdio entry point
mixlock.go                       # THE ecosystem-specific file (parser)
internal/importdef/definition.go # verbatim copy from jobs-iroh (no Platform/FetcherDef fields)
go.mod / go.sum / vendor/        # deps vendored so BUILD.jobs runs offline
*_test.go                        # plain go test
LICENSE                          # AGPL-3.0
README.md
```

## Engine-side protocol (jobs-iroh `runner/plugincaller.go`)

```go
type pluginRequest struct {
    Call   map[string]any    `cbor:"call"`           // the Starlark kwargs
    Source string            `cbor:"source"`         // "/jobs/source" (read-only mount)
    Deps   map[string]string `cbor:"deps,omitempty"` // /jobs/deps/<name>
    Dir    string            `cbor:"dir,omitempty"`  // consumer dir inside source
}
```

The runner execs `<artifact>/plugin` (fixed name), cwd = the artifact,
`PATH=<shell>/bin`, `HOME=/tmp`, namespaces incl. net (no network),
`/jobs/source` + `/jobs/store` read-only, tmpfs `/tmp`. An input spec is
recognized by the engine as a map of **exactly** `{"kind": "import"|"build",
"definition": <CBOR bytes>}`; imports naming non-seed fetchers get
`FetcherDef` injected from the plugin's `fetchers.toml` or hard-error.

## main.go skeleton (plugin-mix, condensed)

```go
var reqDec, _ = cbor.DecOptions{DefaultMapType: reflect.TypeOf(map[string]any(nil))}.DecMode()

type request struct {
    Call   map[string]any `cbor:"call"`
    Source string         `cbor:"source"`
}
type inputSpec struct {
    Kind       string `cbor:"kind"`
    Definition []byte `cbor:"definition"`
}
type pkgOut struct {
    Name    string    `cbor:"name"`
    Version string    `cbor:"version"`
    Input   inputSpec `cbor:"input"`
}

func pkgInput(p hexPackage) (inputSpec, error) {
    params, err := importdef.CanonicalParams(map[string]any{
        "name": p.Name, "version": p.Version, "sha256": p.Sha256,
    })
    if err != nil { return inputSpec{}, err }
    def, err := importdef.Definition{Fetcher: "hexpm", Params: params}.Canonical()
    if err != nil { return inputSpec{}, err }
    return inputSpec{Kind: "import", Definition: def}, nil
}

func run(in io.Reader, out io.Writer) error {
    raw, err := io.ReadAll(in)
    // reqDec.Unmarshal(raw, &req)
    var lock []byte
    switch v := req.Call["mix_lock"].(type) {   // coerce defensively
    case []byte: lock = v
    case string: lock = []byte(v)
    default:     return fmt.Errorf("mix_lock: unsupported type %T", v)
    }
    pkgs, err := parseMixLock(lock)             // sorted, deduped, hard errors
    // build []pkgOut, cbor.Marshal, write to out
}

func main() {
    if err := run(os.Stdin, os.Stdout); err != nil {
        fmt.Fprintln(os.Stderr, "plugin-mix:", err)
        os.Exit(1) // hard failure; 75 would be retryable
    }
}
```

`run(io.Reader, io.Writer)` — not direct `os.Stdin/Stdout` — so the
round-trip test can drive it.

## fetchers.toml

```toml
# Companion fetcher pins for the imports this plugin emits
# (JOBS recipe-declared-fetchers design §7). Bundled into the plugin artifact
# root AND its fetch output; the JOBS pin stage injects these as FetcherDef
# into every emitted import. Bump together with the plugin.
[[fetcher]]
name   = "hexpm"
url    = "https://github.com/jobs-build/fetcher-hexpm/archive/refs/tags/v0.1.0.tar.gz"
sha256 = "938fac757eab6ff1c2ea5f0eaa0e4c3d68e2c75bd74c1cdd0b668baf6286e9ba"
```

## fetch wrapper

```bash
#!/usr/bin/env bash
# mixplugin fetcher: emit the mix.lock build plugin executable (build.md §6).
set -euo pipefail
out="${JOBS_OUTPUT_DIR:?JOBS_OUTPUT_DIR not set}"
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

[ -x "$here/plugin" ] || {
	echo "mixplugin: prebuilt 'plugin' missing in the artifact" >&2
	exit 1
}
cp "$here/plugin" "$out/plugin"
cp "$here/fetchers.toml" "$out/fetchers.toml"
chmod +x "$out/plugin"
```

## BUILD.jobs (the plugin's own build — copy verbatim, change the name)

```starlark
# BUILD.jobs — build the JOBS plugin-mix artifact fully offline & CGO-free.

def build():
    goarch = {"linux/amd64": "amd64", "linux/arm64": "arm64"}[platform]
    go_sha = {
        "amd64": "1153d3d50e0ac764b447adfe05c2bcf08e889d42a02e0fe0259bd47f6733ad7f",
        "arm64": "ef758ae7c6cf9267c9c0ef080b8965f453d89ab2d25d9eb22de4405925238768",
    }
    toolchain = imp(fetcher = "tarball+https", params = {
        "url": "https://go.dev/dl/go1.26.4.linux-" + goarch + ".tar.gz",
        "sha256": go_sha[goarch],
    })
    script = """
export GOROOT=/build/goroot
mkdir -p "$GOROOT" "$out"
cp -dr "$(jq -r .toolchain <<<"$JOBS_DEPS")/go/." "$GOROOT/"
export PATH="$GOROOT/bin:$PATH"
export GOCACHE=/build/gocache GOTOOLCHAIN=local CGO_ENABLED=0 GOPROXY=off GOSUMDB=off
export GOFLAGS="-mod=$([ -d "$SRC/vendor" ] && echo vendor || echo mod)"
chmod -R u+w "$GOROOT" 2>/dev/null || true
cd "$SRC"
go build -o "$out/plugin" .
cp "$SRC/fetch" "$out/fetch"
cp "$SRC/fetchers.toml" "$out/fetchers.toml"
chmod +x "$out/plugin" "$out/fetch"
"""
    return struct(name = "mix plugin", inputs = {"toolchain": toolchain}, env = {}, script = script, runtime_deps = [])
```

## What varies between plugins

| | kwargs | emits | response | notes |
|---|---|---|---|---|
| plugin-go | `go_sum`, `go_mod`, `go_work`, `go_closure` | `gomod` `{module, version}` | array of `{path, version, input}` or `{modules, sources, closure}` | closure = pure-Go transitive import walk over the mounted source |
| plugin-uv | `uv_lock` | `pypi` `{url, filename, sha256, name, version}` | array + `kind: wheel\|sdist` | TOML parse, universal-wheel selection |
| plugin-mix | `mix_lock` | `hexpm` `{name, version, sha256}` | array | the minimal reference |
| plugin-cargo | `cargo_lock`, `workspace` | `cargocrate` | array or `{crates, sources, generated}` | workspace mode reads the source tree, prunes the lock, synthesizes files via `generated` |
| plugin-pybackend | `pyproject`, `platform` | `pypi` | array | not lockfile-driven: curated hardcoded backend pin table, arch-aware |

Only plugin-cargo (workspace) and plugin-go (closure) read `Source`/`Dir`;
the rest are pure bytes-in/specs-out.

## Fetcher repo layout

Shell flavor (fetcher-gomod): `BUILD.jobs`, `fetch` (bash), `LICENSE`
(AGPL-3.0), `README.md`. Go flavor (fetcher-pypi): `BUILD.jobs`, `main.go`,
`main_test.go`, `go.mod` (stdlib-only), `LICENSE`, `README.md` — its
BUILD.jobs ends `go build -o "$out/fetch" .`. The plugin receives its kwarg
params as canonical CBOR and emits them via `importdef.CanonicalParams`; the
engine renders those same params **as JSON** into `JOBS_FETCH_PARAMS` for the
fetcher — read fields with `jq -r '.name // empty'`, don't assume key order.

## Fetcher example (fetcher-gomod's fetch, core)

```bash
set -euo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
if [ -f "$here/env.sh" ]; then . "$here/env.sh"; fi   # baked GOROOT store path
out="${JOBS_OUTPUT_DIR:?JOBS_OUTPUT_DIR not set}"
params="${JOBS_FETCH_PARAMS:?JOBS_FETCH_PARAMS not set}"
module="$(jq -r '.module // empty' <<<"$params")"
version="$(jq -r '.version // empty' <<<"$params")"
…
if ! ( cd "$tmp/work" && go mod init jobs.local/gomodfetch >/dev/null 2>&1 || true
       go mod download -x "$module@$version" ); then
    echo "gomod: go mod download failed for $module@$version" >&2
    exit 75   # retryable — transient proxy/network
fi
…
# determinism scrub — the output tree is content-addressed:
rm -rf "$out/cache/download/sumdb"
find "$out/cache/download" \( -name '*.lock' -o -name list \) -delete 2>/dev/null || true
```

Go-binary fetchers (fetcher-pypi) instead end their BUILD.jobs with
`go build -o "$out/fetch" .` and encode exits as
`const (exitOK = 0; exitHard = 1; exitRetryable = 75)`.
