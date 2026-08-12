# Tooling Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

## GitHub Workflows
- Whenever safe (i.e. not affecting production), enable `workflow_dispatch` and `repository_dispatch` to allow manual triggering of workflows from the GitHub UI or CLI, which is useful for testing and debugging.
- Use OpenID Connect (OIDC) authentication for publishing to PyPI, and set up a separate workflow for testing releases to Test PyPI. This allows testing the release and publish process without affecting the real PyPI index, and provides more detailed logs for debugging.
  - When the Test PyPI workflow synthesizes a throwaway version, derive it from the **base
    release** version, not from the full VCS-derived one. Appending `.devYYYYMMDDHHMMSS` to a
    version that already carries `.dev0` produces two dev segments and is invalid PEP 440 — a bug
    that stays hidden as long as the workflow is only ever dispatched from a tag. Reading the tag
    (`git describe --tags --abbrev=0`) avoids it and needs no build tool.
- **Prefer tools whose mise backend is lockable.** Check with `mise registry <tool>`: a backend
  such as `aqua:` records per-platform URLs and checksums in `mise.lock`, while `vfox:` records
  none, and `mise lock` cannot generate them. An unlockable tool makes `mise install --locked`
  fail on a fresh runner, which forces `install: false` plus a bare `mise install` into every
  workflow — quietly giving up checksum and attestation verification for *all* tools to
  accommodate one. The one legitimate use of `install: false` is a workflow that runs on branches
  where `mise.toml` is deliberately ahead of `mise.lock`, such as one that regenerates the lock
  for Renovate branches.
- Set `timeout-minutes` on every job. Without it, a hung step (a stalled `apt-get`, a network call that never returns) runs until GitHub's 6-hour default before the job is killed, wasting CI minutes and delaying feedback. A tight job-level guard (e.g. `timeout-minutes: 10`, sized to the job) fails fast and legibly. Prefer a single job-level timeout over per-step timeouts: one guard covers the whole job with no per-step bookkeeping.

## New Project Setup

Guidelines to follow when setting up new projects.

### All Projects:
- `renovate.json`: group all dependency updates into a single PR, delay
  updates by 14 days (`minimumReleaseAge`) to avoid adopting broken
  releases and limit risk of supply chain attacks
- **Split Renovate and Dependabot by job, not by ecosystem.** Running both on version updates
  produces duplicate PRs, and Dependabot's half bypasses the 14-day delay above. Give Renovate
  all routine version updates — it also covers what Dependabot has no manager for, such as
  `mise.toml` and `customManagers` regex entries — and declare Dependabot security-only:

  ```yaml
  updates:
    - package-ecosystem: "uv"
      directory: "/"
      schedule:
        interval: "weekly"
      open-pull-requests-limit: 0
  ```

  `open-pull-requests-limit: 0` is the documented way to disable *version* updates, and security
  PRs are exempt from both it and `cooldown`. That division is the point: a deliberate
  `minimumReleaseAge` delay is a supply-chain measure, and it is exactly the wrong response to a
  published advisory — so the delayed tool handles routine upgrades and the immediate one handles
  vulnerabilities.

  **`package-ecosystem` must track the build tool.** A stale value fails *silently*: `pip` reads
  `poetry.lock` and understands neither `uv.lock` nor `[dependency-groups]`, so after a migration
  it keeps running, finds nothing, and reports green — indistinguishable from working coverage.
- **Ignore OS and editor cruft in the committed `.gitignore`**, not in `.git/info/exclude` or a
  personal `core.excludesfile`. Those are not shared on clone, so they look like coverage to
  whoever set them up while giving none to anyone else. This is not cosmetic: hatchling reads only
  `.gitignore` when selecting files to package, so a `.DS_Store` inside the package directory that
  is ignored only locally gets published inside the wheel.
- Github build/test/release workflows
- `git config user.email "<email>"` and `git config user.name "<name>"` in the project.

### Python Projects
- Build tool: `uv`, with `hatchling` as the build backend and
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
- **Mise + uv.** For uv projects mise documents `python.uv_venv_auto` in `[settings]`, *not*
  `_.python.venv` in `[env]` — the latter is for projects that do not use uv, and the two are
  separate code paths, so this is a replacement rather than an addition. Pair it with
  `[deps.uv]` so mise's dependency engine owns installation:

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
  python.uv_venv_auto = "create|source"

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
  | `UV_LOCKED: 1` in CI job env | asserts a sync will not change `uv.lock` | a stale lockfile is papered over by an implicit re-resolve on the runner. Prefer the environment variable over a `--locked` flag, so the `[deps.uv]` command stays identical locally and in CI and only the strictness differs |

  On a genuinely fresh clone, the first `mise install` warns `uv is required to create the venv
  … but is not installed` and does not create it — env directives resolve before the toolset is
  available. This is harmless: the first `mise run` invokes `[deps.uv]`, whose `uv sync` creates
  `.venv` itself. It does, however, mean nothing is on `PATH` for that first run, which is one of
  the two reasons for the next rule.
