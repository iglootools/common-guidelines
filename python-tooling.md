# Python Tooling Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

This file covers the toolchain: how a Python project is built, how its environment is created, and
what tasks it exposes. For the language itself see [python.md](python.md); for the setup steps every
project shares see [project-setup.md](project-setup.md); for editor configuration see
[ide.md](ide.md).

## Build and Packaging

### Build backend

Build tool: `uv`, with `hatchling` as the build backend and
[`uv-dynamic-versioning`](https://github.com/ninoseki/uv-dynamic-versioning) for
git-tag-derived versions. Commit `uv.lock`.

```toml
[build-system]
requires = ["hatchling", "uv-dynamic-versioning"]
build-backend = "hatchling.build"

[tool.hatch.version]
source = "uv-dynamic-versioning"
```

`uv-dynamic-versioning` is powered by the same `dunamai` library as the
`poetry-dynamic-versioning` plugin used before, so version strings are identical across the
switch — `0.6.1` on a tagged commit, `0.6.1.post6.dev0+2a99beb` off one. It is a third-party
plugin; `uv`'s own `uv_build` backend cannot derive versions from a VCS at all
([astral-sh/uv#14561](https://github.com/astral-sh/uv/issues/14561)), and `hatch-vcs` would
change the dev-version format.

### Pin the build backend explicitly

`uv.lock` does **not** cover PEP 517 build dependencies: `uv build` resolves them fresh from PyPI
on a cold cache. `[tool.uv] build-constraint-dependencies` is the only pin, and it must cover the
whole closure — most importantly `dunamai`, which is what computes the version. Regenerate with:

```bash
printf '%s\n' hatchling uv-dynamic-versioning \
  | uv pip compile - --no-annotate --no-header --quiet
```

Leave `[build-system].requires` unpinned so the two can never contradict each other. No
built-in Renovate manager reads `build-constraint-dependencies`, so these pins rot silently
unless a `customManagers` regex entry is added for them.

### Guard against a `0.0.0` release

With a VCS-derived version, `[tool.uv-dynamic-versioning] fallback-version` is required — Renovate
runs `uv lock --upgrade-package`, which builds with no git context, and without a fallback every
such PR fails. The cost is that "no git history" becomes a *successful* build producing
`0.0.0`: verified by building from a source tree with `.git` removed, which yields
`<pkg>-0.0.0-py3-none-any.whl` and exits 0. Nothing else in the publish path catches this, and
PyPI never allows a version to be re-uploaded. So provide a `build-verify` task that reads the
built wheel's own `METADATA` and fails on the fallback version, and wire it into the CI build.
Read the metadata rather than pattern-matching filenames — a legitimate `1.0.0.dev0` contains
`0.0.0` as a substring — and fail when the directory contains no wheel at all, or the guard is
defeated by the very situation it exists to catch.

Building the wheel *from an sdist* is safe, contrary to expectation: hatchling writes the
resolved version into the sdist's `PKG-INFO` and reads it back, verified by building from an
extracted sdist with no `.git`. `uv build --sdist --wheel` (both from source) is still mildly
preferable at no cost, but it is not a workaround for a broken path.

### Know what hatchling packages by default

Which is more than poetry did:

| Default | Consequence |
|---|---|
| sdist includes every file not ignored by the VCS | ships `tests/`, `docs/`, `.github/` and the lockfiles unless constrained with `[tool.hatch.build.targets.sdist] only-include` |
| wheel contents inferred from a `<NAME>/__init__.py` heuristic | when the heuristic misses, the result is a silently *empty* wheel rather than an error — so set `[tool.hatch.build.targets.wheel] packages` explicitly |
| `.gitignore` is the only ignore file read — not `.git/info/exclude`, not `core.excludesfile` | a file ignored only locally is published. See the `.gitignore` rule under [All Projects](project-setup.md#all-projects) |

Prove parity rather than assuming it when migrating a build backend: diff the wheel payload and
sdist file lists against the previous toolchain's at the same tag, and install the wheel and
exercise anything read through `importlib.resources` or `Path(__file__).parent`, which a
file-list diff does not cover.

### Keep the version honest in editable installs

Code that reads its own version through `importlib.metadata` gets whatever was frozen into
`.dist-info` at install time, so a VCS-derived version goes stale on the next commit. Fix it with:

```toml
[tool.uv]
cache-keys = [{ file = "pyproject.toml" }, { git = { commit = true, tags = true } }]
```

This *replaces* the default `[{ file = "pyproject.toml" }]` rather than extending it, hence the
explicit file entry. Note the interaction with `[deps.uv]`: a plain commit changes neither
`uv.lock` nor `pyproject.toml`, so no automatic sync fires and the staleness persists until
something forces one. An uncommitted tree also still reports the last commit's version —
`dunamai`'s default format carries no dirty marker.

## Mise and uv

### Configuration

For uv projects mise documents `python.uv_venv_auto` in `[settings]`, *not* `_.python.venv` in
`[env]` — the latter is for projects that do not use uv, and the two are separate code paths, so
this is a replacement rather than an addition. Pair it with `[deps.uv]` so mise's dependency engine
owns installation:

```toml
[tools]
python = "3.14.3"
uv = "0.12.3"

[env]
UV_PYTHON = { value = "{{ tools.python.path }}", tools = true }
UV_PYTHON_DOWNLOADS = "never"
UV_PYTHON_PREFERENCE = "only-system"

[settings]
lockfile = true
experimental = true            # required by `mise deps`
python.uv_venv_auto = "source"     # creation is [deps.uv]'s job, not mise's

[deps.uv]
auto = true
run = "uv sync"                # add --all-extras if the project has extras
```

| Setting | Why | Consequence if missing |
|---|---|---|
| `UV_PYTHON` as an **absolute path** | mise's docs are explicit that a bare version number "does not guarantee that uv uses the specific interpreter managed by mise" | uv may silently build the venv on a system or self-managed Python of the same version; a raised `[tools] python` pin also stops recreating `.venv`, since the old interpreter still satisfies `requires-python` |
| `experimental = true` in the committed `mise.toml` | `mise deps` is experimental, and relying on each developer exporting `MISE_EXPERIMENTAL` makes a fresh clone behave differently from CI | dependencies mysteriously do not install, with no error naming the cause |
| `[deps.uv] auto = true` | runs `uv sync` before any `mise run`, gated on a blake3 hash of `uv.lock` + `pyproject.toml` and on `.venv` existing | every task needs a manual `install` first, or silently runs against a stale environment |
| `[deps.uv] run = "..."` | the built-in command is a bare `uv sync`, which misses `--all-extras` | extras are absent from the dev environment. Note that overriding `run` **keeps** the built-in source hashing and `.venv` tracking — only `install_command()` reads it |
| `python.uv_venv_auto = "source"`, not `"create\|source"` | mise's cookbook is explicit that with `[deps.uv]` enabled, creating the venv is the deps provider's job — keep the setting at `"source"`, enable `[deps.uv]`, and let `mise deps` create it | the two mechanisms collide. Measured on a fresh clone with `"create\|source"`, `mise install` announced `creating venv with uv at: …` and then failed on its own result: `error: Failed to create virtual environment / Caused by: A virtual environment already exists at: .venv`, then `WARN uv venv creation failed`. With `"source"` the same clone installs silently and the first `mise run` lets `[deps.uv]` create and populate `.venv` |
| `UV_LOCKED: 1` in CI job env | asserts a sync will not change `uv.lock` | a stale lockfile is papered over by an implicit re-resolve on the runner. Prefer the environment variable over a `--locked` flag, so the `[deps.uv]` command stays identical locally and in CI and only the strictness differs |

A consequence of leaving creation to `[deps.uv]` is that on a genuinely fresh clone `.venv` does
not exist until the first `mise run` — `mise install` alone does not create it. That is by design
rather than a gap, but it means nothing is on `PATH` for that first run, which is one of the two
reasons for the next rule.

### Invoke tools through `uv run --no-sync <tool>` in mise tasks, not bare

mise activates the project's `.venv`, but it does not remove a *foreign* `.venv/bin` that is
already on `PATH`, so in a shell where another project's virtualenv is active a bare `ruff`
silently runs that project's copy. Measured in a multi-project checkout: the current project's venv
landed at `PATH` position 16 and the foreign one at position 1. `uv run` resolves the environment
from the project root and ignores `PATH`, so it is immune; `--no-sync` keeps installation the
`[deps.uv]` provider's job alone. It was not measurably slower than a bare invocation.

State the same rule in the project's own docs. Under `poetry run` the requirement for an
activated environment was implicit in every command; without a prefix it has to be written
down, including `mise exec -- <tool>` as the way to run something from an unactivated shell.

### Adding and removing dependencies

Add dependencies with `uv add` / `uv remove`, which update `pyproject.toml` and `uv.lock`
together. `mise deps add` does not support uv — its ecosystems are npm, yarn, pnpm, bun, deno,
aube, dart and flutter — so do not go looking for a mise verb.

## Code Quality Tools

Use `ruff` for linting and formatting, `pyright` for type-checking, and `vermin` for validating the desired Python version compatibility.

## Mise Tasks

Consider providing:
- `configdocs`/`configdocs-check`, and `depgraph`/`depgraph-check` tasks for config and dependency documentation, respectively
- `format`, `lint`, `type-check`, `compat-check` tasks for code quality checks

### Lock checks

`lock-check` and `lock-check-uv` (`uv lock --check`) tasks assert different things:
`uv lock --check` asks whether the lock is consistent with the manifest, while `UV_LOCKED` in
CI asks whether a sync would *change* it.

`mise.lock` has no `--check` equivalent, so that one has to regenerate and compare — and it
should do so **in a scratch copy, never in place**. Copy
[`scripts/lock-check.sh`](scripts/lock-check.sh) into the project and reduce the task to
`run = "./scripts/lock-check.sh"`.

Keep it as a script rather than inlining it as a TOML string. Anything that needs
`set -euo pipefail`, a `trap` and a subshell has outgrown a `run =` value: a file gets a
shebang and therefore `pipefail` (mise tasks otherwise run under POSIX `sh`, where
`<(...)` is a syntax error that does *not* reliably fail the task), it can be invoked and
tested directly, and there is room for the reasoning below next to the code it explains.

Every line of that script is load-bearing, and the reasons are not guessable:

| Detail | Why |
|---|---|
| regenerate in `$tmp` | `mise lock` always writes. Letting it write the real lockfile means undoing it afterwards, and a `git checkout mise.lock` to do so **discards the regenerated file the error message just told you to commit** |
| inputs from the index (`git show :<file>`) | mise rewrites `mise.lock` on its own: with `lockfile = true`, any tool-resolving command updates the version stanza and drops the per-platform checksums it can no longer vouch for — which is exactly the state that makes `mise install --locked` fail on a fresh runner. The working copy is therefore not a stable reference, so "up to date" can only mean "matches what is staged". It also makes the check immune to CI's `mise use python@<matrix>`, which rewrites the working `mise.toml` |
| `env -u MISE_PYTHON_VERSION` | `mise lock` honours it, so an env-selected matrix interpreter would otherwise be locked in place of the committed pin |
| `set -euo pipefail` | without it a failing `mise lock` leaves the copied lockfile untouched, the `diff` finds no difference, and the check **passes** — the guard defeated by the situation it exists to catch |

The threshold cuts both ways: a task body that is two or three straight-line commands with
nothing to clean up and no failure to interpret — like `reinstall` below — stays inline,
where a reader sees what it does without opening a second file.

### Reinstalling the environment

A `reinstall` task that deletes `.venv` and reinstalls from scratch. Most of what this used
to be for is gone: `uv sync` is exact by default, so it removes packages present in neither
`pyproject.toml` nor `uv.lock`, and the `UV_PYTHON` pin above is what picks up a raised
`[tools] python`. It survives for the one case `[deps.uv]`'s hashing cannot see — a venv that
is *dirty* rather than *stale*, after someone ran `uv pip install` into it by hand:

```toml
[tasks.reinstall]
description = "Delete .venv and reinstall dependencies from scratch"
run = """
rm -rf "$MISE_PROJECT_ROOT/.venv"
mise deps install uv --force
"""
```

Keep `install` as a named task delegating to `mise deps install uv`, rather than calling
`uv sync` directly. `mise deps` is experimental, and that indirection is what makes reverting
to a plain `uv sync` a one-line change in one place.

## Console Output

Preferably print to the console using the [Rich](https://github.com/textualize/rich) library that is bundled with [Typer](https://typer.tiangolo.com/)
