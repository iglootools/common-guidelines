# Python Guidelines

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
- **Python Version**: 3.12 (pyright and ruff target).
- **Control Flow**
  - Prefer match-case over if-elif-else chains
  - Prefer comprehensions and built-ins (map, filter) over manual loops when appropriate.
  - Avoid `continue` in loops, and prefer filtering with comprehensions or built-ins instead.
  - Prefer single-expression returns over early returns when the logic can be expressed concisely
    (e.g. `return bool(x) and all(...)` over guard clauses with `return False`).
  - Prefer explicit if/else syntax over implicit else
  - Prefer dict unpacking with a filtered comprehension over if-chains when conditionally
    including keys (e.g. `**{k: v for k, v in {...}.items() if v is not None}`).