- **Invoke tools through `uv run --no-sync <tool>` in mise tasks, not bare.** mise activates the
  project's `.venv`, but it does not remove a *foreign* `.venv/bin` that is already on `PATH`, so
  in a shell where another project's virtualenv is active a bare `ruff` silently runs that
  project's copy. Measured in a multi-project checkout: the current project's venv landed at
  `PATH` position 16 and the foreign one at position 1. `uv run` resolves the environment from
  the project root and ignores `PATH`, so it is immune; `--no-sync` keeps installation the
  `[deps.uv]` provider's job alone. It was not measurably slower than a bare invocation.

  State the same rule in the project's own docs. Under `poetry run` the requirement for an
  activated environment was implicit in every command; without a prefix it has to be written
  down, including `mise exec -- <tool>` as the way to run something from an unactivated shell.
- Use `ruff` for linting and formatting, `pyright` for type-checking, and `vermin` for validating the desired Python version compatibility.
- Add dependencies with `uv add` / `uv remove`, which update `pyproject.toml` and `uv.lock`
  together. `mise deps add` does not support uv — its ecosystems are npm, yarn, pnpm, bun, deno,
  aube, dart and flutter — so do not go looking for a mise verb.
- **Pin the build backend explicitly.** `uv.lock` does **not** cover PEP 517 build
  dependencies: `uv build` resolves them fresh from PyPI on a cold cache. `[tool.uv]
  build-constraint-dependencies` is the only pin, and it must cover the whole closure — most
  importantly `dunamai`, which is what computes the version. Regenerate with:

  ```bash
  printf '%s\n' hatchling uv-dynamic-versioning \
    | uv pip compile - --no-annotate --no-header --quiet
  ```

  Leave `[build-system].requires` unpinned so the two can never contradict each other. No
  built-in Renovate manager reads `build-constraint-dependencies`, so these pins rot silently
  unless a `customManagers` regex entry is added for them.
