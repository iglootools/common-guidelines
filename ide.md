# IDE Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

This file covers editor and agent configuration. For the toolchain those editors point at, see
[python-tooling.md](python-tooling.md); for the setup steps every project shares, see
[project-setup.md](project-setup.md).

## Pyright environment resolution

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

## VSCode

### Commit the extension set to `.vscode/extensions.json`

So a fresh clone is prompted for it, instead of documenting it in prose that nobody reads before
opening the editor:

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

### Check the in-project venv into `.vscode/settings.json`

Rather than relying on each developer running **Python: Select Interpreter**:

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

### Let `hverlin.mise-vscode` write the other tools' paths, but fence it in

Every extension that needs a toolchain binary has its own key (`metals.javaHome`, `deno.path`,
`go.goroot`, …), and hand-maintaining them against `mise.toml` is exactly the kind of duplication
that drifts:

```json
{
    "mise.enable": true,
    "mise.configureExtensionsAutomatically": true,
    "mise.configureExtensionsUseSymLinks": true,
    "mise.configureExtensionsIncludeGlobalTools": false
}
```

| Setting | Non-default? | Why |
|---|---|---|
| `configureExtensionsAutomatically` | yes (`false`) | derives the per-extension tool keys from `mise.toml` instead of leaving them to be typed once and forgotten |
| `configureExtensionsUseSymLinks` | yes (`false`) | writes `${workspaceFolder}/.vscode/mise-tools/<tool>` rather than an absolute `~/.local/share/mise/installs/…` path. Only the symlink form is shareable: an absolute install path is version-specific and home-directory-specific, so committing it hands every other developer a path that does not exist on their machine |
| `configureExtensionsIncludeGlobalTools` | yes (`true`) | restricts configuration to tools the project's own `mise.toml` declares. The extension's own documentation calls `false` the recommended value, and the failure it prevents is visible in practice: a Nuxt repo declaring only `node` and `pnpm` acquired `metals.javaHome` and a `python.defaultInterpreterPath` pointing at a global mise interpreter, purely because those tools were in `~/.config/mise/config.toml` |

