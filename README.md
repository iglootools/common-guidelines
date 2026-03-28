# common-guidelines

Shared coding guidelines for [iglootools](https://github.com/iglootools) projects.

## Contents

- [philosophy.md](philosophy.md) — the reasoning behind the guidelines
- [coding.md](coding.md) — language-agnostic coding principles
- [python.md](python.md) — Python-specific coding guidelines
- [tooling.md](tooling.md) — GitHub Workflows, project setup, IDE configuration

## Usage with Claude Code

Add `@` imports in your project's `CLAUDE.md` (requires this repo cloned as a sibling directory):

```markdown
@../common-guidelines/coding.md
@../common-guidelines/python.md
@../common-guidelines/tooling.md
```

Or add this repository as an additional working directory in Claude Code settings.
