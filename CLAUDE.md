# Common Guidelines

Shared coding guidelines for iglootools projects.

These are defaults, not dogma. A project may deviate from any of these rules or add its own,
as long as the deviation and its rationale are documented — project-wide ones in the project's
`docs/guidelines.md`, local ones in a comment at the point of deviation. A justified exception
is fine; silent drift is not. Where the exception depends on a condition that may change, name the
condition that would retire it — the written rationale is what lets it be re-evaluated later instead
of becoming permanent by default. See [Applying These Guidelines](philosophy.md#applying-these-guidelines).

@coding.md
@python.md

The two files above are imported because they govern every edit. The rest are setup- and
tooling-specific, so they are read when the task reaches them rather than loaded into every
session — each is triggered by a file you can see you are about to touch:

| Read | Before touching |
|---|---|
| [project-setup.md](project-setup.md) | `.github/workflows/`, `renovate.json`, `dependabot.yml`, `.gitignore` — or when setting up a new repository |
| [python-tooling.md](python-tooling.md) | `pyproject.toml`, `mise.toml`, `uv.lock` — or when adding a dependency, a mise task, or anything about building and publishing |
| [ide.md](ide.md) | `.vscode/`, `.claude/settings.json`, `*.code-workspace`, `[tool.pyright]` |

Read the whole file, not the section that looks relevant. These guidelines are mostly about
failures that are silent — a wheel that builds empty, a lockfile check that passes by being
broken, a setting whose extension is absent — so the part you would have skipped is usually the
part that names the failure.