Because the symlinks are per-machine, **`.vscode/mise-tools/` must be in the committed
`.gitignore`** — the extension's documentation makes this a condition of sharing the settings
file at all. The [All Projects](project-setup.md#all-projects) rule applies unchanged: ignoring it
only in `.git/info/exclude` looks like coverage to whoever set it up and gives none to anyone else.

**This does not replace the explicit `.venv` pin above, and must not be read as doing so.** For
`ms-python.python` the extension prefers mise's `VIRTUAL_ENV` and writes
`${workspaceFolder}/.venv/bin/python` — the same value, so the two agree. But it *falls back* to
the mise toolchain interpreter when `VIRTUAL_ENV` is absent, which is precisely the state of a
fresh clone: the first `mise install` deliberately does not create `.venv` (see
[Mise and uv](python-tooling.md#mise-and-uv)). Generated-only, that window writes a python with
none of the project's dependencies into a committed file. Keeping `python.defaultInterpreterPath`
written out by hand makes the generated value a confirmation rather than the sole source.

These settings do **not** isolate projects in a multi-root workspace — for that, see
[Multi-root workspaces](#multi-root-workspaces) below.

### Reload after changing interpreter or pyright settings

Use **Developer: Reload Window**. Current Pylance releases have no "Python: Restart Language
Server" command, and an already-running server keeps serving diagnostics from the configuration it
started with.

### Multi-root workspaces

Opening several projects in one `.code-workspace` is where per-project tool configuration is most
likely to leak across projects, so it is worth being explicit about which mechanisms hold there and
which do not.

**`mise.*` settings are window-scoped, so a committed `.vscode/settings.json` does not carry into a
multi-root window.** Checked against `hverlin.mise-vscode` 1.23.0: every `mise.*` key — `mise.enable`
and all four `configureExtensions*` keys included — declares the default `window` scope. VSCode reads
window-scoped settings from user, remote and *workspace* settings only; per-folder values are
ignored. A project that commits `"mise.enable": true` therefore configures itself when opened as a
single root and configures nothing when it is one folder among several. The setting has to be
repeated in the `.code-workspace` file's `settings` block, where it applies to every folder at once.

**mise-vscode configures one folder at a time, and writes the result where it applies to all of
them.** The extension resolves a single "current" workspace folder — whichever was chosen with
**mise: Select workspace folder**, defaulting to `workspaceFolders[0]` — and writes the generated
keys at `ConfigurationTarget.Workspace`, which in a multi-root window is the `.code-workspace` file.
The paths it emits are relative to `${workspaceFolder}`, unqualified. Composed, those three facts
mean: one folder's tools, expressed with a variable that does not name it, applied to every folder.
Observed on a seven-folder workspace — a `settings` block sending all seven to
`${workspaceFolder}/.vscode/mise-tools/python` when only the first folder had that directory, and
its `python` symlink pointed at a global interpreter rather than any project's `.venv`.

So the practical rule: **turn `configureExtensionsAutomatically` on per project, and let it write
only in single-folder windows.** When working in a multi-root workspace, do not accept generated
`settings` in the `.code-workspace` file, and do not commit them — the workspace file is a view onto
projects, not a place where any one project's toolchain belongs.

**What actually keeps the projects apart** is per-project and does not depend on the mise extension:

| Consumer | Mechanism | Scope |
|---|---|---|
| pyright — CLI, CI, Claude Code's LSP | `[tool.pyright] venvPath`/`venv` in each `pyproject.toml` | per project, no VSCode involvement at all |
| Pylance, test discovery, debugger | `python.defaultInterpreterPath` in the folder's own `.vscode/settings.json` | `machine-overridable`, so a folder-level value overrides the workspace one |
| terminals and tasks | `uv run --no-sync <tool>`, which resolves from the project root and ignores `PATH` | per invocation |

Each of those is a folder-level or file-level fact, which is why they survive being opened alongside
other projects; the mise extension's settings are window-level, which is why they do not. Adding the
`configureExtensions*` keys is worth doing for the toolchains that have no equivalent of `.venv` —
it is not a substitute for any row of that table.

## Claude Code

### Pyright LSP plugin

Install it in every Python project so Claude resolves symbols instead of grepping for them:

```bash
claude plugin install pyright-lsp@claude-plugins-official --scope project
```

It provides go-to-definition, find-references, hover types, document and workspace symbol search,
and call hierarchy, and it pushes diagnostics into Claude's context after each edit. Where the same
name appears in several modules, this is the difference between an answer grounded in the import
graph and one inferred from text matches.

### Multi-project workspaces

Opening several projects in one window — or reaching them as additional
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

### Picking up configuration changes

The plugin's pyright server reads `[tool.pyright]` at startup, so edits to it do not reach a running
session. Restart the editor window (or the Claude Code session) and let the plugin spawn the server
itself. Do not kill the server process expecting a respawn: the plugin does not restart it on
demand, and it then reports `server is running` for a process that no longer exists, failing every
LSP request until the session is reloaded. Note that this server is distinct from the editor's
own — VSCode runs Pylance for the squiggles and the plugin runs its own pyright for Claude, so a
stale diagnostic on one side says nothing about the other.

### Sharing the plugin set with the team

`--scope project` is what makes the install shared: it writes `enabledPlugins` into the repository's
`.claude/settings.json` instead of your own `~/.claude/settings.json`, so everyone who clones the
repo gets the same plugins rather than each developer installing them by hand. **Commit that file** —
only `.claude/settings.local.json` is normally ignored, so it is easy to leave the shared half
untracked and never notice, since your own user-scope install keeps the plugin working locally.

Plugins declared this way come from the repository rather than from the developer, so they load only
after the workspace trust dialog is accepted — LSP servers in particular start only once the
workspace is trusted.
