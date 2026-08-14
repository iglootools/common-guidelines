# common-guidelines

Shared coding guidelines for [iglootools](https://github.com/iglootools) projects.

## Contents

- [philosophy.md](philosophy.md) — the reasoning behind the guidelines, and [how to deviate from them](philosophy.md#applying-these-guidelines)
- [coding.md](coding.md) — language-agnostic coding principles
- [python.md](python.md) — Python-specific coding guidelines
- [project-setup.md](project-setup.md) — GitHub Workflows, dependency automation, new-project setup
- [python-tooling.md](python-tooling.md) — uv, mise, hatchling, and the mise task set
- [ide.md](ide.md) — pyright resolution, VSCode, and Claude Code configuration
- [scripts/](scripts/) — reference implementations to copy into a project, for the few cases where
  a guideline is easier to ship as working code than to describe. The guideline that motivates each
  one links to it, and explains why every line is there.

## These are defaults, not dogma

Every rule here is a default recommendation. A project is free to deviate, or to add rules of its
own, **provided the deviation and the reasoning behind it are documented** — project-wide ones in the
project's `docs/guidelines.md`, local ones in a comment at the point of deviation.

Divergence for a good, documented reason is fine. Drift without one is not.

A major reason to insist on the written rationale is that it makes the exception **re-evaluatable**.
Most exceptions answer a condition that is true at the time — a library gap, a performance constraint, a
version we still support — and those conditions expire. Where one does, name the condition that would
retire the exception ("drop this once we no longer support Z"), so a future reader can check whether the
justification still holds instead of guessing. Undocumented exceptions become permanent by default.

See [Applying These Guidelines](philosophy.md#applying-these-guidelines) for where exceptions belong and
the bar a rationale has to meet.

## Usage with Claude Code

Add `@` imports in your project's `CLAUDE.md` (requires this repo cloned as a sibling directory):

```markdown
@../common-guidelines/coding.md
@../common-guidelines/python.md
```

**Import the two that govern every edit; point at the rest.** An `@` import is loaded into every
session whether or not the task needs it, so it should be reserved for guidelines that apply to any
change. `project-setup.md`, `python-tooling.md` and `ide.md` do not: they are triggered by a
specific file, and a session that never touches that file pays for them anyway. Follow the imports
with a table naming the trigger, so the agent can tell when to go read one:

```markdown
| Read | Before touching |
|---|---|
| ../common-guidelines/project-setup.md | `.github/workflows/`, `renovate.json`, `dependabot.yml`, `.gitignore` |
| ../common-guidelines/python-tooling.md | `pyproject.toml`, `mise.toml`, `uv.lock` — or adding a dependency or mise task |
| ../common-guidelines/ide.md | `.vscode/`, `.claude/settings.json`, `*.code-workspace`, `[tool.pyright]` |
```

Make the trigger a **path**, not a topic. "When working on packaging" requires the agent to have
already understood the task as a packaging task; "before touching `pyproject.toml`" is checkable
against the edit it is about to make. Drop rows for files a project does not have, and drop
`@python.md` from a non-Python project.

`coding.md` carries the "defaults, not dogma" clause, so importing it is enough for the exception
rule to reach the agent. Add `@../common-guidelines/philosophy.md` too if you want the full reasoning
in context — it is considerably longer than all the others combined.

Or add this repository as an additional working directory in Claude Code settings.
