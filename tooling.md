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
- Preferably print to the console using the [Rich](https://github.com/textualize/rich) library that is bundled with [Typer](https://typer.tiangolo.com/)

### CLI Projects
Consider providing:
- `asciinema` demo
- auto-generated `clidocs`/`clidocs-check` tasks for CLI documentation
- Provide preflight checks wherever applicable and troubleshooting instructions with the specific commands that need to be executed to fix the problem.

## IDE

### VSCode
- Install the Mise, Ruff, Pylance, and tombi extensions
    - `.vscode/settings.json`: mise, ruff format-on-save
