# Tooling Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

## GitHub Workflows
- Whenever safe (i.e. not affecting production), enable `workflow_dispatch` and `repository_dispatch` to allow manual triggering of workflows from the GitHub UI or CLI, which is useful for testing and debugging.
- Use OpenID Connect (OIDC) authentication for publishing to PyPI, and set up a separate workflow for testing releases to Test PyPI. This allows testing the release and publish process without affecting the real PyPI index, and provides more detailed logs for debugging.
- Set `timeout-minutes` on every job. Without it, a hung step (a stalled `apt-get`, a network call that never returns) runs until GitHub's 6-hour default before the job is killed, wasting CI minutes and delaying feedback. A tight job-level guard (e.g. `timeout-minutes: 10`, sized to the job) fails fast and legibly. Prefer a single job-level timeout over per-step timeouts: one guard covers the whole job with no per-step bookkeeping.

## New Project Setup

Guidelines to follow when setting up new projects.

### All Projects:
- `renovate.json`: group all dependency updates into a single PR, delay
  updates by 14 days (`minimumReleaseAge`) to avoid adopting broken
  releases and limit risk of supply chain attacks
- Github build/test/release workflows
- `git config user.email "<email>"` and `git config user.name "<name>"` in the project.

### Python Projects
- Build tool: poetry with poetry-dynamic-versioning
- Mise + Poetry: Use poetry `virtualenvs.in-project = true` with mise `_.python.venv = { path = ".venv", create = true }` to ensure that the virtual environment is created inside the project directory and automatically activated when running commands with `mise run`.
- Use `ruff` for linting and formatting, `pyright` for type-checking, and `vermin` for validating the desired Python version compatibility.
- Consider providing:
  - `configdocs`/`configdocs-check`, and `depgraph`/`depgraph-check` tasks for config and dependency documentation, respectively
  - `format`, `lint`, `type-check`, `compat-check` tasks for code quality checks
  - a `reinstall` task that deletes `.venv` and reinstalls from scratch. `install` only reconciles the
    venv with the lock file; it never removes distributions that arrived by other means, so a venv can
    accumulate packages present in neither `pyproject.toml` nor `poetry.lock` — which lets an
    undeclared import type-check and run locally, then fail in CI. `poetry check --lock` does not
    catch this: it compares the lock to the manifest, not to the venv. Deleting the directory is also
    what picks up a raised `[tools] python` pin, since an existing venv keeps the interpreter it was
    built with. Delegate to the `install` task rather than calling poetry directly, so mise recreates
    the venv on the pinned interpreter, and unset `VIRTUAL_ENV` first — mise activated the
    now-deleted venv for the task, and poetry would otherwise install into that stale path:

    ```toml
    [tasks.reinstall]
    description = "Delete .venv and reinstall dependencies from scratch"
    run = """
    rm -rf "$MISE_PROJECT_ROOT/.venv"
    env -u VIRTUAL_ENV mise run install
    """
    ```
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
type-check` and `poetry run pyright` pass, while an editor reports `Import "<dep>" could not be
resolved` on the same file. `poetry run` activates the venv; a language server spawned by an editor
inherits no such activation and lands on the system python, where none of the project's dependencies
exist. The two settings remove the ambiguity for every consumer at once — Pylance, the Claude Code
pyright server, the CLI, and CI.

Both keys are needed: `venv` is a directory *name* looked up inside `venvPath`, not a path.
`pythonVersion` is a separate concern — it is the language level to check against, deliberately
independent of the interpreter in `.venv` (see the
[Python Version Policy](python.md#python-version-policy)); a project can develop on a newer
interpreter while checking against its supported floor.

Verify it the way a language server sees it, with the venv deliberately off `PATH` — `poetry run`
masks the exact failure being tested:

```bash
env -u VIRTUAL_ENV PATH=/usr/bin:/bin ./.venv/bin/pyright <file importing a dependency>
```

Expect `0 errors`. With `--verbose`, the search paths should include
`.venv/lib/python<X.Y>/site-packages`.

### VSCode
- Install the Mise, Ruff, Pylance, and tombi extensions
    - `.vscode/settings.json`: mise, ruff format-on-save
- Check the in-project venv into `.vscode/settings.json` rather than relying on each developer
  running **Python: Select Interpreter**:

    ```json
    {
        "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
        "python-envs.pythonProjects": [
            {
                "path": ".",
                "envManager": "ms-python.python:venv",
                "packageManager": "ms-python.python:poetry"
            }
        ]
    }
    ```

  Both values are worth stating explicitly. `packageManager` defaults to `ms-python.python:pip`
  (`python-envs.defaultPackageManager`), which is wrong for a poetry project. And because
  `pythonProjects` entries are committed, an `envManager` of `ms-python.python:system` — pointing the
  workspace at the system interpreter rather than `.venv` — persists for everyone who clones, long
  after whoever wrote it moved on. Type-checking is already covered by `[tool.pyright]` above; these
  settings are what the *rest* of the Python extension reads — test discovery, the debugger, and the
  environment new terminals activate.
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
