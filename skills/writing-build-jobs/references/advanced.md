# Advanced BUILD.jobs patterns

Patterns beyond the single-service recipes, extracted from
`jobs-build/gitlab-build` (8-component Cloud Native GitLab build),
`jobs-build/fizzy-build`, `wispers-client-build`, and `examples/subbuild`.

## Param-driven multi-component recipes (gitlab-build)

One `BUILD.jobs`, one component per invocation, selected by
`--param component=`:

```starlark
COMPONENTS = {
    # component: ([(binary, go package), …], smoke command, entrypoint binary, subdir)
    "pages": ([("gitlab-pages", ".")], '"$out/bin/gitlab-pages" -version', "gitlab-pages", ""),
    "workhorse": ([("gitlab-workhorse", "./cmd/gitlab-workhorse"), …],
                  '"$out/bin/gitlab-workhorse" -version', "gitlab-workhorse", "workhorse"),
}

def plugins():
    component = params["component"]
    if component not in COMPONENTS and component not in ["gitaly", "rails", "webservice"]:
        fail("unknown component %r (want one of %s)" % (component, …))
    …

def build():
    component = params["component"]
    if component == "rails":
        return rails_build(False)
    bins, smoke, entry_bin, subdir = COMPONENTS[component]
    …
    build_lines = "\n".join(['go build -o "$out/bin/%s" %s' % (b, p) for b, p in bins])
```

Validate params early in `plugins()` with `fail(...)` so a typo dies at
resolve time with the list of valid values. Each component keeps its own
`caches` id and entrypoint. **Always add an in-sandbox smoke test** between
build and entrypoint (`"$out/bin/x" -version` or `help >/dev/null`) — it
catches a mislinked binary before anything ships.

## Building upstream sources (source-as-input)

When the recipe repo does NOT contain the code (gitlab-build, fizzy-build,
wispers): the source is itself an import, staged like any dependency:

```starlark
def src_import():
    project, ref, _ = SATELLITES[component]
    return imp(fetcher = GITLABFETCHER, params = {"project": project, "ref": ref})
…
inputs["src"] = src_import()
# script:
SRCTREE=/build/src2
mkdir -p "$SRCTREE"
cp -dr "$(jq -r .src <<<"$JOBS_DEPS")/." "$SRCTREE/"
chmod -R u+w "$SRCTREE"
cd "$SRCTREE"
```

`fetcher-github`/`fetcher-gitlab` are ref-addressed (no sha256; the ref is
the trust anchor) — pin **commit SHAs or release tags**, never branches.

## Resolution deps: reading upstream lockfiles at pin time

The plugin needs the upstream `go.sum`/`Gemfile.lock`, but it lives in the
fetched tree, not the recipe repo. Declare the tree as a **resolution dep**
in `plugins()` and read it in `build()`:

```starlark
def plugins():
    return struct(
        plugins = {"go": imp(fetcher = GOPLUGIN, params = {})},
        deps    = {"src": src_import(), "gitlab": gitlab_import()},
    )

def build():
    mods = plugins["go"](go_sum = deps["src"].read("go.sum"))
```

Use the **identical import definition** for the resolution dep and the build
input — it's one graph node, fetched once.

## Self-checking satellite pins (gitlab-build)

When one upstream release declares versions of others, make static pins
assert against the master tree so drift fails loudly with the fix:

```starlark
def assert_satellite_pin(component):
    if component in SATELLITES:
        project, ref, vfile = SATELLITES[component]
        want = str(deps["gitlab"].read(vfile)).strip()
        if ref[1:] != want:
            fail("%s pin %s drifted: gitlab-foss %s declares %s = %s — update SATELLITES[%r]"
                 % (component, ref, GITLAB, vfile, want, component))
```

Bump procedure becomes mechanical: update the master pin, run any build,
copy expected versions out of the assertion failures. Same idea for
toolchains: assert `.ruby-version`/`go.mod` against the recipe's pin.

## Wrapper repos + inline sub-recipes (wispers-client-build)

A thin repo whose `BUILD.jobs` pins only an upstream ref and delegates to a
sibling recipe file shipped in the wrapper:

```starlark
WISPERS_REF = "<commit sha>"

def build():
    sub = bld(
        source    = imp(fetcher = GITHUB_OR_GITLAB, params = {"owner": …, "repo": …, "ref": WISPERS_REF}),
        buildJobs = str(source.read("wispers.BUILD.jobs")),
        platform  = platform,
    )
    …copy what you need out of the sub-build's output…
```

The sub-recipe runs against the *real upstream tree*, so its plugin reads the
upstream lockfile — no lockfile copies in the wrapper. Caveat: a sub-build's
runtime closure does **not** propagate through a plain file copy — re-declare
`runtime_deps` (e.g. `hostmusl`) in the outer recipe.

`subbuild()` is the intra-repo flavor: `subbuild("sub/app")` builds a
descendant dir; `subbuild("//lib/x")` builds any dir in the widened context
(sibling builds dedupe per commit). `bld(source = <sdist import>, buildJobs = GENERIC)`
is the per-dependency flavor (see python.md's sdist pattern).

## Version stamping and build flags

Derive versions from the source at pin time rather than hardcoding:

```starlark
module = str(deps["src"].read("go.mod")).splitlines()[0].split(" ")[1].strip()
version = str(deps["src"].read("VERSION")).strip()
# → go build -ldflags "-X ${module}/internal/version.version=${version}" …
```

gitaly even parses commit refs out of the upstream Makefile at pin time to
build its two bundled gits — self-deriving pins beat manual ones wherever the
source declares the truth.

## Scaling lessons (the ~950-import builds)

- Every recipe touching many inputs needs the `JOBS_DEPS_FILE` fallback line
  (herestrings don't traverse execve; the env var is dropped >64 KiB).
- Precompile/pre-seed "musl monster" dependencies (grpc, nokogiri-class gems,
  fat crates) into caches or sub-builds; keep the main script's critical path
  short.
- `resources = struct(cpu = "4", memory = "16Gi")` for monolith components;
  parallelism flags (`-j`, `bundle install --jobs`) are the real memory
  control since the cgroup caps memory, not CPU.
- Shared caches with named subpaths (`/gem-cache/webpack`) let several
  components reuse one cache id without collisions.
