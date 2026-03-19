# Coding Guidelines

- **Functional Style**:
  - Prefer functional programming style over procedural style. Use pure functions and avoid mutability when possible.
- **Code comments**: When making changes to the codebase, explain the reasoning when the implementation is non-obvious, and document any non-trivial design decisions or trade-offs that were made.
- **Charsets**:
  - UTF-8 everywhere.
- **Time Management**
  - UTC for all timestamps
  - Do not generate the current timestamps directly inside the core logic: pass the timestamps from the higher-level functions, tests, and other entry points.
- **Mocks**
  - Prefer passing values as explicit parameters (with sensible defaults) over reading global/ambient state internally. This makes functions testable without mocking. For example, pass `now: datetime` instead of calling `datetime.now()` internally, pass `platform: str = sys.platform` instead of reading `sys.platform` internally. Tests should pass these values explicitly rather than patching modules.
- **Console Output**
  - Do not hardcode indents in strings, compute the indent at the call site
- **Version Management**
  - Pin specific versions of all dependencies or use a lock file (e.g. poetry.lock) to ensure reproducible builds and avoid issues with breaking changes in dependencies.

    ```bash
    # examples
    mise use --pin pipx:poetry
    ```
- **Command Line**
  - When calling external commands, build the command lines as lists of arguments instead of strings to avoid issues with quoting and escaping.
- **Testability**
  - Expose exceptions/errors as structured data classes and perform the assertions on the structured output in tests instead of matching against raw error message strings. This allows for more robust tests that are not brittle to changes in error message formatting.
- **No Silent Failures**
  - Avoid silent failures and ensure that all errors are surfaced with clear messages. This includes validating inputs and configurations early, and providing informative error messages when something goes wrong.
