# common-guidelines

Shared coding guidelines for [iglootools](https://github.com/iglootools) projects.

## Contents

- [philosophy.md](philosophy.md) — the reasoning behind the guidelines, and [how to deviate from them](philosophy.md#applying-these-guidelines)
- [coding.md](coding.md) — language-agnostic coding principles
- [python.md](python.md) — Python-specific coding guidelines
- [tooling.md](tooling.md) — GitHub Workflows, project setup, IDE configuration

## These are defaults, not dogma

Every rule here is a default recommendation. A project is free to deviate, or to add rules of its
own, **provided the deviation and the reasoning behind it are documented** — project-wide ones in the
project's `docs/guidelines.md`, local ones in a comment at the point of deviation.

Divergence for a good, documented reason is fine. Drift without one is not. See
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for where exceptions belong and
the bar a rationale has to meet.

## Usage with Claude Code

Add `@` imports in your project's `CLAUDE.md` (requires this repo cloned as a sibling directory):

```markdown
@../common-guidelines/coding.md
@../common-guidelines/python.md
@../common-guidelines/tooling.md
```

`coding.md` carries the "defaults, not dogma" clause, so importing it is enough for the exception
rule to reach the agent. Add `@../common-guidelines/philosophy.md` too if you want the full reasoning
in context — it is considerably longer than the other three combined.

Or add this repository as an additional working directory in Claude Code settings.