- **Guard against a `0.0.0` release.** With a VCS-derived version,
  `[tool.uv-dynamic-versioning] fallback-version` is required — Renovate runs
  `uv lock --upgrade-package`, which builds with no git context, and without a fallback every
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
- **Know what hatchling packages by default**, which is more than poetry did:

  | Default | Consequence |
  |---|---|
  | sdist includes every file not ignored by the VCS | ships `tests/`, `docs/`, `.github/` and the lockfiles unless constrained with `[tool.hatch.build.targets.sdist] only-include` |
  | wheel contents inferred from a `<NAME>/__init__.py` heuristic | when the heuristic misses, the result is a silently *empty* wheel rather than an error — so set `[tool.hatch.build.targets.wheel] packages` explicitly |
  | `.gitignore` is the only ignore file read — not `.git/info/exclude`, not `core.excludesfile` | a file ignored only locally is published. See the `.gitignore` rule under [All Projects](#all-projects) |

  Prove parity rather than assuming it when migrating a build backend: diff the wheel payload and
  sdist file lists against the previous toolchain's at the same tag, and install the wheel and
  exercise anything read through `importlib.resources` or `Path(__file__).parent`, which a
  file-list diff does not cover.
- **Keep the version honest in editable installs.** Code that reads its own version through
  `importlib.metadata` gets whatever was frozen into `.dist-info` at install time, so a
  VCS-derived version goes stale on the next commit. Fix it with:

  ```toml
  [tool.uv]
  cache-keys = [{ file = "pyproject.toml" }, { git = { commit = true, tags = true } }]
  ```

  This *replaces* the default `[{ file = "pyproject.toml" }]` rather than extending it, hence the
  explicit file entry. Note the interaction with `[deps.uv]`: a plain commit changes neither
  `uv.lock` nor `pyproject.toml`, so no automatic sync fires and the staleness persists until
  something forces one. An uncommitted tree also still reports the last commit's version —
  `dunamai`'s default format carries no dirty marker.
- Consider providing:
  - `configdocs`/`configdocs-check`, and `depgraph`/`depgraph-check` tasks for config and dependency documentation, respectively
  - `format`, `lint`, `type-check`, `compat-check` tasks for code quality checks
  - `lock-check` and `lock-check-uv` (`uv lock --check`) tasks. These assert different things:
    `uv lock --check` asks whether the lock is consistent with the manifest, while `UV_LOCKED` in
    CI asks whether a sync would *change* it.

    `mise.lock` has no `--check` equivalent, so that one has to regenerate and compare — and it
    should do so **in a scratch copy, never in place**:

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    cd "$(git rev-parse --show-toplevel)"

    tmp=$(mktemp -d)
    trap 'rm -rf "$tmp"' EXIT

    git show :mise.toml > "$tmp/mise.toml"
    git show :mise.lock > "$tmp/staged.lock"
    cp "$tmp/staged.lock" "$tmp/mise.lock"
    ( cd "$tmp" && MISE_TRUSTED_CONFIG_PATHS="$tmp" env -u MISE_PYTHON_VERSION mise lock )

    if ! diff -u "$tmp/staged.lock" "$tmp/mise.lock" >&2; then
      echo "mise.lock is out of date. Run 'mise lock' and commit the result." >&2
      exit 1
    fi
    ```

    Keep it in a script (`scripts/lock-check.sh`) with the task reduced to
    `run = "./scripts/lock-check.sh"`, rather than inlining it as a TOML string. Anything that
    needs `set -euo pipefail`, a `trap`, and a subshell has outgrown a `run =` value: the
    reasoning below does not fit in one, a file gets a shebang and `pipefail` (mise tasks
    otherwise run under POSIX `sh`, where `<(...)` is a syntax error that does *not* reliably
    fail the task), and it can be run directly and tested.

    Each line of it is load-bearing, and the reasons are not guessable:

    | Detail | Why |
    |---|---|
    | regenerate in `$tmp` | `mise lock` always writes. Letting it write the real lockfile means undoing it afterwards, and a `git checkout mise.lock` to do so **discards the regenerated file the error message just told you to commit** |
    | inputs from the index (`git show :<file>`) | mise rewrites `mise.lock` on its own: with `lockfile = true`, any tool-resolving command updates the version stanza and drops the per-platform checksums it can no longer vouch for — which is exactly the state that makes `mise install --locked` fail on a fresh runner. The working copy is therefore not a stable reference, so "up to date" can only mean "matches what is staged". It also makes the check immune to CI's `mise use python@<matrix>`, which rewrites the working `mise.toml` |
    | `env -u MISE_PYTHON_VERSION` | `mise lock` honours it, so an env-selected matrix interpreter would otherwise be locked in place of the committed pin |
    | `set -euo pipefail` | without it a failing `mise lock` leaves the copied lockfile untouched, the `diff` finds no difference, and the check **passes** — the guard defeated by the situation it exists to catch |
  - a `reinstall` task that deletes `.venv` and reinstalls from scratch. Most of what this used
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
- Preferably print to the console using the [Rich](https://github.com/textualize/rich) library that is bundled with [Typer](https://typer.tiangolo.com/)

### CLI Projects
Consider providing:
- `asciinema` demo
- auto-generated `clidocs`/`clidocs-check` tasks for CLI documentation
- Provide preflight checks wherever applicable and troubleshooting instructions with the specific commands that need to be executed to fix the problem.

## IDE

### Pyright environment resolution

Pin the project's virtualenv in `pyproject.toml`, so pyright resolves third-party imports the same
way no matter how it was launched:

```toml
[tool.pyright]
venvPath = "."
venv = ".venv"
```

Pyright finds third-party packages through a Python environment, and with nothing configured it falls
back to the `python` on `PATH`. That is the asymmetry behind a confusing symptom: `mise run
type-check` passes, while an editor reports `Import "<dep>" could not be resolved` on the same file.
Tasks run in an activated environment; a language server spawned by an editor inherits no such
activation and lands on the system python, where none of the project's dependencies exist. The two
settings remove the ambiguity for every consumer at once — Pylance, the Claude Code pyright server,
the CLI, and CI. Since uv creates the project environment at `.venv` by default, these keys restate
that default for the consumers that cannot discover it.

Both keys are needed: `venv` is a directory *name* looked up inside `venvPath`, not a path.
`pythonVersion` is a separate concern — it is the language level to check against, deliberately
independent of the interpreter in `.venv` (see the
[Python Version Policy](python.md#python-version-policy)); a project can develop on a newer
interpreter while checking against its supported floor.

Verify it the way a language server sees it, with the venv deliberately off `PATH`. An activated
shell — or a `uv run` prefix — masks the exact failure being tested:

```bash
env -u VIRTUAL_ENV PATH=/usr/bin:/bin ./.venv/bin/pyright <file importing a dependency>
```

Expect `0 errors`. With `--verbose`, the search paths should include
`.venv/lib/python<X.Y>/site-packages`.

### VSCode
- Commit the extension set to `.vscode/extensions.json` so a fresh clone is prompted for it, instead
  of documenting it in prose that nobody reads before opening the editor:

    ```json
    {
        "recommendations": [
            "ms-python.python",
            "ms-python.vscode-pylance",
            "ms-python.vscode-python-envs",
            "charliermarsh.ruff",
            "hverlin.mise-vscode",
            "tombi-toml.tombi"
        ]
    }
    ```

  | Extension | Needed for | Consequence if missing |
  |---|---|---|
  | `ms-python.python` | every Python project | no test discovery, debugger, or terminal activation |
  | `ms-python.vscode-pylance` | every Python project | the language server. Without it the Python extension falls back to Jedi and `[tool.pyright]` is ignored outright |
  | `ms-python.vscode-python-envs` | projects committing `python-envs.*` settings | owns those keys; without it the committed `pythonProjects` entry does nothing |
  | `charliermarsh.ruff` | projects using ruff | `editor.defaultFormatter` names it, so format-on-save silently stops working |
  | `hverlin.mise-vscode` | projects using mise | the `mise.enable` setting has no reader; tasks and tool versions stop surfacing |
  | `tombi-toml.tombi` | projects with TOML — in practice all of them, via `pyproject.toml` and `mise.toml` | no TOML validation or formatting |

  The rule behind the list: **every extension a committed setting depends on belongs in
  `extensions.json`.** A setting whose extension is absent is not an error — it is a silent no-op, and
  the developer who cloned the repo has no way to tell the difference from a working setup.
- Check the in-project venv into `.vscode/settings.json` rather than relying on each developer
  running **Python: Select Interpreter**:

    ```json
    {
        "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
        "python-envs.pythonProjects": [
            {
                "path": ".",
                "envManager": "ms-python.python:venv"
            }
        ]
    }
    ```

  Because `pythonProjects` entries are committed, an `envManager` of `ms-python.python:system` —
  pointing the workspace at the system interpreter rather than `.venv` — persists for everyone who
  clones, long after whoever wrote it moved on, so it is worth stating explicitly. Type-checking is
  already covered by `[tool.pyright]` above; these settings are what the *rest* of the Python
  extension reads — test discovery, the debugger, and the environment new terminals activate.

  **`packageManager` is deliberately omitted.** `ms-python.vscode-python-envs` (checked at 1.36.0)
  registers only `conda`, `pip`, `pipenv`, `poetry`, `pyenv`, `system` and `venv` — there is no
  `ms-python.python:uv`. Naming `:poetry` would point at a tool the project no longer uses, so the
  key is left to default to `python-envs.defaultPackageManager` (`:pip`), which is inaccurate but
  inert: it drives only the extension's own install/uninstall UI, not interpreter resolution or type
  checking. Re-add it once the extension registers a uv package manager.
- After changing interpreter or pyright settings, reload the window (**Developer: Reload Window**).
  Current Pylance releases have no "Python: Restart Language Server" command, and an already-running
  server keeps serving diagnostics from the configuration it started with.

### Claude Code

**Pyright LSP plugin.** Install it in every Python project so Claude resolves symbols instead of
grepping for them:

```bash
claude plugin install pyright-lsp@claude-plugins-official --scope project
```

It provides go-to-definition, find-references, hover types, document and workspace symbol search,
and call hierarchy, and it pushes diagnostics into Claude's context after each edit. Where the same
name appears in several modules, this is the difference between an answer grounded in the import
graph and one inferred from text matches.

**Multi-project workspaces.** Opening several projects in one window — or reaching them as additional
working directories — does not degrade resolution, *provided each project pins its own venv* with
[`venvPath`/`venv`](#pyright-environment-resolution). This holds even when the session itself is
rooted somewhere unrelated to any of them:

- **One server can cover several projects.** There is no guarantee of one server per project, and the
  binary serving them all is whichever project's `.venv` supplied it first.
- **Imports still resolve per project.** Each file is checked against the venv its own project pins,
  so one project's dependencies never stand in for another's.
- **`goToDefinition` and `findReferences` stay correct**, including for a name defined in more than
  one project — they follow the import graph rather than the name, so a call lands in the definition
  its own project imports.

**Picking up configuration changes.** The plugin's pyright server reads `[tool.pyright]` at startup,
so edits to it do not reach a running session. Restart the editor window (or the Claude Code session)
and let the plugin spawn the server itself. Do not kill the server process expecting a respawn: the
plugin does not restart it on demand, and it then reports `server is running` for a process that no
longer exists, failing every LSP request until the session is reloaded. Note that this server is
distinct from the editor's own — VSCode runs Pylance for the squiggles and the plugin runs its own
pyright for Claude, so a stale diagnostic on one side says nothing about the other.

**Sharing the plugin set with the team.** `--scope project` is what makes the install shared: it
writes `enabledPlugins` into the repository's `.claude/settings.json` instead of your own
`~/.claude/settings.json`, so everyone who clones the repo gets the same plugins rather than each
developer installing them by hand. **Commit that file** — only `.claude/settings.local.json` is
normally ignored, so it is easy to leave the shared half untracked and never notice, since your own
user-scope install keeps the plugin working locally.

Plugins declared this way come from the repository rather than from the developer, so they load only
after the workspace trust dialog is accepted — LSP servers in particular start only once the
workspace is trusted.
