# Python Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

- **Functional Style**:
  - Avoid mutable accumulator lists (`errors = []; errors.append(...)`). Instead, build lists as single expressions using `[*(...), *(...)]` unpacking, conditional `[item] if cond else []` fragments, and helper functions that return lists.
  - Prefer dict/list comprehensions over imperative loops for building collections.
  - When a function computes a list from multiple independent branches, compose the result by unpacking sub-expressions rather than mutating a shared list across branches.
- **String Literals**:
  - Prefer `dedent("""\...""")` multiline strings over concatenated single-line strings with `\n` escapes when the content has meaningful structure (e.g. YAML, config snippets, multi-line templates). Short single-line strings (e.g. `"key: value\n"`) are fine as-is.
- **Typing**: Use type annotations for all functions and methods, including return types. Use `pyright` for static type checking.
- **Data Classes**:
  - All serialized model objects are frozen pydantic dataclasses, immutable once created.
  - Other data classes should also be frozen.
- **Formatting**:
  - 88 characters (ruff default).
- **Python Version**: 3.12 floor, 3.14 for local development. See
  [Python Version Policy](#python-version-policy) below.
- **Control Flow**
  - Prefer match-case over if-elif-else chains
  - Prefer comprehensions and built-ins (map, filter) over manual loops when appropriate.
  - Avoid `continue` in loops, and prefer filtering with comprehensions or built-ins instead.
  - Prefer single-expression returns over early returns when the logic can be expressed concisely
    (e.g. `return bool(x) and all(...)` over guard clauses with `return False`).
  - Prefer explicit if/else syntax over implicit else
  - Prefer dict unpacking with a filtered comprehension over if-chains when conditionally
    including keys (e.g. `**{k: v for k, v in {...}.items() if v is not None}`).

## Python Version Policy

Projects target **two** Python versions at once:

| Role | Version | Why |
|---|---|---|
| **Floor** (minimum supported) | **3.12** | Default `python3` on Ubuntu 24.04 LTS |
| **Local development** | **3.14** | Latest stable; also the default on Ubuntu 26.04 LTS |

### Why 3.12 is the floor

Ubuntu 24.04 LTS ships Python 3.12 as its system `python3`. Keeping the floor there
means users on the previous LTS can `uv tool install` (or `pipx install`) a tool without
adding a PPA, building Python from source, or upgrading the distro. Nothing in the code
may use a feature newer than 3.12, even though development happens on 3.14.

> **Considering moving the floor to 3.14 soon.** Ubuntu 26.04 LTS ships Python 3.14
> (upgraded directly from 3.12 — 26.04 skips 3.13). Once we no longer need to support
> 24.04, raising the floor to 3.14 lets us drop the compatibility constraint entirely,
> since it would then match the local development version. Deliberately not done yet:
> 24.04 is in standard support until 2029, so dropping it now would strand users still
> on it.

### How compatibility with both is maintained

Five knobs enforce the floor. All of them must move together when the floor changes:

| Knob | Where | Value |
|---|---|---|
| `requires-python` | `pyproject.toml` | `>=3.12,<3.15` |
| ruff `target-version` | `pyproject.toml` | `py312` |
| pyright `pythonVersion` | `pyproject.toml` | `3.12` |
| `vermin --target` | `mise.toml` (`compat-check` task) | `3.12-` |
| CI matrix | `.github/workflows/test.yml` | `3.12.x` |

`vermin` is what actually catches accidental use of newer syntax and stdlib APIs —
ruff's and pyright's targets catch some cases but not all, so `compat-check` is not
redundant. The `<3.15` upper bound keeps the package from installing on a Python we
have never tested; it is raised deliberately, not automatically.

The local toolchain version lives in `mise.toml` (`[tools] python`), separate from all
of the above. It is intentionally *ahead* of the floor so problems on new Python
versions surface during development.

### Known gap

**CI only tests the floor.** The matrix entry for 3.14 is commented out to save CI
minutes, so the newer version is exercised only on developer machines. Uncomment it in
`.github/workflows/test.yml` if a Python-version-specific bug ever slips through.
