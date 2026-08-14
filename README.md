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
@../common-guidelines/project-setup.md
@../common-guidelines/python-tooling.md
@../common-guidelines/ide.md
```

Import only what a project needs — a non-Python project has no use for `python.md` or
`python-tooling.md`, and a project with no editor configuration to share can leave out `ide.md`.

`coding.md` carries the "defaults, not dogma" clause, so importing it is enough for the exception
rule to reach the agent. Add `@../common-guidelines/philosophy.md` too if you want the full reasoning
in context — it is considerably longer than all the others combined.

Or add this repository as an additional working directory in Claude Code settings.
