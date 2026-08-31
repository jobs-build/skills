# jobs-build skills

Agent skills for the [JOBS](https://github.com/draganm/jobs) hermetic build
system, distilled from the recipes across the `jobs-build` org
(`examples`, `jobs-iroh`, `gitlab-build`, `fizzy-build`,
`wispers-client-build`, the `plugin-*`/`fetcher-*` repos) and the
production recipes in `validator-watch`.

| Skill | Use it to |
|---|---|
| [`writing-build-jobs`](skills/writing-build-jobs/SKILL.md) | Author `BUILD.jobs` recipes — Go, Rust, Python, Elixir/Phoenix, Ruby/Rails, and multi-component/upstream-source builds |
| [`writing-jobs-plugins`](skills/writing-jobs-plugins/SKILL.md) | Create new JOBS plugins (lockfile → imports) and fetchers (params → fetched tree) for additional ecosystems |

## Install

As a Claude Code plugin:

```sh
claude plugin marketplace add jobs-build/skills
claude plugin install jobs-build-skills
```

Or copy/symlink the skill directories into `~/.claude/skills/` (also
recognized at `~/.agents/skills/` by other runtimes):

```sh
git clone https://github.com/jobs-build/skills
ln -s "$PWD/skills/skills/writing-build-jobs" ~/.claude/skills/
ln -s "$PWD/skills/skills/writing-jobs-plugins" ~/.claude/skills/
```

## Layout

Each skill is a `SKILL.md` (workflow, quick reference, common mistakes) plus
`references/` with per-language worked examples and the authoritative recipe
API extracted from `jobs-iroh`'s `recipe/` + `builddef/` packages. Pins
quoted in the references were current as of 2026-08-31; verify sha256s by
downloading before reusing them in new recipes.
