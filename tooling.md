# Tooling Guidelines

## GitHub Workflows
- Whenever safe (i.e. not affecting production), enable `workflow_dispatch` and `repository_dispatch` to allow manual triggering of workflows from the GitHub UI or CLI, which is useful for testing and debugging.
- Use OpenID Connect (OIDC) authentication for publishing to PyPI, and set up a separate workflow for testing releases to Test PyPI. This allows testing the release and publish process without affecting the real PyPI index, and provides more detailed logs for debugging.

## New Project Setup

Guidelines to follow when setting up new projects.

### All Projects:
- `renovate.json`
- Github build/test/release workflows
- `git config user.email "<email>"` and `git config user.name "<name>"` in the project.

### Python Projects
- Build tool: poetry with poetry-dynamic-versioning
- Mise + Poetry: Use poetry `virtualenvs.in-project = true` with mise `_.python.venv = { path = ".venv", create = true }` to ensure that the virtual environment is created inside the project directory and automatically activated when running commands with `mise run`.
- Use `ruff` for linting and formatting, `pyright` for type-checking, and `vermin` for validating the desired Python version compatibility.
- Consider providing:
  - `configdocs`/`configdocs-check`, and `depgraph`/`depgraph-check` tasks for config and dependency documentation, respectively
  - `format`, `lint`, `type-check`, `compat-check` tasks for code quality checks

### CLI Projects
Consider providing:
- `asciinema` demo
- auto-generated `clidocs`/`clidocs-check` tasks for CLI documentation

## IDE

### VSCode
- Install the Mise, Ruff, Pylance, and tombi extensions
    - `.vscode/settings.json`: mise, ruff format-on-save
